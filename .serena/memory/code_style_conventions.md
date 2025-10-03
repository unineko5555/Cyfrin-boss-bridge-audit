# Code Style & Conventions

## Solidity Version
- **Version**: `0.8.20` (strict)
- **License**: MIT

## Formatting Rules (from foundry.toml)
```toml
bracket_spacing = true          # { foo } instead of {foo}
int_types = "long"             # uint256 instead of uint
line_length = 120              # Maximum line length
multiline_func_header = "all"  # Multi-line function signatures
number_underscore = "thousands" # 100_000 instead of 100000
quote_style = "double"         # "string" instead of 'string'
tab_width = 4                  # 4 spaces per indent
wrap_comments = true           # Wrap long comments
```

## Naming Conventions

### Contracts
- PascalCase: `L1BossBridge`, `L1Token`, `L1Vault`

### Functions
- camelCase: `depositTokensToL2`, `withdrawTokensToL1`, `setSigner`

### Variables
- **Public/External**: camelCase (`token`, `vault`, `signers`)
- **Private/Internal**: Prefix with `s_` for storage (e.g., `s_tokenToAddress` in TokenFactory)
- **Constants**: UPPER_SNAKE_CASE (`DEPOSIT_LIMIT`, `INITIAL_SUPPLY`)
- **Immutable**: lowercase (`token`, `vault`)

### Custom Errors
- Format: `ContractName__ErrorReason`
- Examples:
  - `L1BossBridge__DepositLimitReached`
  - `L1BossBridge__Unauthorized`
  - `L1BossBridge__CallFailed`

### Events
- PascalCase: `Deposit`, `TokenDeployed`

## Documentation Style

### NatSpec Comments
```solidity
/**
 * @notice Brief description
 * @dev Technical implementation details
 * @param paramName Parameter description
 * @return returnValue Return value description
 */
```

### Single-line comments
```solidity
// Explanation of the following code
```

## Import Style
- Use named imports from OpenZeppelin:
```solidity
import { IERC20 } from "@openzeppelin/contracts/interfaces/IERC20.sol";
import { Ownable } from "@openzeppelin/contracts/access/Ownable.sol";
```

## Security Patterns
- **Access Control**: Use `Ownable` for admin functions
- **Pausability**: Use `Pausable` for emergency stops
- **Reentrancy**: Use `ReentrancyGuard` for external calls
- **Safe Transfers**: Use `SafeERC20` for token operations

## Known Deviations (Intentional)
- Missing zero-address checks (gas optimization)
- Magic numbers as literals (not constants)
- Centralized ownership model
