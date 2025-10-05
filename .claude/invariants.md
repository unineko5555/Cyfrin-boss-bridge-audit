# Boss Bridge プロトコル不変条件

## 概要
このドキュメントでは、Boss Bridgeプロトコルが常に満たすべき不変条件（Invariants）を定義します。これらの条件が破られた場合、プロトコルのセキュリティや整合性が損なわれます。

---

## 🔒 不変条件リスト

### INV-1: Vault Token Balance Limit
**条件:**
```solidity
token.balanceOf(address(vault)) <= DEPOSIT_LIMIT
```

**説明:**
- Vaultのトークン残高は常にDEPOSIT_LIMIT（100,000 ether）以下でなければならない
- depositTokensToL2関数で強制される（L1BossBridge.sol:80-81）

**違反時の影響:**
- プロトコルの設計意図に反する過剰な預金
- システムリスクの増加

**検証方法:**
- Invariant testで全操作後にチェック

---

### INV-2: Total Supply Conservation
**条件:**
```solidity
vault_balance + L2_total_minted == total_deposited_amount
```

**説明:**
- L1 Vaultの残高 + L2でミントされた総量 = 総預け入れ額
- トークンは消滅せず、必ずL1のVaultかL2に存在する
- デポジットとウィズドローは1:1の対応関係

**違反時の影響:**
- トークンの不正な増減
- ユーザー資金の喪失または複製

**検証方法:**
- デポジット/ウィズドロー操作の追跡
- オフチェーンL2ミント量との照合（テストではモック使用）

---

### INV-3: Vault Approval Persistence
**条件:**
```solidity
token.allowance(address(vault), address(bridge)) == type(uint256).max
```

**説明:**
- VaultはBridgeに対して常に無制限の承認を持つ
- コンストラクタで設定される（L1BossBridge.sol:47）
- この承認が失われると出金が不可能になる

**違反時の影響:**
- CRITICAL: 全ての出金が不可能になる
- ユーザー資金が永久にロックされる

**検証方法:**
- 全操作後にallowanceをチェック
- approveTo関数の呼び出しを監視

---

### INV-4: Vault Ownership Immutability
**条件:**
```solidity
vault.owner() == address(bridge)
```

**説明:**
- VaultのオーナーはBridgeコントラクトのみ
- オーナー権限でapproveTo関数を呼び出せる
- オーナーシップ移転は想定されていない

**違反時の影響:**
- CRITICAL: 不正なアドレスがVaultを制御可能
- 資金の盗難

**検証方法:**
- Vault初期化後にowner()をチェック
- transferOwnership呼び出しを禁止

---

### INV-5: Token Address Consistency
**条件:**
```solidity
bridge.token() == vault.token()
```

**説明:**
- BridgeとVaultは同一のトークンを扱う
- 両方ともimmutableで設定される
- トークンアドレスの不一致は設計エラー

**違反時の影響:**
- プロトコルの完全な機能不全
- デプロイメントエラー

**検証方法:**
- デプロイ直後にチェック
- immutableのため変更不可

---

### INV-6: Vault Balance Accounting
**条件:**
```solidity
token.balanceOf(address(vault)) == Σ(deposits) - Σ(withdrawals)
```

**説明:**
- Vaultの実際の残高は、全デポジット総額から全ウィズドロー総額を引いた値に等しい
- 直接送金されたトークンは会計に含まれない
- Deposit/Withdrawイベントで追跡可能

**違反時の影響:**
- 会計の不整合
- 意図しないトークンの滞留（直接送金された場合）

**検証方法:**
- デポジット/ウィズドローの累積値を追跡
- 実際の残高と比較

---

### INV-7: Signer-Only Withdrawals
**条件:**
```solidity
∀ withdrawal: ∃ valid_signer_signature
```

**説明:**
- 全てのウィズドロー操作は、有効な署名者による署名を持つ
- signers mappingに登録されたアドレスの署名のみ有効
- ECDSA署名検証で強制される（L1BossBridge.sol:123-126）

**違反時の影響:**
- CRITICAL: 不正な出金
- 署名検証のバイパス

**検証方法:**
- 無効な署名者での出金試行をテスト
- 署名なし出金の失敗を確認

---

### INV-8: Deposit Limit Enforcement
**条件:**
```solidity
∀ deposit: vault_balance_after_deposit <= DEPOSIT_LIMIT
```

**説明:**
- 各デポジット操作後、Vault残高はDEPOSIT_LIMIT以下
- オーナーがDEPOSIT_LIMITを変更可能（L1BossBridge.sol:58-60）
- リミット超過はrevert

**違反時の影響:**
- プロトコルリスク管理の失敗
- 過剰なエクスポージャー

**検証方法:**
- DEPOSIT_LIMIT超過デポジットの失敗を確認
- setDepositLimit後の整合性チェック

---

## 🚨 破られやすい不変条件（脆弱性関連）

### BROKEN-1: Unauthorized Deposit Protection (現在破損)
**期待される条件:**
```solidity
∀ deposit: msg.sender == from || msg.sender has permission from 'from'
```

**現状:**
- **違反している**: depositTokensToL2は`from`パラメータを信頼し、msg.senderチェックなし
- 誰でも他人の承認済みトークンを盗める（C-01脆弱性）

**修正後の条件:**
```solidity
depositTokensToL2(address l2Recipient, uint256 amount) {
    // msg.senderのトークンのみ転送可能
    token.safeTransferFrom(msg.sender, address(vault), amount);
}
```

---

### BROKEN-2: No Replay Attack (署名の再利用防止)
**期待される条件:**
```solidity
∀ signature: signature can only be used once
```

**現状:**
- **潜在的違反**: 署名の再利用防止メカニズムがない
- nonceやタイムスタンプによる保護なし
- 同じ署名で複数回出金可能（Replay Attack）

**推奨される条件:**
```solidity
mapping(bytes32 => bool) usedSignatures;

function withdrawTokensToL1(...) {
    bytes32 sigHash = keccak256(abi.encode(v, r, s, message));
    require(!usedSignatures[sigHash], "Signature already used");
    usedSignatures[sigHash] = true;
    ...
}
```

---

## 📊 不変条件の優先度

| ID | 不変条件 | 重要度 | 現在の状態 |
|---|---|---|---|
| INV-1 | Vault Balance Limit | HIGH | ✅ 保護されている |
| INV-2 | Total Supply Conservation | CRITICAL | ⚠️ オフチェーン依存 |
| INV-3 | Vault Approval Persistence | CRITICAL | ✅ 保護されている |
| INV-4 | Vault Ownership | CRITICAL | ✅ 保護されている |
| INV-5 | Token Consistency | HIGH | ✅ 保護されている |
| INV-6 | Balance Accounting | MEDIUM | ⚠️ 直接送金で破損 |
| INV-7 | Signer-Only Withdrawals | CRITICAL | ✅ 保護されている |
| INV-8 | Deposit Limit | HIGH | ✅ 保護されている |
| BROKEN-1 | Unauthorized Deposit Protection | CRITICAL | ❌ 破損 (C-01) |
| BROKEN-2 | No Replay Attack | HIGH | ❌ 未実装 |

---

## 🧪 Invariant Testでテストすべき項目

### 優先度: CRITICAL
1. ✅ **INV-3**: Vault Approval Persistence
2. ✅ **INV-7**: Signer-Only Withdrawals
3. ❌ **BROKEN-1**: Unauthorized Deposit (現在破損、修正後にテスト)

### 優先度: HIGH
4. ✅ **INV-1**: Vault Balance Limit
5. ✅ **INV-8**: Deposit Limit Enforcement
6. ❌ **BROKEN-2**: Replay Attack Protection (未実装)

### 優先度: MEDIUM
7. ✅ **INV-6**: Balance Accounting
8. ⚠️ **INV-2**: Total Supply Conservation (オフチェーンモック必要)

---

## 📝 次のステップ

1. **Handler契約の作成**: ランダムな操作を実行するHandler
2. **Invariant Testスイート作成**: 各不変条件を検証
3. **テスト実行**: Foundryの`forge test --mt invariant`
4. **脆弱性の確認**: BROKEN-1, BROKEN-2が実際に破られることを証明
