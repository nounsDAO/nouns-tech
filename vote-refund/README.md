# Vote Refund

## Simple Summary

Atomically refund vote transaction costs to governance participants.

## Abstract

This specification introduces four functions to the governance logic contract. These include two additional vote functions, which atomically refund vote transaction costs to governance participants, an ether withdrawal function, and a receive ether function.

The governance logic contract will need to be filled with ETH in order to send refunds. As an example, this can be done by sending ETH from the treasury via a proposal.

This feature will NOT refund voters for past transaction costs.

## Technical Specification

### Overview

Vote Functions:

1. `castRefundableVote` - Cast a refundable vote for a proposal.
2. `castRefundableVoteWithReason` - Cast a refundable vote for a proposal with a reason.

Rules:

1. `msg.sender` must only be refunded if they have at least one vote. This prevents the depletion of refund reserves by non-vote holders at no cost.
2. The gas price used in the refund calculation must be capped using a `MAX_REFUND_PRIORITY_FEE` constant. This enables the transaction to succeed while providing a partial refund if a high priority fee is used.
3. No refund will be provided if the governance contract has no balance.
4. A partial refund will be provided if the contract has a non-zero balance that is less than the full refund amount.
5. Refund transfer failures must be ignored. The transaction must succeed even though no refund has been provided.

---

#### Withdrawal Function:

1. `withdraw` - Withdraw the ETH balance to the caller.

Rules:

1. This function must transfer the entire ETH balance to the caller.
2. This function must only be callable by the timelock (treasury).

---

#### Receive Function

1. `receive` (keyword) - Enable the governance contract to receive ETH.

Rules:

1. The `receive` function must be empty.

### Implementation

Not Started.
