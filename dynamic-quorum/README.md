# Dynamic Governance Quorum

## Simple Summary

A dynamic vote quorum that adjusts as a result of opposition, allowing uncontested proposals to pass at a minimum defined quorum, while requiring higher assurance on contested proposals.

## Abstract

This specification defines an upgrade to the `NounsDAOLogicV1` contract that replaces the static `proposal.quorumVotes` variable with a function that adjusts the vote quorum between a defined minimum and maximum as a result of voter opposition.

## Technical Specification

### Overview

The dynamic quorum must be implemented as an upgrade to the Nouns DAO governance logic contract (`NounsDAOLogicV2`).

- The following configuration values must be implemented as storage variables:
  - Values:
    - **Minimum Quorum Votes Basis Points** - The minimum basis point number of votes in support of a proposal required in order for a quorum to be reached and for a vote to succeed.
    - **Maximum Quorum Votes Basis Points** - The maximum basis point number of votes in support of a proposal required in order for a quorum to be reached and for a vote to succeed.
    - **Dynamic Quorum Linear Coefficient** - The coefficient used to control the dynamic quorum slope.
  - Rules:
    - These values must be **checkpointed** by block number upon update. A publicly exposed function must allow for historical configuration values to be read.
    - Minimum quorum votes BPS must be guarded by constant lower and upper bounds.
    - Maximum quorum votes BPS must be greater than or equal to the minimum and be guarded by a constant upper bound.
- The `totalSupply` at the time of proposal creation must be stored for use by the dynamic quorum function.
- The `state` function, which determines proposal outcomes, must consume the `quorumVotes` public view function. Previously, the `state` function consumed the static `proposal.quorumVotes` value that was assigned during proposal creation.
  - `quorumVotes` Rules:
    - Given a proposal ID, the `quorumVotes` function must return the number of votes required for the proposal to succeed.
    - `quorumVotes` must return the minimum quorum if the proposal was created prior to the deployment of `NounsDAOLogicV2`.
    - `quorumVotes` must read the checkpointed configuration values from the proposal creation block.
    - The dynamic quorum function is as follows (pseudocode):
      ```js
      /*
        DAO-Controlled Configuration Values:

        min_quorum_bps - Minimum quorum in basis points
        max_quorum_bps - Maximum quorum in basis points
        linear_coefficient - Adjust the slope of the dynamic quorum

        Dynamic Quorum Calculation:

        quorum = min((min_quorum_bps + bx) / 10_000, max_quorum)

        Reference:
      */

      function get_percent(bps, num) {
        return (bps * num) / 10_000
      }
      function get_quorum_adjustment_bps(against_votes_adjusted_bps) {
        return linear_coefficient * against_votes_adjusted_bps
      }
      function max_quorum() {
        return get_percent(max_quorum_bps, total_supply)
      }

      quorum_adjustment_bps = get_quorum_adjustment_bps(against_votes_bps)
      adjusted_quorum = get_percent(min_quorum_bps + quorum_adjustment_bps, total_supply)
      quorum = min(adjusted_quorum, max_quorum)
      ```
- `_setQuorumVotesBPS` must be removed as it no longer serves a purpose.
-  `minQuorumVotes` must be exposed as an external view function, which returns the current minimum quorum votes using Noun total supply.
-  `maxQuorumVotes` must be exposed as an external view function, which returns the current maximum quorum votes using Noun total supply.

A dynamic quorum simulation can be found [here](https://docs.google.com/spreadsheets/d/1trmtpgSLx8cfm-wD79mQWyh2y_GwW3X7f2rEnSWJ3AM/edit?usp=sharing), courtesy of the Verbs team.

### Implementation

In Progress.
