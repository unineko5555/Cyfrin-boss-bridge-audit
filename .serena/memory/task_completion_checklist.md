# Task Completion Checklist

## When Completing Any Code Task

### 1. Code Quality
- [ ] Follow naming conventions (see code_style_conventions.md)
- [ ] Add NatSpec documentation for new functions
- [ ] Use OpenZeppelin security patterns
- [ ] Follow 120-character line length limit
- [ ] Use double quotes for strings
- [ ] Use number underscores (100_000 not 100000)

### 2. Testing
- [ ] Run tests: `forge test`
- [ ] Verify all tests pass
- [ ] Check gas usage: `forge snapshot` (if relevant)
- [ ] Review test coverage: `forge coverage`

### 3. Code Formatting
- [ ] Format code: `make format` or `forge fmt`
- [ ] Verify no formatting violations

### 4. Security Analysis
- [ ] Run Slither: `make slither`
- [ ] Run Aderyn: `make aderyn`
- [ ] Review and address critical/high findings
- [ ] Document accepted risks for low/medium findings

### 5. Compilation
- [ ] Clean build: `make clean && make build`
- [ ] Verify no compilation errors
- [ ] Check compiler warnings

### 6. Documentation
- [ ] Update relevant comments
- [ ] Document security assumptions
- [ ] Note any known issues
- [ ] Update README.md if needed (only if explicitly requested)

### 7. Version Control
- [ ] Review changes: `git diff`
- [ ] Stage files: `git add <files>`
- [ ] Commit with descriptive message
- [ ] Verify commit scope matches audit scope

## Security Audit Specific Checklist

### When Analyzing Contracts
- [ ] Check for reentrancy vulnerabilities
- [ ] Verify access control (onlyOwner usage)
- [ ] Review signature validation logic
- [ ] Check for front-running risks
- [ ] Analyze deposit/withdrawal flows
- [ ] Verify event emissions
- [ ] Check for integer overflow/underflow
- [ ] Review external calls safety
- [ ] Validate input parameters
- [ ] Check for replay attack prevention

### When Writing Audit Reports
- [ ] Categorize severity (Critical/High/Medium/Low/Informational)
- [ ] Provide clear vulnerability description
- [ ] Include proof of concept (PoC)
- [ ] Suggest mitigation strategies
- [ ] Reference specific line numbers
- [ ] Test exploits in isolated environment

## Quick Pre-Commit Commands
```bash
# Run all quality checks
make format && make build && make test && make slither
```
