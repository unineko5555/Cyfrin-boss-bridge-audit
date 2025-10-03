# Project Structure

```
7-boss-bridge-audit/
├── src/                          # Smart contract source files
│   ├── L1BossBridge.sol         # Main bridge contract (deposit/withdraw logic)
│   ├── L1Token.sol              # ERC20 token implementation
│   ├── L1Vault.sol              # Token custody vault
│   └── TokenFactory.sol         # Token deployment factory
│
├── test/                         # Test files
│   ├── L1TokenBridge.t.sol      # Bridge contract tests
│   └── TokenFactoryTest.t.sol   # Factory contract tests
│
├── lib/                          # External dependencies
│   ├── forge-std/               # Foundry standard library
│   └── openzeppelin-contracts/  # OpenZeppelin security libraries
│
├── out/                          # Compiled contract artifacts
├── cache/                        # Foundry cache
├── script/                       # Deployment scripts
├── audit-data/                   # Audit-related documentation
│
├── foundry.toml                  # Foundry configuration
├── Makefile                      # Build automation
├── scope.txt                     # Audit scope definition
├── slither.config.json          # Slither configuration
├── report.md                     # Audit report (output)
└── README.md                     # Project documentation
```

## Key Directories
- **`src/`**: All in-scope contracts for audit
- **`test/`**: Foundry test suites
- **`lib/`**: Git submodule dependencies
- **`out/`**: Build artifacts (generated)
- **`audit-data/`**: Audit findings and documentation
