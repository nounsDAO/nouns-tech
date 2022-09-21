# Proposal Payment in Stablecoins

## Simple Summary

Allows the DAO to execute proposals that send funds denominated in USD. This change addresses the ETH/USD price volatility risk both sides suffer from today, as well as the DAO's ability to perform ETH-to-USD swaps trustlessly.

### Simple User flow

1. Proposal creation: new UI will allow proposer to choose to pay out in USD and specify the amount, e.g. 50K USD
2. Proposal execution: builder receives the desired USD amount
   1. If the DAO doesn't have sufficient USD, a debt is registered on chain, and shortly after a trading bot would sell the DAO the outstanding USD and immediately close the builder's balance

### Detailed User Flow

TokenBuyer is open to trading whenever:

1. its USD balance is below the desired position, or
2. the DAO has a debt towards a builder

When TokenBuyer is open to trading, it offers trading bots the opportunity to sell their stablecoins (e.g. USDC) in exchange for ETH. Bots are incentivized by offering a price premium, providing them an arbitrage opportunity.

Since acquiring USD from bots can take time, we expect the proposal page to clearly show proposal builders their outstanding balance and explain the balance should be settled shortly.

TokenBuyer's USD position (baseline amount) configuration value helps mitigate debt delays: the higher the position, the less proposals will ever have to enter debt state.

## Abstract

This specification introduces two new contracts that facilitate stablecoin payments:

1. `TokenBuyer`, a contract that:

   1. allows anyone to sell it stablecoins in exchange for ETH with some additional incentive
   2. funds the `Payer` contract with said stablecoins
   3. auto-pays `Payer` debts upon receiving stablecoins

2. `Payer`, a contract that:
   1. allows the DAO to pay builders in stablecoins (e.g. USDC)
   2. registers debt when the DAO's obligations exceed its stabecoin balance
   3. allows anyone to run its outstanding debt payment function

## Technical Specification

### Overview

#### Interaction flow

1. A proposal is created, where the DAO wants to pay the builder $100K, with the following transactions
   1. `Payer.sendOrRegisterDebt(recipient, 100_000)`
   2. send TokenBuyer ETH at the amount of `TokenBuyer.ethNeeded(100_000, 2)` (assuming we want a 2x volatility buffer)
2. Upon proposal execution:
   1. Proposal recipient receives 100K stablecoins (e.g. USDC)
   2. 133.33 ETH is sent from the treasury to `TokenBuyer` (assuming ETH/USD is $1500 and 2x buffer above)
3. Arbitrageurs sell USD to `TokenBuyer` in exchange for ETH:
   1. They call `TokenBuyer.buyETH(100_000)` (or smaller amounts across multiple txs)
      1. Expected to happen when `TokenBuyer.price()` is better than other DEX prices
   2. `Payer`'s USD balance grows by $100K, and an appropriate ETH balance is spent

### `TokenBuyer` contract

A new contract that helps the DAO acquire stablecoins by buying them from bots for a margin.

#### Contract state variables

Immutable:

- `paymentToken` the ERC20 token to use as USD, e.g. USDC
- `paymentTokenDecimalsDigits` as `10^paymentToken.decimals()`, used in conversions between token decimals and ETH decimals (18)

Mutable:

- `admin` contract admin, allowed to do certain lower risk operations
- `payer` the `Payer` contract it helps fund with stablecoins
- `priceFeed` the address of the `IPriceFeed` contract used to fetch stablecoin/ETH prices
- `baselineStablecoinAmount` The minimum `paymentToken` balance the `payer` contract should have
- `minAdminBaselinePaymentTokenAmount` The minimum value for `baselinePaymentTokenAmount` that `admin` is allowed to set
- `maxAdminBaselinePaymentTokenAmount` The maximum value for `baselinePaymentTokenAmount` that `admin` is allowed to set
- `botDiscountBPs` the amount of basis points to decrease the price by, to increase the incentive to transact with this contract
- `minAdminBotDiscountBPs` The minimum value for `botDiscountBPs` that `admin` is allowed to set
- `maxAdminBotDiscountBPs` The maximum value for `botDiscountBPs` that `admin` is allowed to set

#### View functions

`function ethNeeded(uint256 additionalTokens, uint256 bufferBPs) public view returns (uint256)`

- Get how much ETH this contract needs in order to fund its current obligations plus `additionalTokens`, with a safety buffer `bufferBPs` basis points

`function tokenAmountNeeded() public view returns (uint256)`

- Returns the amount of tokens this contract is willing to exchange of ETH
  - zero if it has enough tokens
  - otherwise `baselinePaymentTokenAmount + payer.totalDebt() - paymentToken.balanceOf(address(payer))`

`function price() public view returns (uint256)`

- Returns the ETH/`paymentToken` price this contract is willing to exchange ETH at, including the discount

`function ethAmountPerTokenAmount(uint256 tokenAmount) public view returns (uint256)`

- Returns the amount of ETH this contract will send in exchange for `tokenAmount` tokens

`function tokenAmountNeededAndETHPayout() public view returns (uint256, uint256)`

- Returns the amount of tokens the contract can buy and the amount of ETH it will pay for it

`function tokenAmountPerEthAmount(uint256 ethAmount) public view returns (uint256)`

- Returns the amount of tokens the contract expects in return for eth

#### `buyETH` transaction

##### Description

- Buy ETH from this contract in exchange for `paymentToken` tokens
- The price is determined using `priceFeed` plus `botDiscountBPs`
- Immediately invokes `payer` to pay back outstanding debt

##### Rules

`function buyETH(uint256 tokenAmount)`

- `msg.sender` must approve this contract to spend `amount` of stablecoin, which it will transfer to `Payer`
- should revert if contract has insufficient ETH balance

`function buyETH(uint256 tokenAmount, address to, bytes calldata data)`

- `to` must implement the interface `IBuyETHCallback.buyETHCallback`
- as part of `buyETHCallback`, sender must transfer `amount` stablecoins to `Payer`
- should revert if contract has insufficient ETH balance

#### Admin / Owner transactions

##### General Rules

- `msg.sender` must be `admin` or `owner`

##### Specific Rules

`function setBotDiscountBPs(uint16 newBotDiscountBPs)`

- if `msg.sender == admin`, the new value must be within the min/max bounds set by `owner`

`function setBaselinePaymentTokenAmount(uint256 newBaselinePaymentTokenAmount)`

- if `msg.sender == admin`, the new value must be within the min/max bounds set by `owner`

`function withdrawETH()`

- the contract's ETH balance must be sent to `owner`
- should revert if the transfer fails

`function pause()`
`function unpause()`
`function setAdmin(address newAdmin)`

- no additional rules

#### Owner-only transactions

##### Rules

- `msg.sender` must be `owner`

`function setMinAdminBotDiscountBPs(uint16 newMinAdminBotDiscountBPs)`
`function setMaxAdminBotDiscountBPs(uint16 newMaxAdminBotDiscountBPs)`
`function setMinAdminBaselinePaymentTokenAmount(uint256 newMinAdminBaselinePaymentTokenAmount)`
`function setMaxAdminBaselinePaymentTokenAmount(uint256 newMaxAdminBaselinePaymentTokenAmount)`
`function setPriceFeed(IPriceFeed newPriceFeed)`
`function setPayer(address newPayer)`

No function-specific rules.

#### `receive` transaction

- This contract accepts ETH, allowing the DAO to fund further stablecoin purchasing as needed
- should not revert

### `Payer` contract

This contract is used to pay recipients from a balance of ERC20 tokens, supporting a state of outstanding debt to recipients.

#### Contract state variables

Immutable:

- `paymentToken` The ERC20 token used to pay users

Mutable:

- `totalDebt` The total debt owed, in `paymentToken` tokens
- `queue` A queue of debt entries waiting to be paid

#### `sendOrRegisterDebt` transaction

`function sendOrRegisterDebt(address account, uint256 amount)`

##### Description

- Pays `account` an `amount` of `paymentToken`s
- Adds a debt entry if there's not enough tokens

##### Rules

- msg.sender must be `owner`

#### `payBackDebt` transaction

`function payBackDebt(uint256 amount)`

##### Description

- Pays back debt up to `amount` of `paymentToken`
- Debt is paid in FIFO order

##### Rules

- should only attempt to pay up to the contract's `paymentToken` balance

### `IPriceFeed` interface

The interface `TokenBuyer` uses to fetch ETH/stablecoin prices.

- `function price()` returns the latest price

### `PriceFeed` contract

`PriceFeed` is the first implementation of `IPriceFeed`. It uses Chainlink as its only oracle, to avoid the extra gas costs of using multiple oracles, under the assumption that Chainlink is highly available.

#### Contract state variables

Immutable:

- `chainlink` Chainlink price feed
- `decimals` Number of decimals of the chainlink price feed answer
- `decimalFactor` A factor to multiply or divide by to get to 18 decimals
- `staleAfter` Max staleness allowed from chainlink, in seconds
- `priceLowerBound` Sanity check: minimal price allowed
- `priceUpperBound` Sanity check: maximal price allowed

#### `price` view function

##### Description

- Returns the price of ETH/Token by fetching from Chainlink

##### Rules

- should revert if Chainlink's lastest update is older than `staleAfter` seconds ago
- should revert if Chainlink's price is outside the min/max sanity bounds

### Implementation

[Token Buyer repo](https://github.com/nounsDAO/token-buyer/).
