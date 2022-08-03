# Proposal Payment in Stablecoins

## Simple Summary

Allows the DAO to execute proposals that send funds denominated in USD. This change addresses the ETH/USD price volatility risk both sides suffer from today, as well as the DAO's ability to perform ETH-to-USD swaps trustlessly.

## Abstract

This specification introduces two new contracts that facilitate stablecoin payments:

- `NOU` a new ERC20 token the DAO can mint to say "I owe you this amount of USD"
- `NounsStablecoinPayments` a contract that:
  - allows the DAO to mint NOU tokens as payment to proposal recipients
  - allows anyone to sell it stablecoins in exchange for ETH with some additional incentive
  - allows proposal recipients to redeem NOU for stablecoins

## Technical Specification

### Overview

#### `NOU` contract

A new ERC20 token contract:

- Inherits from OpenZeppelin's `AccessControlEnumerable`
  - Assigns the `ADMIN_ROLE` to an `admin` provided in the constructor; admin can administer its own role and all other roles
- `function mint(address recipient, uint amount)`
  - Mints `amount` tokens to `recipient`
  - Permits only senders of role `MINTER_ROLE`
- `function burn(address from, uint amount)`
  - Burns `amount` tokens from `from`'s account
  - Permits only senders of role `BURNER_ROLE`

#### `NounsStablecoinPayments` contract

A new contract that helps the DAO acquire stablecoins by buying them from bots for a margin, and helps proposal recipients redeem stablecoin payments.

##### Configuration variables

- `Stablecoin` the address of the ERC20 token to use as USD, e.g. USDC or DAI
- `BotIncentiveBPs` the basis points to add on top of the oracle price to further incentivize bots to sell USD to this contract
- `Oracle` the Chainlink Stablecoin/ETH oracle address
- `NOU` the address of the NOU token ERC20 contract
- `Admin` the address of the admin that can withdraw this contract's balances
- `FundingRate` the value multiplier that determines how much ETH funding is required to mint NOU tokens; e.g. if set to 2.5, and the DAO wants to mint 100 NOU, it would need to send ETH worth 250 USD

##### View functions

- `function stablecoinAmountNeeded()` returns the amount of USD the DAO needs to buy, should be the same as `NOU.totalSupply() - Stablecoin.balanceOf(this)`
- `function price(uint usdAmount)` returns the amount of ETH this contract is willing to pay for the given `usdAmount`, taking into account oracle pricing and the additional bot incentive factor
- `function stablecoinBalance()` returns the contract's balance in the desired stablecoin

##### `sellStablecoin` transaction

- `function sellStablecoin(uint amount)` sell this contract stablecoins in exchange for ETH at the exchange rate determined by the `price` function

Rules

- `amount` cannot exceed `stablecoinAmountNeeded()`
- `msg.value == price(amount)`
- `msg.sender` must approve this contract to spend `amount` of stablecoin, which it will transfer to itself

##### `redeem` transactions

- `function redeem(uint amount)` redeem `amount` stablecoins in exchange for the same amount of NOU tokens you own
- `function redeem()` same as above, using `msg.sender`'s NOU balance as the amount

Rules

- `NOU.balanceOf(msg.sender) > 0`
- `amount` cannot exceed `NOU.balanceOf(msg.sender)`
- `amount` cannot exceed `Stablecoin.balanceOf(this)`

##### `mintNOU` transaction

- `function mintNOU(uint amount) payable` mints `amount` new NOU tokens, and accepts ETH payments, allowing `NounsDAOExecutor` to provide ETH funding for buying `amount` stablecoins later

Rules:

- `msg.sender == admin`
- `msg.value >= amount * FundingRate`

##### Admin transactions

- All configuration variables are set in the constructor
- `function withdrawETH()`
- `function withdrawStablecoin()`

Rules:

- `msg.sender == admin`

### Implementation

Hasn't started yet.
