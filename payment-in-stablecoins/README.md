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

#### Interaction flow

1. A proposal is created, where the DAO wants to pay the builder $100K:
   1. Assuming ETH/USD is $1500, and a volatility buffer of 2x
   2. The proposal transaction would be:
      `NounsStablecoinPayments.mintIOUs(recipient, 100_000)`
      where `msg.value = NounsStablecoinPayments.ethNeeded(100_000, 2)`
2. Upon proposal execution:
   1. Proposal recipient is minted 100K NOU tokens
   2. 133.33 ETH is sent from the treasury to NounsStablecoinPayments
3. Arbitrageurs sell USD to NounsStablecoinPayments in exchange for ETH:
   1. They call `NounsStablecoinPayments.sellStablecoin(100_000)` (or smaller amounts across multiple txs)
      1. Expected to happen when `NounsStablecoinPayments.price()` is better than other DEX prices
   2. NounsStablecoinPayments's USD balance grows by $100K, and an appropriate ETH balance is spent
4. Proposal recipient redeems USD in exchange for NOU tokens:
   1. They call `NounsStablecoinPayments.redeem()`
   2. 100K NOU tokens are burned, and 100K USD is transferred from NounsStablecoinPayments to recipient

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

##### Contract state variables

Immutable:

- `iouToken` the address of the NOU token ERC20 contract
- `stablecoin` the address of the ERC20 token to use as USD, e.g. USDC or DAI
- `oracle` the Chainlink Stablecoin/ETH oracle address

Mutable:

- `baselineStablecoinAmount` how much stablecoin should the contract buy on top of outstanding IOU, to serve as a buffer than can expedite recipient redemption
- `botIncentiveBPs` the basis points to add on top of the oracle price to further incentivize bots to sell USD to this contract
- `admin` the address of the admin that can withdraw this contract's balances

##### View functions

- `function stablecoinAmountNeeded()` returns the amount of USD the DAO needs to buy, should be the same as `iouToken.totalSupply() - stablecoin.balanceOf(this) + baselineStablecoinAmount`
- `function price()` returns the amount stablecoin this contract is asking for in exchange for one ETH, taking into account oracle pricing and the additional bot incentive factor: `Chainlink(eth-usd.data.eth).latestRoundData().answer * (10_000 + botIncentiveBPs) / 10_000` (the chainlink address above is just for example purposes)
- `function stablecoinBalance()` returns the contract's balance in the desired stablecoin
- `function ethNeeded(uint additionalUsdAmount, uint bufferFactor)` returns the amount of additional ETH needed in the contract to buy the USD backing needed for additionalUsdAmount + any unbacked minted NOU; `bufferFactor` is a volatility buffer scalar, e.g. when set to 2 the contract will ask for ETH with 2 times the USD value it needs to buy
  - `bufferFactor` should have a hard-coded lower bound, e.g. 1.2 for 20% over-funding

##### `sellStablecoin` transaction

- `function sellStablecoin(uint amount)` sell this contract stablecoins in exchange for ETH at the exchange rate determined by the `price` function

Rules

- `amount` cannot exceed `stablecoinAmountNeeded()`
- `msg.sender` must approve this contract to spend `amount` of stablecoin, which it will transfer to itself
- should revert if `this.balance < amount / price()`

##### `redeem` transactions

Non-strict:

- `function redeem(uint amount)` redeem `amount` stablecoins in exchange for the same amount of NOU tokens you own
- `function redeem()` same as above, using `msg.sender`'s NOU balance as the amount

Strict:

- `function redeemExact(uint amount)` redeem `amount` stablecoins in exchange for the same amount of NOU tokens you own
- `function redeemExact()` same as above, using `msg.sender`'s NOU balance as the amount

Rules

- `iouToken.balanceOf(msg.sender) > 0` must be true
- `amount` cannot exceed `iouToken.balanceOf(msg.sender)`
- must call `iouToken.burn` with the amount of stablecoins sent to `msg.sender`
- `redeemExact` functions should revert if `amount > stablecoin.balanceOf(this)`

##### `mintIOUs` transaction

- `function mintIOUs(address to, uint amount) payable` mints `amount` new `iouToken`s to `to`, and accepts ETH payments, allowing `NounsDAOExecutor` to provide ETH funding for buying `amount` stablecoins later

Rules:

- `msg.sender == admin`
- `msg.value >= amount * FundingRate`

##### Admin transactions

- `function withdrawETH()`
- `function withdrawStablecoin()`
- `function setBotIncentiveBPs(uint newBotIncentiveBPs)`
- `function setAdmin(address newAdmin)`
- `function setbBaselineStablecoinAmount(uint newAmount)`

##### `receive` transaction

- This contract accepts ETH, allowing the DAO to top off its ETH balance without having to mint iou tokens

Rules:

- `msg.sender == admin`

### Implementation

Hasn't started yet.
