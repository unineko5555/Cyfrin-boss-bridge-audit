# Boss Bridge セキュリティ監査 - 脆弱性分析レポート

## 監査実施日: 2025-10-03

---

## 🚨 CRITICAL 脆弱性

### [C-01] depositTokensToL2による無許可のトークン盗難

**ファイル:** `src/L1BossBridge.sol:78-86`

**脆弱性の詳細:**
```solidity
function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused {
    if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
        revert L1BossBridge__DepositLimitReached();
    }
    token.safeTransferFrom(from, address(vault), amount);  // ❌ msg.senderチェックなし
    emit Deposit(from, l2Recipient, amount);
}
```

**問題点:**
- `from`パラメータが完全に信頼されている
- `msg.sender == from`のチェックが一切ない
- 誰でも他人の承認済みトークンを盗める

**攻撃シナリオ:**
1. 被害者Aliceがブリッジに1000トークンを承認: `approve(bridge, 1000e18)`
2. 攻撃者BobがAliceのトークンを盗む:
   ```solidity
   tokenBridge.depositTokensToL2(alice, bob_L2_address, 1000e18);
   ```
3. Aliceのトークンがvaultに転送され、BobのL2アドレスにミントされる

**影響:**
- **重大度:** CRITICAL
- **影響度:** HIGH - 全ユーザーの承認済みトークンが盗難可能
- **発生確率:** HIGH - 攻撃実行が極めて簡単
- MEV botによる自動攻撃が可能

**概念実証コード (PoC):**
```solidity
function testAttackerCanStealApprovedTokens() public {
    // 被害者がブリッジを承認
    vm.startPrank(user);
    token.approve(address(tokenBridge), 100e18);
    vm.stopPrank();

    // 攻撃者が盗む
    address attacker = makeAddr("attacker");
    vm.prank(attacker);
    tokenBridge.depositTokensToL2(user, attacker, 100e18);

    // 被害者のトークンが盗まれ、攻撃者がL2で受け取る
}
```

**推奨される修正:**
```solidity
function depositTokensToL2(address l2Recipient, uint256 amount) external whenNotPaused {
    if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
        revert L1BossBridge__DepositLimitReached();
    }
    // msg.senderのトークンを転送
    token.safeTransferFrom(msg.sender, address(vault), amount);
    emit Deposit(msg.sender, l2Recipient, amount);
}
```

---

### [C-02] トークンデプロイ失敗時のチェック不足

**ファイル:** `src/TokenFactory.sol:23-35`

**脆弱性の詳細:**
```solidity
function deployToken(string memory symbol, bytes memory contractBytecode) public onlyOwner returns (address addr) {
    assembly {
        addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
    }
    s_tokenToAddress[symbol] = addr;  // ❌ addrが0x0でも保存される
    emit TokenDeployed(symbol, addr);
}
```

**問題点:**
- `create`が失敗すると`address(0)`を返すが、チェックがない
- デプロイ失敗時も`s_tokenToAddress[symbol] = 0x0`として保存される
- ユーザーが`getTokenAddressFromSymbol`で`0x0`を取得し、資金が永久ロック

**攻撃シナリオ:**
1. オーナーが無効なバイトコードでデプロイ試行
2. `create`が失敗し`address(0)`を返す
3. `s_tokenToAddress["TOKEN"] = 0x0`が保存される
4. ユーザーが`0x0`にトークン送信し、資金が永久ロック

**影響:**
- **重大度:** CRITICAL
- **影響度:** HIGH - 資金の永久喪失
- **発生確率:** MEDIUM - デプロイ失敗時に発生

**推奨される修正:**
```solidity
error TokenFactory__DeploymentFailed();

function deployToken(string memory symbol, bytes memory contractBytecode)
    public onlyOwner returns (address addr)
{
    assembly {
        addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
    }

    if (addr == address(0)) {
        revert TokenFactory__DeploymentFailed();
    }

    s_tokenToAddress[symbol] = addr;
    emit TokenDeployed(symbol, addr);
}
```

---

## 🔴 HIGH 脆弱性

### [H-01] ZKSync EraのCREATEオペコード非互換性

**ファイル:** `src/TokenFactory.sol:27-32`, `README.md:108-109`

**脆弱性の詳細:**
- ZKSync EraはEVMの`CREATE` opcodeをネイティブサポートしていない
- ZKSync Eraでは`CREATE`のアドレス計算方法が異なる
- L1とL2で同じアドレスにデプロイできない

**READMEの記載:**
```markdown
Chain(s) to deploy contracts to:
- ZKSync Era: TokenFactory.sol
```

**問題点:**
```solidity
assembly {
    addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
}
// ❌ ZKSync Eraでは正しく動作しない
```

**影響:**
- **重大度:** HIGH
- **影響度:** HIGH - ブリッジ機能が完全に破綻
- **発生確率:** HIGH - ZKSync Eraデプロイ時に確実に発生
- L1/L2間でトークンアドレスの不一致

**推奨される修正:**
```solidity
// CREATE2を使用してクロスチェーン互換性を確保
assembly {
    addr := create2(0, add(contractBytecode, 0x20), mload(contractBytecode), salt)
}
```

---

### [H-02] Vault自己転送による不正なL2ミント

**ファイル:** `src/L1BossBridge.sol:78-87`

**脆弱性の詳細:**
攻撃者がVaultに直接トークンを送信し、VaultからVault自身への転送を実行することで、実際にトークンを預けずにL2でミントさせることができる。

**問題点:**
```solidity
function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused {
    // ❌ fromがvaultアドレスかチェックしていない
    token.safeTransferFrom(from, address(vault), amount);
    emit Deposit(from, l2Recipient, amount);
}

// Vaultは自身にtype(uint256).maxの承認を持っている
// vault.token.allowance(vault, bridge) == type(uint256).max
```

**攻撃シナリオ:**

1. 攻撃者が外部からVaultに500トークンを直接送信（`transfer`で）
2. `depositTokensToL2(address(vault), attacker, 500e18)`を呼び出し
3. VaultからVaultへの自己転送が成功（Vaultは自身にBridgeを通じて無制限承認済み）
4. `Deposit(vault, attacker, 500e18)`イベントが発行される
5. オフチェーンサービスがこのイベントを検知し、L2で500トークンをミント
6. 攻撃者は何も失わずL2で500トークンを獲得
7. **無限に繰り返し可能** - 同じVault内のトークンを使い回せる

**影響:**
- **重大度:** HIGH
- **影響度:** HIGH - 無制限のトークンミント、総供給量の破壊
- **発生確率:** MEDIUM - 事前にVaultへのトークン送信が必要
- L2での無限トークン生成
- L1/L2間の供給量不整合

**概念実証コード (PoC):**
```solidity
function testVaultSelfTransferAttack() public {
    address attacker = makeAddr("attacker");
    uint256 vaultBalance = 500 ether;

    // Vaultに直接トークンを送信（通常のデポジットではない）
    deal(address(token), address(vault), vaultBalance);

    // 無限に繰り返し可能 - 同じトークンを使い回す
    for (uint i = 0; i < 10; i++) {
        vm.expectEmit(true, true, true, true, address(tokenBridge));
        emit Deposit(address(vault), attacker, vaultBalance);
        tokenBridge.depositTokensToL2(address(vault), attacker, vaultBalance);
    }

    // 結果: 攻撃者はL2で 500 * 10 = 5000 トークンを獲得
    // しかしL1では500トークンしか消費していない
    // L2総供給量 = L1総供給量 + 4500 (不正増加)
}
```

**テストコード修正:**
```solidity
// 元のテストコード（vm.startEmitはエラー）
function testCanTransferVaultToVault() public {
    address attacker = makeAddr("attacker");
    uint256 vaultBalance = 500 ether;
    deal(address(token), address(vault), vaultBalance);

    // ❌ vm.startEmit は存在しない
    vm.startEmit(address(tokenBridge));
    emit Deposit(attacker, attacker, vaultBalance);
    tokenBridge.depositTokensToL2(address(vault), attacker, vaultBalance);
}

// 修正版（vm.expectEmitを使用）
function testCanTransferVaultToVault() public {
    address attacker = makeAddr("attacker");
    uint256 vaultBalance = 500 ether;
    deal(address(token), address(vault), vaultBalance);

    // ✅ vm.expectEmit(checkTopic1, checkTopic2, checkTopic3, checkData, emitter)
    vm.expectEmit(true, true, true, true, address(tokenBridge));
    emit Deposit(address(vault), attacker, vaultBalance);
    tokenBridge.depositTokensToL2(address(vault), attacker, vaultBalance);
}
```

**推奨される修正:**
```solidity
error L1BossBridge__VaultCannotDeposit();

function depositTokensToL2(address l2Recipient, uint256 amount) external whenNotPaused {
    if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
        revert L1BossBridge__DepositLimitReached();
    }

    // msg.senderがvaultの場合は拒否
    if (msg.sender == address(vault)) {
        revert L1BossBridge__VaultCannotDeposit();
    }

    token.safeTransferFrom(msg.sender, address(vault), amount);
    emit Deposit(msg.sender, l2Recipient, amount);
}
```

**または、より根本的な修正（C-01と同時解決）:**
```solidity
// fromパラメータを削除し、msg.senderのみ使用
function depositTokensToL2(address l2Recipient, uint256 amount) external whenNotPaused {
    if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
        revert L1BossBridge__DepositLimitReached();
    }
    // msg.senderのトークンのみ転送可能
    token.safeTransferFrom(msg.sender, address(vault), amount);
    emit Deposit(msg.sender, l2Recipient, amount);
}
```

---

## 🟠 MEDIUM 脆弱性

### [M-01] トークンシンボルの上書き可能性

**ファイル:** `src/TokenFactory.sol:23-34`

**脆弱性の詳細:**
```solidity
function deployToken(string memory symbol, bytes memory contractBytecode) public onlyOwner {
    assembly {
        addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
    }
    s_tokenToAddress[symbol] = addr;  // ❌ 既存シンボルを上書き可能
}
```

**問題点:**
- 同じシンボルで複数回デプロイ可能
- 古いトークンアドレスが上書きされ、アクセス不能になる
- 古いトークンに資金が残っていた場合、回収不可能

**攻撃シナリオ:**
1. オーナーが"BBT"でトークンAをデプロイ
2. ユーザーがトークンAに資金をロック
3. オーナーが誤って"BBT"でトークンBをデプロイ
4. `s_tokenToAddress["BBT"]`がトークンBに上書き
5. トークンAの資金が永久に回収不能

**影響:**
- **重大度:** MEDIUM
- **影響度:** HIGH - 資金の永久喪失
- **発生確率:** LOW - オーナーのオペレーションミス

**推奨される修正:**
```solidity
error TokenFactory__SymbolAlreadyExists();

function deployToken(string memory symbol, bytes memory contractBytecode)
    public onlyOwner returns (address addr)
{
    if (s_tokenToAddress[symbol] != address(0)) {
        revert TokenFactory__SymbolAlreadyExists();
    }

    assembly {
        addr := create(0, add(contractBytecode, 0x20), mload(contractBytecode))
    }

    s_tokenToAddress[symbol] = addr;
    emit TokenDeployed(symbol, addr);
}
```

---

## 📊 脆弱性サマリー

| ID | タイトル | 重大度 | ファイル | 状態 |
|---|---|---|---|---|
| C-01 | 無許可のトークン盗難 | CRITICAL | L1BossBridge.sol:78 | 未修正 |
| C-02 | デプロイ失敗チェック不足 | CRITICAL | TokenFactory.sol:31 | 未修正 |
| H-01 | ZKSync CREATE非互換性 | HIGH | TokenFactory.sol:31 | 未修正 |
| H-02 | Vault自己転送による不正ミント | HIGH | L1BossBridge.sol:78 | 未修正 |
| M-01 | シンボル上書き可能 | MEDIUM | TokenFactory.sol:33 | 未修正 |

---

## 🛠️ 監査推奨事項

1. **即時対応が必要 (CRITICAL):**
   - C-01: `depositTokensToL2`から`from`パラメータを削除し、`msg.sender`のみ使用
   - C-02: デプロイ失敗時のリバートを追加

2. **デプロイ前に必須 (HIGH):**
   - H-01: ZKSync Era対応のため`CREATE2`に変更
   - H-02: `depositTokensToL2`でVaultからのデポジットを禁止（またはC-01修正で解決）

3. **改善推奨 (MEDIUM):**
   - M-01: シンボル重複チェックを追加

4. **テストカバレッジ改善:**
   - 攻撃者が他人のトークンを盗むケースのテスト追加
   - Vault自己転送攻撃のテスト追加（`testCanTransferVaultToVault`を修正）
   - デプロイ失敗ケースのテスト追加
   - ZKSync Eraでのデプロイテスト追加

5. **テストコード修正:**
   - `vm.startEmit` → `vm.expectEmit(true, true, true, true, address(emitter))`に修正

---

## 📝 監査メモ

### テストの盲点
- `testUserCanDepositTokens`では常に`msg.sender == from`
- 攻撃者が`from`に別アドレスを指定するケースが未テスト
- デプロイ失敗時の動作が未検証

### 根本原因
- **Trust Boundary違反**: ユーザー制御の入力を無条件に信頼
- **Checks欠落**: Checks-Effects-Interactionsパターンの違反
- **エラーハンドリング不足**: 失敗ケースの考慮不足
