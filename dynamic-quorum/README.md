# Dynamic Governance Quorum

## Simple Summary

A dynamic vote quorum that adjusts as a function of against votes, allowing uncontested proposals to pass at a minimum defined quorum, while requiring higher assurance on contested proposals.

## Abstract

This specification defines an upgrade to the `NounsDAOLogicV1` contract that replaces the static `proposal.quorumVotes` variable with a function that adjusts the vote quorum between a defined minimum and maximum based on the amount of voter opposition.

## Technical Specification

### Overview

The dynamic quorum must be implemented in a new Nouns DAO governance logic contract (`NounsDAOLogicV2`).

#### Additions to V1:

1. `MIN_QUORUM_VOTES_BPS_LOWER_BOUND` (Public Constant) - The lower bound of minimum quorum votes basis points.
2. `MIN_QUORUM_VOTES_BPS_UPPER_BOUND` (Public Constant) - The upper bound of minimum quorum votes basis points.
3. `MAX_QUORUM_VOTES_BPS_UPPER_BOUND` (Public Constant) - The upper bound of maximum quorum votes basis points.
4. `minQuorumVotesBPS` (Public State Variable) - The minimum basis point number of votes in support of a proposal required in order for a quorum to be reached and for a vote to succeed.
5. `maxQuorumVotesBPS` (Public State Variable) - The maximum basis point number of votes in support of a proposal required in order for a quorum to be reached and for a vote to succeed.
6. `_setMinQuorumVotesBPS` (External Function) - Admin function for setting the minimum quorum votes basis points.
    - Rules:
      - Function can only be called by `admin`.
      - Value must be greater than or equal to constant `MIN_QUORUM_VOTES_BPS_LOWER_BOUND`.
      - Value must be less than or equal to constant `MIN_QUORUM_VOTES_BPS_UPPER_BOUND`.
      - Sets `minQuorumVotesBPS` to provided value.
      - Emits `event MinQuorumVotesBPSSet(uint256 oldMinQuorumVotesBPS, uint256 newMinQuorumVotesBPS)` event on success.
7. `_setMaxQuorumVotesBPS` (External Function) - Admin function for setting the maximum quorum votes basis points.
    - Rules:
      - Function can only be called by `admin`.
      - Value must be greater than or equal to state variable `minQuorumVotesBPS`.
      - Value must be less than or equal to constant `MAX_QUORUM_VOTES_BPS_UPPER_BOUND`.
      - Sets `maxQuorumVotesBPS` to provided value.
      - Emits `event MaxQuorumVotesBPSSet(uint256 oldMaxQuorumVotesBPS, uint256 newMaxQuorumVotesBPS)` event on success.
8. `minQuorumVotes` (External View Function) - Current min quorum votes using Noun total supply.
9. `maxQuorumVotes` (External View Function) - Current max quorum votes using Noun total supply.
10. `minQuorumVotes` (`Proposal` Struct Member) - The minimum number of votes in support of a proposal required in order for a quorum to be reached and for a vote to succeed at the time of proposal creation. Note: This is a rename of an existing struct member. See [Modifications to V1](#modifications-to-v1).
11. `totalSupply` (`Proposal` Struct Member) - The total supply of Nouns at the time of proposal creation.
12. `quorumVotes` (Public View Function) - Quorum votes required for a specific proposal to succeed.
    Rules:
      - Given a `proposalId`, this function must return the quorum votes required for the proposal to succeed.
      - If `snapshot.totalSupply` equals `0`, `proposal.minQuorumVotes` must be returned to maintain backwards compatibility.
      - Dynamic quorum function is as follows (pseudocode):
        ```js
        /*
          DAO-Controlled Configuration Values:

          slope_coefficient - Adjust the slope of the dynamic quorum
          curvature_coefficient - Adjust the rate of change in the slope of the dynamic quorum
          offset_bps - Adjust the point at which the quorum adjustment activates
        */

        // Dynamic Quorum Calculation:
        function quorum_adjustment_bps(against_votes_adjusted_bps) {
          return (curvature_coefficient * against_votes_adjusted_bps) + (slope_coefficient * against_votes_adjusted_bps)
        }

        adjusted_against_votes_bps = against_votes_bps > offset_bps ? against_votes_bps - offset_bps : 0
        quorum_adjustment = (quorum_adjustment_bps(adjusted_against_votes_bps) * total_supply) / 10_000
        quorum = min(min_quorum(total_supply) + quorum_adjustment, max_quorum(total_supply))
        ```
13. All dynamic quorum configuration values must be checkpointed by block number. This will allow `quorumVotes` to access historical values from the proposal creation block.

#### Modifications to V1:

1. `quorumVotesBPS` (Public State Variable) should be renamed to `minQuorumVotesBPS`.
2. `quorumVotes` (`Proposal` Struct Member ) should be renamed to `minQuorumVotes`.
3. `quorumVotes` (Public View Function) signature should be updated to `quorumVotes(uint256 proposalId)`.

### Implementation

In Progress.
