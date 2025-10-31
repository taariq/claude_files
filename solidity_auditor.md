# SOLIDITY_AUDITOR.md

This file provides guidance to Claude Code when auditing Solidity smart contracts for security vulnerabilities and MEV exploits.

## Foundational Audit Principles

- Security is paramount. A missed vulnerability can result in millions of dollars lost.
- YOU MUST be systematic and thorough - never skip checks because code "looks fine"
- YOU MUST assume adversarial conditions - attackers will exploit any weakness
- YOU MUST think like an attacker: what would you do to drain funds or manipulate this contract?
- When in doubt about a potential vulnerability, YOU MUST flag it for Taariq to review

## Audit Methodology

### Phase 1: Initial Assessment

1. **Understand the Contract Purpose**
   - What is this contract supposed to do?
   - What assets does it control?
   - Who are the actors (users, admin, external contracts)?
   - What are the trust assumptions?

2. **Map the Attack Surface**
   - List all external/public functions
   - Identify all points where value flows in or out
   - Note all external calls and integrations
   - Document all privileged operations

3. **Identify Critical Invariants**
   - What properties MUST always be true? (e.g., "total shares equals sum of user shares")
   - What conditions would indicate the contract is broken?
   - What would constitute a successful exploit?

### Phase 2: Systematic Vulnerability Analysis

YOU MUST check for ALL of the following vulnerability classes:

#### 1. Reentrancy Vulnerabilities

**Check for:**
- Any external call before state updates (checks-effects-interactions violation)
- Missing reentrancy guards on functions that transfer value
- Cross-function reentrancy (function A calls external, which calls function B)
- Read-only reentrancy (exploiting stale state in view functions)

**Red flags:**
```solidity
// BAD: External call before state update
(bool success, ) = msg.sender.call{value: amount}("");
balance[msg.sender] = 0;

// GOOD: State update before external call
balance[msg.sender] = 0;
(bool success, ) = msg.sender.call{value: amount}("");
```

**Test strategy:**
- Create malicious receiver contracts with fallback/receive functions
- Attempt to call back into the target contract during execution
- Check if state can be exploited mid-execution

#### 2. Access Control Vulnerabilities

**Check for:**
- Missing or incorrect access modifiers (should be internal/private but is public)
- Weak authorization checks (e.g., `tx.origin` instead of `msg.sender`)
- Unprotected initialization functions
- Privilege escalation paths
- Default visibility (pre-0.5.0 functions default to public)

**Red flags:**
```solidity
// BAD: Using tx.origin for authorization
require(tx.origin == owner);

// BAD: Unprotected initialization
function initialize() public { owner = msg.sender; }

// GOOD: Protected initialization (OpenZeppelin pattern)
function initialize() public initializer { owner = msg.sender; }
```

**Test strategy:**
- Try calling privileged functions from non-privileged accounts
- Test initialization functions multiple times
- Check if admin can be changed by non-admin

#### 3. Integer Overflow/Underflow

**Check for:**
- Arithmetic operations without SafeMath (for Solidity < 0.8.0)
- Explicit `unchecked` blocks in Solidity >= 0.8.0
- Downcasting from larger to smaller types
- Multiplication before division (precision loss)

**Red flags:**
```solidity
// BAD (Solidity < 0.8.0): Can overflow
uint256 total = amount * multiplier;

// BAD (Solidity >= 0.8.0): Explicitly unsafe
unchecked { total = amount * multiplier; }

// GOOD: Safe by default (Solidity >= 0.8.0)
uint256 total = amount * multiplier;
```

**Test strategy:**
- Test with max uint256 values
- Test with zero and edge case values
- Verify precision loss doesn't allow exploits

#### 4. Front-Running and MEV Exploits

**Check for:**
- Price calculations that can be manipulated via front-running
- Slippage protection missing or insufficient
- Transaction ordering dependencies
- Mempool visibility of profitable transactions
- Oracle price manipulation via sandwich attacks

**Common MEV attack vectors:**

1. **Sandwich Attacks**
   - Attacker sees profitable trade in mempool
   - Front-runs with buy order
   - Victim's trade executes at worse price
   - Attacker back-runs with sell order

2. **Front-Running**
   - Liquidations (front-run to liquidate first)
   - Arbitrage opportunities
   - NFT mints or purchases
   - Governance votes

3. **Back-Running**
   - Oracle updates
   - Token listings
   - Liquidation cleanup

**Protection patterns:**
```solidity
// GOOD: Require minimum output amount (slippage protection)
function swap(uint256 amountIn, uint256 minAmountOut) external {
    uint256 amountOut = calculateSwap(amountIn);
    require(amountOut >= minAmountOut, "Slippage exceeded");
    // ... execute swap
}

// GOOD: Commit-reveal pattern for sensitive operations
function commit(bytes32 commitment) external {
    commitments[msg.sender] = commitment;
    commitBlock[msg.sender] = block.number;
}

function reveal(uint256 value, bytes32 salt) external {
    require(block.number > commitBlock[msg.sender], "Too early");
    require(keccak256(abi.encodePacked(value, salt)) == commitments[msg.sender], "Invalid reveal");
    // ... process value
}
```

**Test strategy:**
- Simulate front-running scenarios
- Test with manipulated price oracles
- Verify slippage protection works
- Check for flash loan attack vectors

#### 5. Oracle Manipulation

**Check for:**
- Reliance on single price oracle
- Using spot prices from DEXs (manipulable via flash loans)
- Missing staleness checks on oracle data
- No circuit breakers for extreme price movements

**Red flags:**
```solidity
// BAD: Using spot price directly (flash loan manipulable)
uint256 price = pair.getReserves();

// BAD: Single oracle without staleness check
uint256 price = oracle.getPrice();

// GOOD: Time-weighted average price (TWAP)
uint256 price = oracle.getTWAP(1800); // 30-minute TWAP

// GOOD: Multiple oracles with deviation checks
uint256 price1 = oracle1.getPrice();
uint256 price2 = oracle2.getPrice();
require(abs(price1 - price2) < maxDeviation, "Price deviation too high");
```

**Test strategy:**
- Simulate flash loan attacks to manipulate spot prices
- Test with stale oracle data
- Test with extreme price movements

#### 6. Flash Loan Attacks

**Check for:**
- Balance-based access control (can be bypassed with flash loans)
- Governance voting based on token balance snapshots
- Price calculations using manipulable reserves

**Red flags:**
```solidity
// BAD: Balance-based access control
require(token.balanceOf(msg.sender) >= threshold);

// GOOD: Time-locked balance or voting power
require(votingPower[msg.sender] >= threshold);
```

**Test strategy:**
- Simulate flash loan scenarios
- Test if governance can be manipulated
- Verify time-locks prevent flash loan exploits

#### 7. Denial of Service (DoS)

**Check for:**
- Unbounded loops over user-controlled arrays
- Revert conditions in loops (one bad actor can break everything)
- Gas limit DoS (block.gaslimit dependence)
- External call failures breaking critical functionality

**Red flags:**
```solidity
// BAD: Unbounded loop
for (uint256 i = 0; i < users.length; i++) {
    users[i].transfer(amount);
}

// BAD: One failure breaks everything
for (uint256 i = 0; i < recipients.length; i++) {
    payable(recipients[i]).transfer(amounts[i]); // If one fails, all fail
}

// GOOD: Pull over push pattern
function withdraw() external {
    uint256 amount = pendingWithdrawals[msg.sender];
    pendingWithdrawals[msg.sender] = 0;
    payable(msg.sender).transfer(amount);
}
```

**Test strategy:**
- Test with large arrays
- Test with failing external calls
- Verify gas limits aren't exceeded

#### 8. Delegate Call Vulnerabilities

**Check for:**
- `delegatecall` to user-controlled addresses
- Storage collision between proxy and implementation
- Unprotected `selfdestruct` in implementation contracts
- Missing storage gaps in upgradeable contracts

**Red flags:**
```solidity
// BAD: Delegatecall to user input
address(userProvidedAddress).delegatecall(data);

// BAD: Storage collision in proxy pattern
// Proxy stores 'owner' at slot 0
// Implementation also stores 'token' at slot 0 -> collision!
```

**Test strategy:**
- Verify proxy storage layout doesn't collide with implementation
- Test upgrade scenarios
- Ensure `selfdestruct` is protected

#### 9. Signature Replay Attacks

**Check for:**
- Missing nonce in signed messages
- Missing chain ID in signed messages (cross-chain replay)
- Missing contract address in signature (cross-contract replay)
- Signature malleability (using `recover` without checking `s` and `v` values)

**Red flags:**
```solidity
// BAD: No replay protection
function executeWithSignature(uint256 amount, bytes memory signature) external {
    bytes32 hash = keccak256(abi.encodePacked(amount));
    address signer = recoverSigner(hash, signature);
    require(signer == owner, "Invalid signature");
    // ... execute
}

// GOOD: Nonce + chain ID + contract address
function executeWithSignature(
    uint256 amount,
    uint256 nonce,
    bytes memory signature
) external {
    require(nonces[owner] == nonce, "Invalid nonce");
    bytes32 hash = keccak256(abi.encodePacked(
        amount,
        nonce,
        block.chainid,
        address(this)
    ));
    address signer = recoverSigner(hash, signature);
    require(signer == owner, "Invalid signature");
    nonces[owner]++;
    // ... execute
}
```

**Test strategy:**
- Try replaying signatures multiple times
- Test cross-chain replay (if applicable)
- Test signature malleability

#### 10. Uninitialized Storage Pointers

**Check for:**
- Uninitialized local struct/array variables (points to storage slot 0)
- Missing `storage` or `memory` keyword (older Solidity)

**Red flags:**
```solidity
// BAD (Solidity < 0.5.0): Points to storage slot 0
function vulnerable() public {
    User user; // Uninitialized - dangerous!
    user.balance = 100; // Writes to storage slot 0
}

// GOOD: Explicitly memory or storage
function safe() public {
    User memory user = User(100, msg.sender);
}
```

#### 11. tx.origin Authentication

**Check for:**
- Using `tx.origin` for authorization (vulnerable to phishing)

**Red flags:**
```solidity
// BAD: tx.origin can be exploited via phishing
require(tx.origin == owner);

// GOOD: Use msg.sender
require(msg.sender == owner);
```

#### 12. Unchecked External Calls

**Check for:**
- `call`, `delegatecall`, `send` without checking return value
- `transfer` used instead of `call` (2300 gas limit issues)

**Red flags:**
```solidity
// BAD: Ignoring return value
msg.sender.call{value: amount}("");

// BAD: Transfer has 2300 gas limit (breaks with smart contract wallets)
payable(msg.sender).transfer(amount);

// GOOD: Check return value
(bool success, ) = msg.sender.call{value: amount}("");
require(success, "Transfer failed");
```

#### 13. Timestamp Dependence

**Check for:**
- Critical logic depending on `block.timestamp` (miners can manipulate ~15 seconds)
- Using `block.number` for time measurements (varies by chain)

**Red flags:**
```solidity
// BAD for critical operations: Miner can manipulate
require(block.timestamp > deadline);

// ACCEPTABLE for long timeframes (hours/days)
require(block.timestamp > startTime + 1 days);
```

#### 14. Gas Griefing

**Check for:**
- External calls with all remaining gas
- Allowing receivers to consume arbitrary gas

**Test strategy:**
- Test with malicious contracts that consume all gas
- Verify gas limits are reasonable

### Phase 3: Business Logic Vulnerabilities

Beyond generic vulnerabilities, YOU MUST analyze business logic:

1. **Invariant Violations**
   - Can total supply become inconsistent?
   - Can accounting be manipulated?
   - Can funds be locked permanently?

2. **Economic Exploits**
   - Are there arbitrage opportunities?
   - Can rewards be gamed?
   - Are there edge cases in pricing formulas?

3. **Edge Cases**
   - What happens with zero amounts?
   - What happens when contract balance is zero?
   - What happens first/last user?
   - What happens at max/min values?

### Phase 4: Integration Vulnerabilities

**Check for:**
- Assumptions about external contracts (can they be manipulated?)
- ERC20 tokens with weird behavior (fee-on-transfer, rebasing, etc.)
- Approval race conditions (ERC20 approve/transferFrom)
- Contract addresses that could be upgraded

**Test strategy:**
- Test with weird ERC20 tokens
- Test with malicious external contracts
- Verify assumptions about external behavior

## Testing Requirements

### Required Test Coverage

YOU MUST create tests for:

1. **Exploit Proof of Concepts (PoCs)**
   - For each vulnerability found, write a test that demonstrates the exploit
   - The test should show funds being stolen or contract being broken
   - Include assertions proving the exploit worked

2. **Fuzzing**
   - Use Echidna or Foundry fuzzing for invariant testing
   - Define invariants that should always hold
   - Run for extended periods (10k+ runs)

3. **Edge Cases**
   - Zero amounts
   - Maximum amounts (type(uint256).max)
   - First user scenarios
   - Empty contract scenarios

4. **Integration Tests**
   - Test with actual mainnet forks
   - Test against real protocol integrations
   - Test with various ERC20 tokens

### Example Exploit PoC Structure

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/VulnerableContract.sol";

contract ExploitTest is Test {
    VulnerableContract target;
    Attacker attacker;

    function setUp() public {
        target = new VulnerableContract();
        attacker = new Attacker(address(target));

        // Fund target with 100 ETH
        vm.deal(address(target), 100 ether);
    }

    function testReentrancyExploit() public {
        // Attacker starts with 0 ETH
        assertEq(address(attacker).balance, 0);

        // Execute exploit
        attacker.exploit();

        // Attacker now has 100 ETH (drained from target)
        assertEq(address(attacker).balance, 100 ether);
        assertEq(address(target).balance, 0);
    }
}

contract Attacker {
    VulnerableContract target;

    constructor(address _target) {
        target = VulnerableContract(_target);
    }

    function exploit() external {
        // Describe the exploit step by step
        target.deposit{value: 1 ether}();
        target.withdraw(1 ether);
    }

    receive() external payable {
        // Reentrancy: withdraw again before state is updated
        if (address(target).balance >= 1 ether) {
            target.withdraw(1 ether);
        }
    }
}
```

## Reporting Requirements

### For Each Vulnerability Found

YOU MUST provide:

1. **Severity**: Critical / High / Medium / Low / Informational
   - **Critical**: Direct theft of funds or permanent freezing of funds
   - **High**: Funds at risk under specific conditions, or severe protocol disruption
   - **Medium**: State inconsistency, griefing, or minor fund loss
   - **Low**: Best practice violation or theoretical risk
   - **Informational**: Code quality, gas optimization, or style issues

2. **Title**: Clear, concise description

3. **Location**: File name, line numbers, function names

4. **Description**:
   - What is vulnerable?
   - Why is it vulnerable?
   - What are the preconditions?

5. **Impact**:
   - What can an attacker do?
   - How much funds are at risk?
   - What is the likelihood?

6. **Proof of Concept**:
   - Working code that demonstrates the exploit
   - Step-by-step explanation

7. **Remediation**:
   - Specific code changes to fix the vulnerability
   - Why this fix works
   - Any trade-offs or considerations

### Report Format

```markdown
## [SEVERITY] Title

**Location**: `contracts/Token.sol:123-145`, `withdraw()` function

**Description**:
The `withdraw()` function is vulnerable to reentrancy because it makes an external call to `msg.sender` before updating the user's balance. An attacker can create a malicious contract that calls `withdraw()` again in its `receive()` function, draining the contract.

**Impact**:
An attacker can steal all ETH held by the contract. With 100 ETH at risk, this is a critical vulnerability.

**Likelihood**: High - Easy to exploit, no special conditions required.

**Proof of Concept**:
[Include working exploit code]

**Recommendation**:
1. Update state before making external calls (checks-effects-interactions)
2. Add OpenZeppelin's `ReentrancyGuard` modifier
3. Consider using pull-over-push pattern

Fixed code:
[Show the corrected code]
```

## Tools and Resources

### Recommended Tools

- **Static Analysis**:
  - Slither (must run on every audit)
  - Mythril
  - Securify

- **Fuzzing**:
  - Echidna (invariant fuzzing)
  - Foundry fuzzing (`forge test --fuzz`)

- **Testing Frameworks**:
  - Foundry (preferred)
  - Hardhat

- **Mainnet Fork Testing**:
  - Foundry's `--fork-url`
  - Test against real deployed contracts

### Commands

```bash
# Run Slither static analysis
slither . --exclude-dependencies

# Run Foundry tests with high fuzz runs
forge test --fuzz-runs 10000

# Test against mainnet fork
forge test --fork-url $RPC_URL --match-test testExploit

# Generate gas report
forge test --gas-report

# Get code coverage
forge coverage
```

## MEV-Specific Analysis

### MEV Attack Patterns to Check For

1. **Liquidation Front-Running**
   - Can liquidations be front-run profitably?
   - Are liquidation incentives too high (encourages front-running)?

2. **Arbitrage Opportunities**
   - Are there internal price discrepancies?
   - Can flash loans be used to arbitrage?

3. **Governance Attacks**
   - Can proposals be front-run?
   - Can large holders manipulate outcomes?

4. **Oracle Manipulation for MEV**
   - Can attackers profit from stale oracle prices?
   - Can oracle updates be front-run?

### MEV Protection Patterns

```solidity
// Pattern 1: Minimum/Maximum Bounds
function swap(uint256 amountIn, uint256 minOut, uint256 maxIn) external {
    require(amountIn <= maxIn, "Amount too high");
    uint256 amountOut = getAmountOut(amountIn);
    require(amountOut >= minOut, "Slippage too high");
    // ... execute
}

// Pattern 2: Deadline Protection
function execute(uint256 deadline) external {
    require(block.timestamp <= deadline, "Expired");
    // ... execute
}

// Pattern 3: Private Transactions (Flashbots)
// Document if contract should be used with Flashbots Protect

// Pattern 4: Batch Auctions (avoid continuous trading)
function submitOrder() external {
    // Orders collected, executed in batch at specific block
}
```

## Final Checklist

Before completing an audit, verify:

- [ ] All external/public functions reviewed
- [ ] All external calls analyzed
- [ ] All state changes checked (reentrancy, race conditions)
- [ ] All arithmetic operations verified (overflow/underflow)
- [ ] All access control reviewed
- [ ] Slither run with no critical issues
- [ ] Exploit PoCs written for all findings
- [ ] Integration tests with mainnet fork passed
- [ ] MEV analysis completed
- [ ] Gas optimization opportunities identified
- [ ] Documentation and comments reviewed
- [ ] Upgrade patterns reviewed (if applicable)
- [ ] Emergency pause mechanisms tested

## Remember

- **Never** assume code is safe because it "looks right"
- **Never** skip a check because you're confident
- **Always** write exploit PoCs for vulnerabilities
- **Always** consider the attacker's perspective
- **Always** verify fixes with tests before marking resolved

The goal is not to find vulnerabilities quickly - it's to find ALL vulnerabilities thoroughly.
