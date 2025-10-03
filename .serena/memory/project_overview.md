# Boss Bridge - Project Overview

## Purpose
Boss Bridge is a **Layer 1 to Layer 2 bridge protocol** designed to transfer ERC20 tokens from Ethereum L1 to a custom L2 network. This is an **audit-focused security project** for analyzing smart contract vulnerabilities.

## Key Functionality
- **Token Deposits (L1 → L2)**: Users deposit tokens into L1Vault, triggering off-chain minting on L2
- **Token Withdrawals (L2 → L1)**: Authorized signers approve withdrawal requests via cryptographic signatures
- **Security Mechanisms**:
  - Owner can pause/unpause in emergencies
  - Deposit limit of 100,000 tokens
  - Signer-based withdrawal authorization
  - ReentrancyGuard protection

## Target Deployment
- **Ethereum Mainnet**: L1BossBridge, L1Token, L1Vault, TokenFactory
- **ZKSync Era**: TokenFactory only

## Audit Scope
Commit: `07af21653ab3e8a8362bf5f63eb058047f562375`

**In-Scope Contracts** (all in `src/`):
- `L1BossBridge.sol` - Main bridge contract with deposit/withdrawal logic
- `L1Token.sol` - Standard ERC20 token (1M initial supply)
- `L1Vault.sol` - Token custody contract
- `TokenFactory.sol` - Deploys new ERC20 tokens via assembly

## Key Actors
- **Bridge Owner**: Can pause/unpause, set signers
- **Signers**: Authorize L2 → L1 token transfers
- **Vault**: Holds locked tokens
- **Users**: Deposit tokens to L2

## Known Design Decisions
- Centralized bridge ownership (acknowledged)
- Missing zero-address checks for gas optimization
- Magic numbers as literals (not constants)
- Only L1Token.sol and copies supported (weird ERC20s out of scope)
