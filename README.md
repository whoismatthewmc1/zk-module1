# yAcademy Rate-Limit Nullifier Review

**Auditors:**

 - Parsely

## Table of Contents <!-- omit in toc -->

TODO_Insert_TOC

## Review Summary

**Rate-Limit Nullifier**

Rate-Limit Nullifier provides a mechanism by which a user can stake an amount of ERC20 tokens in exchange for the right to send anonymous messages off-chain, the staked amount denotes an agreement to limit the number of messages they can send off-chain to a certain number during each epoch.

The circuits of the Rate-Limit Nullifier [circom-rln](https://github.com/zBlock-1/circom-rln) were reviewed over 15 days. The code review was performed by 1 auditor between 31st May, 2023 and 14th June, 2023. The repository was static during the review.

The code was very well written and commented, and followed the specification documents correctly.

## Scope

The scope of the review consisted of the following circuits within the repo:

- **Circuits**
- rln.circom
- utils.circom
- withdraw.circom


This review is a code review to identify potential vulnerabilities in the code. The reviewers did not investigate security practices or operational security and assumed that privileged accounts could be trusted. The review was done based on the official specification. The review may not have identified all potential attack vectors or areas of vulnerability.

yAcademy and the auditors make no warranties regarding the security of the code and do not warrant that the code is free from defects. yAcademy and the auditors do not represent nor imply to third parties that the code has been audited nor that the code is free from defects. By deploying or using the code, Rate-Limit Nullifier and users of the circuits agree to use the code at their own risk.

## Assumptions
- We have assumed that the trusted setup will be done correctly and all relevant artefacts kept safe. 
- We have assumed that the node operators have restricted access and/or are trusted.

## Issues not addressed
- We have not addressed the fact that a user can self-slash using a different receiver wallet address, as the user will then forfeit the fees portion of their stake.

## Tools used
- Manual review
- Circomspect, although limited did prduce the findings at the bottom of the report

Code Evaluation Matrix
---

| Category                 | Mark    | Description |
| ------------------------ | ------- | ----------- |
| Mathematics              | Good | correctly implemented |
| Complexity               | Good | Not coded to make it complex, kept as simple as possible |
| Code stability           | Good    | No issues found |
| Documentation            | Very Good | Clear,concise and well-written |
| Testing and verification | Good |  All needed tests implemented |

## Findings Explanation

Findings are broken down into sections by their respective impact:
---

## Low Findings

### 1. Low - Difference between documents and implementation

the documantation mentions that the rate limit is between 1 and userMessageLimit:
```text
Signaling will use other circuit, where your limit is private input, and the counter k is checked that it's in the range from 1 to userMessageLimit.
```
Although the implementation is sound, the counter ```k``` should be documented as being between ```0``` to ```userMessageLimit-1```
Testing the circuit allows for a messageid of 0  and does not allow for a messageId of ```userMessageLimit``` as tested in zkrepl.dev
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
signal rangeCheck <== LessThan(LIMIT_BIT_SIZE)([messageId, limit]);```
will always return true for a messageId of  "0".

#### Impact

Low. there is no impact except for clarity to potential developers/users.

#### Recommendation

The project may consider chaging the wording to be something like :
```text
Signaling will use other circuit, where your limit is private input, and the counter k is checked that it's in the range from 0 to userMessageLimit -1.
```

#### Developer Response



### 2. Low - It is possible that unused public input may be optimized out by the compiler

According to the comman vulnerabilites list on the [0xParc github #5](https://github.com/0xPARC/zk-bug-tracker#5-unused-public-inputs-optimized-out) it is possbile that unused public inputs may be optimised out.

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
    signal addressSquared == address * address;
}
```

#### Developer Response

### 3. Low - It is possible that unused public input may be optimized out by the compiler

According to the comman vulnerabilites list on the [0xParc github #5](https://github.com/0xPARC/zk-bug-tracker#5-unused-public-inputs-optimized-out) it is possbile that unused public inputs may be optimised out.

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
    signal addressSquared == address * address;
}
```

#### Developer Response

## Final remarks

The code is very well written and did not present any vulnerabilites that would cause significant impact. The logic is easy to follow and concise. It was a pleasure to learn by auditing this codebase.

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
