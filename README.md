# yAcademy Rate-Limit Nullifier Review

**Review Resources:**

- TODO, generic text is: None beyond the code repositories

**Auditors:**

 - Parsely

## Table of Contents <!-- omit in toc -->

TODO_Insert_TOC

## Review Summary

**Rate-Limit Nullifier**

Rate-Limit Nullifier provides a mechanism in which a user can stake an amount of ERC20 tokens in exchange for the right to send anonymous messages off-chain, the staked amount denotes an agreement to limit the number of messages they can send off-chain to a certain number during each epoch.

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
- We have assumed that the trusted setup will be done correctly and all needed artefacts kept safe. 
- We have assumed that the node operators have restricted access and are trusted.

## Issues not addressed
- We have not addressed the fact that a user can self-slash using a different receiver wallet address, as the user will then forfeit the fees portion of their stake.

Code Evaluation Matrix
---

| Category                 | Mark    | Description |
| ------------------------ | ------- | ----------- |
| Access Control           | Good | TODO |
| Mathematics              | Good | TODO |
| Complexity               | Good | TODO |
| Decentralization         | Good | TODO |
| Code stability           | Good    | TODO |
| Documentation            | Very Good | TODO |
| Testing and verification | Average | TODO  |

## Findings Explanation

Findings are broken down into sections by their respective impact:
---

## Low Findings

### 1. Low - Difference between documents and implementation

the documantation mentions that the rate limit is between 1 and k:
```text
Signaling will use other circuit, where your limit is private input, and the counter k is checked that it's in the range from 1 to userMessageLimit.
```
Although the implementation is sound, the counter ```k``` should be documented as being between ```0``` to ```k-1```
However the circuit allows for a messageid of 0 as tested in zkrepl.dev
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

the project may consider chaging the wording to be something like :
```text
Signaling will use other circuit, where your limit is private input, and the counter k is checked that it's in the range from 0 to userMessageLimit -1.
```

#### Developer Response



### 2. Low - TODO_Title

TODO

#### Technical Details

TODO

#### Impact

Low. TODO_reasoning.

#### Recommendation

TODO

#### Developer Response



## Gas Savings Findings

### 1. Gas - TODO_Title

TODO

#### Technical Details

TODO

#### Impact

Gas savings.

#### Recommendation

TODO

## Informational Findings

### 1. Informational - TODO_Title

TODO

#### Technical Details

TODO

#### Impact

Informational.

#### Recommendation

TODO

## Final remarks

TODO
