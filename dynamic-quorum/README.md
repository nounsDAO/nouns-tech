# Dynamic Governance Quorum

## Simple Summary

A dynamic vote quorum that adjusts as a function of for/against votes, allowing uncontested proposals to pass at a minimum defined quorum, while requiring higher assurance on contested proposals.

## Abstract

This specification defines an upgrade to the `NounsDAOLogicV1` contract that replaces the static `proposal.quorumVotes` variable with a function that adjusts the vote quorum between a defined minimum and maximum based on the amount of voter opposition.

## Technical Specification

