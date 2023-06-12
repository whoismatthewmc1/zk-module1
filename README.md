# yAcademy Rate-Limit Nullifier Review

**Auditors:**

 - Parsely
 - lwltea
 - whoismatthewmc1

## Table of Contents <!-- omit in toc -->

1. [Review Summary](#review-summary)
2. [Scope](#scope)
3. [Assumptions](#assumptions)
4. [Issues not addressed](#issues-not-addressed)
5. [Tools used](#tools-used)
6. [Code Evaluation Matrix](#code-evaluation-matrix)
7. [Findings Explanation](#findings-explanation)
    - [Low 1](#low1)
    - [Low 2](#low2)
    - [Low 3](#low3)
    - [Gas 1](#gas1)
    - [Gas 2](#gas2)
    - [Gas 3](#gas3)
    - [Informational 1](#informational1)
    - [Informational 2](#informational2)
8. [Final remarks](#final-remarks)
9. [CircomSpect Output](#circomSpect-output)

## Review Summary

**Rate-Limit Nullifier**

Rate-Limit Nullifier provides a mechanism by which a user can stake an amount of ERC20 tokens in exchange for the right to send anonymous messages off-chain, the staked amount denotes an agreement to limit the number of messages they can send off-chain to a certain number during each epoch.

The circuits of the [Rate-Limit Nullifier Github](https://github.com/zBlock-1/circom-rln) as well as the accompanying [RLN contracts repo](https://github.com/Rate-Limiting-Nullifier/rln-contracts) were reviewed over 15 days. The code review was performed by 3 auditors between 31st May, 2023 and 14th June, 2023. The repository was static during the review.

The code was very well written and commented, and followed the specification documents correctly.

## Scope

The scope of the review consisted of the following circuits and contracts within the repo:

**Circuits**
- rln.circom
- utils.circom
- withdraw.circom

**Contracts**
- RLN.sol
- IVerifier.sol

This review is a code review to identify potential vulnerabilities in the code. The reviewers did not investigate security practices or operational security and assumed that privileged accounts could be trusted. The review was done based on the official specification. The review may not have identified all potential attack vectors or areas of vulnerability.

yAcademy and the auditors make no warranties regarding the security of the code and do not warrant that the code is free from defects. yAcademy and the auditors do not represent nor imply to third parties that the code has been audited nor that the code is free from defects. By deploying or using the code, Rate-Limit Nullifier and users of the circuits agree to use the code at their own risk.

## Assumptions
- We have assumed that the trusted setup will be done correctly and all relevant artefacts kept safe. 

## Issues not addressed
- We have not addressed the fact that a user can self-slash using a different receiver wallet address, as the user will then forfeit the fees portion of their stake.

## Tools used
- Manual review
- Circomspect, although limited, did produce the findings at the bottom of the report

## Code Evaluation Matrix
---

| Category                 | Mark    | Description |
| ------------------------ | ------- | ----------- |
| Mathematics              | Good | Correctly implemented |
| Complexity               | Good | Not coded to make it complex, kept as simple as possible |
| Code stability           | Good    | No issues found |
| Documentation            | Very Good | Clear, concise and well-written |
| Testing and verification | Good |  All needed tests implemented |

## Findings Explanation
3 low impact bugs, 3 Gas optimizations, and 2 Informational findings are being reported.

Findings are broken down into sections by their respective impact:

---

## Low Findings

### 1. Low - Difference between documents and implementation<a name="low1"></a>

the documantation mentions that the rate limit is between 1 and userMessageLimit:
```text
Signaling will use other circuit, where your limit is private input, and the counter k is checked that it's in the range from 1 to userMessageLimit.
```

Although the implementation is sound, the counter `k` should be documented as being between `0` to `userMessageLimit-1`
Testing the circuit allows for a messageid of 0  and does not allow for a messageId of `userMessageLimit` as tested in zkrepl.dev
```
pragma circom 2.1.4;

include "circomlib/bitify.circom";
// include "https://github.com/0xPARC/circom-secp256k1/blob/master/circuits/bigint.circom";

template RangeCheck(LIMIT_BIT_SIZE) {
    assert(LIMIT_BIT_SIZE < 253);

    signal input messageId;
    signal input limit;

    signal bitCheck[LIMIT_BIT_SIZE] <== Num2Bits(LIMIT_BIT_SIZE)(messageId);
    log(bitCheck[LIMIT_BIT_SIZE]);
    signal rangeCheck <== LessThan(LIMIT_BIT_SIZE)([messageId, limit]);
    log(rangeCheck);
    rangeCheck === 1;
}

component main = RangeCheck(252);

/* INPUT = {
    "messageId": "0",
    "limit": "5"
} */
```
If the code above is run with an input messageId of "0" it passes the assertion.

#### Technical Details

The code:
```
signal rangeCheck <== LessThan(LIMIT_BIT_SIZE)([messageId, limit]);
```
will always return true for a messageId of "0".

#### Impact

Low. there is no impact except for clarity to potential developers/users.

#### Recommendation

The project may consider chaging the wording to be something like :
```text
Signaling will use other circuit, where your limit is private input, and the counter k is checked that it's in the range from 0 to userMessageLimit -1.
```

#### Developer Response


### 2. Low - It is possible that unused public input may be optimized out by the compiler<a name="low2"></a>

According to the common vulnerabilites list on the [0xParc github #5](https://github.com/0xPARC/zk-bug-tracker#5-unused-public-inputs-optimized-out) it is possbile that unused public inputs may be optimised out.

#### Technical Details
The withdraw circuit includes a public input for ```address``` to prevent front-running by a withdrawer/slasher.
```
template Withdraw() {
    signal input identitySecret;
    signal input address;

    signal output identityCommitment <== Poseidon(1)([identitySecret]);
}
```
```address``` is unused in the circuit.

#### Impact
Low. It seems to be hypothetical and I was unable to recreate it.

#### Recommendation

As per the Tornado cash Resolution the project may consider adding a constraint just to have the value used within the circuit.
```
template Withdraw() {
    signal input identitySecret;
    signal input address;

    signal output identityCommitment <== Poseidon(1)([identitySecret]);
    signal addressSquared <== address * address;
}
```

#### Developer Response

### 3. Low - Contract RLN inherits from Ownable but ownable functionality isn't actually used by the contract.<a name="low3"></a>

**REPORTED BY lwltea**:

Contract RLN inherits from Ownable but ownable functionality isn't actually used by the contract.

#### Technical Details
The withdraw circuit includes a public input for ```address``` to prevent front-running by a withdrawer/slasher.
```
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import {IVerifier} from "./IVerifier.sol";

/// @title Rate-Limiting Nullifier registry contract
/// @dev This contract allows you to register RLN commitment and withdraw/slash.
contract RLN is Ownable {
    using SafeERC20 for IERC20;
```
Ownable is never used within the RLN contract.

#### Impact
Low.

#### Recommendation

Remove ```Ownable``` import and inheritance from RLN.sol

#### Developer Response

## Gas Savings Findings

### 1. Gas - Custom errors are more gas efficient than `require` statements.<a name="gas1"></a>

**REPORTED BY lwltea**:

Custom errors are more gas efficient than `require` statements.

#### Technical Details
According to [https://blog.soliditylang.org/2021/04/21/custom-errors/](https://blog.soliditylang.org/2021/04/21/custom-errors/) custom errors are more gas efficient than require statements

#### Impact
Low.

#### Recommendation

Consider refactoring code in Solidity contracts to rather use Custom Errors

#### Developer Response

### 2. Gas - Incrementing within an unchecked block saves gas.<a name="gas2"></a>

**REPORTED BY lwltea**:

Put `identityCommitmentIndex += 1;` in a unchecked block as it is a uint256 being incremented by 1. Range checks are unnecessary here.

#### Technical Details
In RLN.sol [https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L126](https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L126)
```
    identityCommitmentIndex += 1;
```
Put `identityCommitmentIndex += 1;` in an unchecked block as its a uint256 being incremented by 1. Range checks are unnecessary here

#### Impact
Low.

#### Recommendation

Consider using the an unchecked block for incrementing `identityCommitmentIndex`

#### Developer Response

### 3. Gas - Unnecessary initialization of default uint256<a name="gas3"></a>

**REPORTED BY whoismatthewmc1**:

Remove the unnecessary initialization of `identityCommitmentIndex = 0` since its default value will be 0.

#### Technical Details
In RLN.sol [https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L52](https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L52)
```
    uint256 public identityCommitmentIndex = 0;
```
costs gas to re-assign `identityCommitmentIndex = 0;` when the default uint256 value is already set to 0.

#### Impact
Low.

#### Recommendation
Change the code at line 52 as follows:
```
-   uint256 public identityCommitmentIndex = 0;
+   uint256 public identityCommitmentIndex;
```

#### Developer Response


## Informational Findings

### 1. Informational - Consider adding appropriate require statements for RLN.sol constructor params<a name="informational1"></a>

Params such as `minimalDeposit`, `feePercentage`, `freezePeriod`, and `feeReceiver` should have appropriate restrictions or bounds.

#### Technical Details

A few cases to cover:
- `MINIMAL_DEPOSIT = 0` can cause a DoS in the `register` function, rendering the contract useless.
https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L121
```
        uint256 messageLimit = amount / MINIMAL_DEPOSIT;
```

- `FEE_PERCENTAGE > 100` will cause a DoS in the `slash` function.
https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L173-L176
```
        uint256 withdrawAmount = member.messageLimit * MINIMAL_DEPOSIT;
        uint256 feeAmount = (FEE_PERCENTAGE * withdrawAmount) / 100;

        token.safeTransfer(receiver, withdrawAmount - feeAmount);
```
with `FEE_PERCENTAGE > 100`, `feeAmount` becomes greater than `withdrawAmount`, causing a revert on `withdrawAmount - feeAmount`.

- `FEE_RECEIVER = address(0)` causes a DoS in the `slash` function due to transfers requiring a non-0 address.
https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L177
```
        token.safeTransfer(FEE_RECEIVER, feeAmount);
```

- A very high `FREEZE_PERIOD` may not release withdrawals within a "reasonable" time frame.

#### Impact

Informational.
While these would all require the contract deployer to input strange params upon deployment, it is nevertheless possible and could result in contract DoS.

#### Recommendation

Consider adding appropriate bounds via `require` or `revert` statements where applicable. ie:
- `minimalDeposit != 0`
- `feeReceiver != address(0)`
- `feePercentage < 100`
- `freezePeriod < 3e7` (ie: ~1 year assuming 1s block times on an L2)

Additionally, if desired, `freezePeriod` and `FREEZE_PERIOD` can likely be reduced in size and produce gas savings if the change reduces the contract's required storage slots.

### 2. Informational - Weird ERC20 tokens may cause unexpected behaviour in RLN.sol<a name="informational2"></a>

Blacklisted `FEE_RECEIVER` can DoS `slash` functionality:
- Certain ERC20 tokens may include a blacklist feature for compliance or other reasons.
- If the `FEE_RECEIVER` happens to be part of that blacklist, the `slash` function will always revert.

Fee-on-transfer tokens will result in the contract holding less tokens than it expects:
- Certain ERC20 tokens may charge a tax on transfers.
- `register` and `withdraw` do not verify that the contract is holding enough tokens.

#### Technical Details

**Blacklist**
The following will revert on a blacklisted `FEE_RECEIVER` address:
https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L177
```
        token.safeTransfer(FEE_RECEIVER, feeAmount);
```

**Fee-on-transfer**
Imagine the following scenario:
- `token` is set to an ERC20 token that charges a 10% transfer tax
- `MINIMAL_DEPOSIT` is 1 and `register` is called with `amount = 100`
- After the transfer is made as part of the `register` function: https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L120, the contract holds 90 tokens and `members[identityCommitment].messageLimit = 100`
- The user creates a withdrawal, leading to `withdrawals[identityCommitment].amount = 100`
- Finally on `release` (or `slash`), the contract does not hold enough tokens to fulfill the transfer request, causing a revert: https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L154 & https://github.com/zBlock-1/rln-contracts/blob/main/src/RLN.sol#L176.

Note that if multiple users have registered, this issue may not appear until many users have withdrawn or been slashed.

#### Impact

Informational.

With blacklisted `FEE_RECEIVER`, in the worst case, a new instance of the contract will need to be deployed with a non-blacklisted `FEE_RECEIVER` if `slash` functionality is to be restored. Users will still be able to withdraw their stakes from the original contract.

For fee-on-transfer tokens, possible loss of funds for users choosing to interact with the protocol. However, interacting with fee-on-transfer tokens generally presents a similar risk as this.

#### Recommendation
Consider documenting or restricting the tokens that the RLN.sol contract may use to exclude tokens that have blacklists or that charge a fee on transfer.
Specifically to exclude fee-on-transfer tokens, a check can be added to verify the different in contract balance before and after the transfer is made in `register` at L120.

### 3. Informational - In RLN.sol, users can create a withdrawal immediately after registering<a name="informational3"></a>

#### Technical Details

Nothing stops users from immediately creating a withdrawal after they register, allowing them to avoid the `FREEZE_PERIOD` usually associated with a `release` call.

#### Impact

Informational.

#### Recommendation
Consider documenting that external applications should disallow `identityCommitment`s from sending messages if a withdrawal has already been created for them.

## Final remarks

The code is very well written and did not present any vulnerabilites that would cause significant impact. The logic is easy to follow and concise. It was a pleasure to learn by auditing this codebase.

I have looked into the vulnerability [discussed](https://github.com/0xPARC/zk-bug-tracker#1-dark-forest-v03-missing-bit-length-check) about the ```LessThan``` circomlib template, however since the values are first checked with ```Num2Bits```, this negates any possible attacks.


## CircomSpect Output
### RLN circuit
```
circomspect: analyzing template 'RLN'
note: Field element arithmetic could overflow, which may produce unexpected results.
   ┌─ /home/testadmin/Desktop/zk/rln/circom-rln/circuits/rln.circom:34:11
   │
34 │     y <== identitySecret + a1 * x;
   │           ^^^^^^^^^^^^^^^^^^^^^^^ Field element arithmetic here.
   │
   = For more details, see https://github.com/trailofbits/circomspect/blob/main/doc/analysis_passes.md#field-element-arithmetic.

circomspect: 1 issue found.
```

### Utils circuit
```
circomspect: analyzing template 'RangeCheck'
note: Comparisons with field elements greater than `p/2` may produce unexpected results.
   ┌─ /home/testadmin/Desktop/zk/rln/circom-rln/circuits/utils.circom:37:12
   │
37 │     assert(LIMIT_BIT_SIZE < 253);
   │            ^^^^^^^^^^^^^^^^^^^^ Field element comparison here.
   │
   = Field elements are always normalized to the interval `(-p/2, p/2]` before they are compared.
   = For more details, see https://github.com/trailofbits/circomspect/blob/main/doc/analysis_passes.md#field-element-comparison.

warning: Intermediate signals should typically occur in at least two separate constraints.
   ┌─ /home/testadmin/Desktop/zk/rln/circom-rln/circuits/utils.circom:42:5
   │
42 │     signal bitCheck[LIMIT_BIT_SIZE] <== Num2Bits(LIMIT_BIT_SIZE)(messageId);
   │     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   │     │
   │     The intermediate signal array `bitCheck` is declared here.
   │     The intermediate signals in `bitCheck` are constrained here.
   │
   = For more details, see https://github.com/trailofbits/circomspect/blob/main/doc/analysis_passes.md#under-constrained-signal.

warning: Using `Num2Bits` to convert field elements to bits may lead to aliasing issues.
   ┌─ /home/testadmin/Desktop/zk/rln/circom-rln/circuits/utils.circom:42:41
   │
42 │     signal bitCheck[LIMIT_BIT_SIZE] <== Num2Bits(LIMIT_BIT_SIZE)(messageId);
   │                                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Circomlib template `Num2Bits` instantiated here.
   │
   = Consider using `Num2Bits_strict` if the input size may be >= than the prime size.
   = For more details, see https://github.com/trailofbits/circomspect/blob/main/doc/analysis_passes.md#non-strict-binary-conversion.

circomspect: analyzing template 'MerkleTreeInclusionProof'
circomspect: 3 issues found.
```

### Withdraw circuit
```
circomspect: analyzing template 'Withdraw'
warning: The signal `address` is not used by the template.
  ┌─ /home/testadmin/Desktop/zk/rln/circom-rln/circuits/withdraw.circom:7:5
  │
7 │     signal input address;
  │     ^^^^^^^^^^^^^^^^^^^^ This signal is unused and could be removed.
  │
  = For more details, see https://github.com/trailofbits/circomspect/blob/main/doc/analysis_passes.md#unused-variable-or-parameter.

circomspect: 1 issue found.
~~~
