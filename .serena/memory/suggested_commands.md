# Suggested Commands

## Essential Development Commands

### Building & Compilation
```bash
make build          # Compile contracts
forge build         # Alternative compilation
make clean          # Clean build artifacts
```

### Testing
```bash
make test           # Run all tests
forge test          # Alternative test command
forge test -vvv     # Verbose test output
forge test --match-test <testName>  # Run specific test
```

### Test Coverage
```bash
forge coverage                  # Generate coverage report
forge coverage --report debug   # Detailed coverage analysis
make snapshot                   # Create gas snapshot
```

### Security Analysis
```bash
make slither        # Run Slither static analysis
make aderyn         # Run Aderyn security scanner
```

### Code Formatting
```bash
make format         # Format Solidity code
forge fmt           # Alternative format command
```

### Local Development
```bash
make anvil          # Start local blockchain
                    # Uses mnemonic: "test test test test test test test test test test test junk"
                    # Block time: 1 second
                    # Steps tracing enabled
```

### Dependency Management
```bash
make install        # Install forge-std and OpenZeppelin
make update         # Update dependencies
make remove         # Remove all dependencies and modules
```

### Project Scope
```bash
make scope          # Display source tree
make scopefile      # Generate scope.txt file
```

## macOS-Specific Utilities
```bash
ls -la              # List files with details
find . -name "*.sol"  # Find Solidity files
grep -r "pattern" src/  # Search in source
tree src/           # View directory tree (if installed)
git status          # Check git status
git log             # View commit history
```

## Audit Workflow Commands
```bash
# 1. Setup
make install && make build

# 2. Run tests
make test

# 3. Check coverage
forge coverage

# 4. Security analysis
make slither
make aderyn

# 5. Review findings
cat report.md
```
