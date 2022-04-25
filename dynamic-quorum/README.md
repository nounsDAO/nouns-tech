# Dynamic Governance Quorum

## Simple Summary

A dynamic vote quorum that adjusts as a function of for/against votes, allowing uncontested proposals to pass at a minimum defined quorum, while requiring higher assurance on contested proposals.

## Abstract

This specification defines an upgrade to the `NounsDAOLogicV1` contract that replaces the static `proposal.quorumVotes` variable with a function that adjusts the vote quorum between a defined minimum and maximum based on the amount of voter opposition.

## Technical Specification

### Overview

The dynamic quorum must be implemented in a new Nouns DAO governance logic contract (`NounsDAOLogicV2`).

Additions to V1:

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
10. `minQuorumVotes` (`Proposal` Struct Member) - The minimum number of votes in support of a proposal required in order for a quorum to be reached and for a vote to succeed at the time of proposal creation. Note: This is a rename of an existing struct member. See `Modifications to V1`.
11. `mapping(uint256 => StateSnapshot) public snapshot` (Public Mapping) - A small state snapshot that's taken at the time of proposal creation.
    Rules:
      - `StateSnapshot` struct:
        ```solidity
        struct StateSnapshot {
            /// @notice Total Noun supply at the time of proposal creation
            uint256 totalSupply;
            /// @notice Minimum quorum votes BPS at the time of proposal creation
            uint256 minQuorumVotesBPS;
            /// @notice Maximum quorum votes BPS at the time of proposal creation
            uint256 maxQuorumVotesBPS;
        }
        ```
      - Value is set in `propose` function. `uint256` key is the `proposalCount`.
12. `quorumVotes` (Public View Function) - Quorum votes required for a specific proposal to succeed.
    Rules:
      - Given a `proposalId`, this function must return the quorum votes required for the proposal to succeed.
      - If `snapshot.totalSupply` equals `0`, `proposal.minQuorumVotes` must be returned to maintain backwards compatibility.
      - Dynamic quorum reference code in it's simplest, linear, unparameterized form:
        ```js
        const consensus = 1 - (for_votes / (for_votes + against_votes));
        const max_adjustment = max_quorum_votes_bps - min_quorum_votes_bps;
        const quorumBPS = (max_adjustment * consensus) + min_quorum_votes_bps;
        const quorum = (quorumBPS * total_supply) / 10_000;
        ```

Modifications to V1:

1. `quorumVotesBPS` (Public State Variable) should be renamed to `minQuorumVotesBPS`.
2. `quorumVotes` (`Proposal` Struct Member ) should be renamed to `minQuorumVotes`.
3. `quorumVotes` (Public View Function) signature should be updated to `quorumVotes(uint256 proposalId)`.

### Implementation

Not Started.
