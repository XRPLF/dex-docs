# Index

- [1. Introduction](#1-introduction)
- [2. Deposit Logic](#2-deposit-logic)
  - [2.1. Pseudo-Code](#21-pseudo-code)
- [3. getTradingFee](#3-gettradingfee)
  - [3.1. getTradingFee Pseudo-Code](#31-gettradingfee-pseudo-code)
- [4. Multi-Asset Deposit Modes](#4-multi-asset-deposit-modes)
  - [4.1. equalDepositLimit (tfTwoAsset)](#41-equaldepositlimit-tftwoasset)
    - [4.1.1. equalDepositLimit Pseudo-Code](#411-equaldepositlimit-pseudo-code)
  - [4.2. equalDepositTokens (tfLPToken)](#42-equaldeposittokens-tflptoken)
    - [4.2.1. equalDepositTokens Pseudo-Code](#421-equaldeposittokens-pseudo-code)
  - [4.3. equalDepositInEmptyState (tfTwoAssetIfEmpty)](#43-equaldepositinemptystate-tftwoassetifempty)
- [5. Single-Asset Deposit Modes](#5-single-asset-deposit-modes)
  - [5.1. singleDeposit (tfSingleAsset)](#51-singledeposit-tfsingleasset)
    - [5.1.1. singleDeposit Pseudo-Code](#511-singledeposit-pseudo-code)
  - [5.2. singleDepositTokens (tfOneAssetLPToken)](#52-singledeposittokens-tfoneassetlptoken)
    - [5.2.1. singleDepositTokens Pseudo-Code](#521-singledeposittokens-pseudo-code)
  - [5.3. singleDepositEPrice (tfLimitLPToken)](#53-singledepositeprice-tflimitlptoken)
    - [5.3.1. singleDepositEPrice Pseudo-Code](#531-singledepositeprice-pseudo-code)
- [6. Common Deposit Function](#6-common-deposit-function)
  - [6.1. deposit Pseudo-Code](#61-deposit-pseudo-code)

# 1. Introduction

The AMMDeposit transaction allows liquidity providers to add assets to an AMM pool in exchange for LP tokens. This document provides technical implementation details for the deposit logic in `xrpld`.

The [AMM documentation](README.md#32-ammdeposit-transaction) describes the high-level business logic of deposits, including the six different [deposit modes](README.md#321-deposit-modes), their purposes, and user-facing behavior. This document focuses on the implementation: how the code validates constraints, calculates token amounts, and executes asset transfers.

**Terminology Notes**:

Throughout the document we refer to equations by their number in the [XLS-30 specification](https://github.com/XRPLF/XRPL-Standards/tree/master/XLS-0030-automated-market-maker).
AMM helpers use **token** as a subject in many function names. This refers to any supported asset in the system, not only a particular token implementation, like trust lines or MPTs.

# 2. Deposit Logic

A deposit[^apply-guts] reads the pool's balances and picks the trading fee that applies to the depositor, which is lower for [auction slot holders](#3-gettradingfee). The transaction flags then select a deposit mode. The [multi-asset modes](#4-multi-asset-deposit-modes) keep the pool's asset ratio or refill an empty pool. The [single-asset modes](#5-single-asset-deposit-modes) add one asset only. Each mode calculates the amounts and LP tokens, then calls the [common deposit function](#6-common-deposit-function) to move the assets and issue the LP tokens.

Under the `fixCleanup3_3_0` amendment, `preclaim` runs the freeze checks for both pool assets whether or not they are deposited. See the [failure conditions](README.md#322-failure-conditions) for the resulting behavior change.

Under the `fixCleanup3_4_0` amendment, the mode dispatch is wrapped in a guard that catches an out-of-range solved amount.[^tflptoken-guard] See the [failure conditions](README.md#322-failure-conditions) for the resulting behavior change.

[^tflptoken-guard]: In 3.4.0, the tfLPToken mode (`equalDepositTokens`) catches these errors itself and returns `tecINTERNAL`, so the guard never sees them: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.4.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L745-L751)

[^apply-guts]: `AMMDeposit::applyGuts`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.4.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L412-L548) 

## 2.1. Pseudo-Code

```python
def deposit_into_pool(pool, depositor, tx):
    # The AMM's reserves, in the order of Amount and Amount2. Frozen or locked
    # reserves, and MPTs the AMM may not hold, read as zero.
    balance1, balance2, lp_supply = pool.balances(tx.Amount, tx.Amount2)

    if lp_supply == 0:
        fee = tx.TradingFee or 0              # an empty pool takes a new fee, as in AMMCreate
    else:
        fee = getTradingFee(pool, depositor)  # discounted for an active slot's owner and authorized accounts

    deposit_mode = {
        tfTwoAsset: equalDepositLimit,
        tfLPToken: equalDepositTokens,
        tfTwoAssetIfEmpty: equalDepositInEmptyState,
        tfSingleAsset: singleDeposit,
        tfOneAssetLPToken: singleDepositTokens,
        tfLimitLPToken: singleDepositEPrice,
    }[tx.deposit_mode_flag]

    try:
        new_lp_supply = deposit_mode(pool, balance1, balance2, lp_supply, tx, fee)
    except AmountOutOfRange:
        if fixCleanup3_4_0:
            fail(tecAMM_FAILED)
        fail(tefEXCEPTION)

    pool.lp_token_supply = new_lp_supply
    if lp_supply == 0:
        # As in AMMCreate: the depositor gets the auction slot for free and the
        # only fee vote, and the pool takes the new trading fee
        pool.initialize_fee_auction_vote(owner=depositor, fee=fee)
```

# 3. getTradingFee

The `getTradingFee` function[^get-trading-fee] is used by the [deposit logic](#2-deposit-logic) to pick the depositor's trading fee. An [auction slot](README.md#121-auction-slot) holder pays a tenth of the regular fee, rounded down. That discount is what they paid for with their [bid](bidding.md). The discount also goes to the accounts the owner authorized. It lasts until the slot expires. Expiry is checked against the close time of the previous ledger, not the wall clock.

[^get-trading-fee]: `getTradingFee`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L602-L627)  

## 3.1. getTradingFee Pseudo-Code

```python
def getTradingFee(pool, account):
    slot = pool.auction_slot
    if now() < slot.expiration and (account == slot.owner or account in slot.auth_accounts):
        return slot.discounted_fee
    return pool.trading_fee
```

# 4. Multi-Asset Deposit Modes

Multi-asset deposit modes maintain proportional deposits by adding both pool assets simultaneously. This preserves the pool's asset ratio while increasing liquidity. Since these deposits maintain proportional ratios, they incur no trading fees.

The three modes are:
- [`equalDepositLimit`](#41-equaldepositlimit-tftwoasset) - Specify maximum amounts for both assets, receive proportional LP tokens
- [`equalDepositTokens`](#42-equaldeposittokens-tflptoken) - Specify LP tokens to receive, system calculates required asset amounts
- [`equalDepositInEmptyState`](#43-equaldepositinemptystate-tftwoassetifempty) - Reinitialize an empty pool with new asset amounts and establish a new price ratio

## 4.1. equalDepositLimit (tfTwoAsset)

> "I'll deposit up to X and Y, how many LP tokens do I get?"

The `equalDepositLimit` function[^equal-deposit-limit] handles proportional deposits where the depositor specifies maximum amounts they're willing to provide for both assets (`Amount` and `Amount2`). Since the deposit must maintain the pool's ratio, the function cannot simply use both maximum amounts - one will typically be limiting while the other has excess.

[^equal-deposit-limit]: `AMMDeposit::equalDepositLimit`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L710-L780)

The function tries two strategies to maximize the deposit within the user's constraints. First, it attempts to use all of `Amount` by calculating the pool fraction this represents (`frac = Amount / amountBalance`), converting this to LP tokens with proper rounding, then recalculating the fraction from the rounded LP tokens (`frac = tokensAdj / lptAMMBalance`) to ensure precision consistency. Using this adjusted fraction, it calculates the proportional amount2 needed. If this amount2 fits within `Amount2`, the deposit proceeds immediately. 

Only if the first strategy fails does it try the second strategy: starting with all of `Amount2`, going through the same fraction -> LP tokens -> adjusted fraction -> amount1 calculation. If the calculated amount1 exceeds `Amount`, the entire deposit fails. The optional minimum LP token constraint (`LPTokenOut`) is validated by the [deposit function](#6-common-deposit-function).

**Example:**

Alice has an AMM with 100 USD and 100 EUR (1:1 ratio). Bob wants to make a proportional deposit.

- **Case 1: Bob tries to deposit 100 USD + 50 EUR**
  - Strategy 1: Use all 100 USD and needs 100 EUR to maintain 1:1 ratio - FAILS (only has 50 EUR)
  - Strategy 2: Use all 50 EUR and needs 50 USD to maintain 1:1 ratio - SUCCESS (has 100 USD)
  - Result: Deposits 50 USD + 50 EUR

- **Case 2: Bob tries to deposit 50 USD + 100 EUR**
  - Strategy 1: Use all 50 USD and needs 50 EUR to maintain 1:1 ratio - SUCCESS (has 100 EUR)
  - Result: Deposits 50 USD + 50 EUR, receives 50 LP tokens, 50 EUR unused

### 4.1.1. equalDepositLimit Pseudo-Code

```python
def equalDepositLimit(pool, balance1, balance2, lp_supply, tx, fee):
    def spend_all_of(amount, balance, other_balance):
        share = amount / balance
        lp_tokens = getRoundedLPTokens(lp_supply, share, IsDeposit=True)
        if lp_tokens == 0:
            fail(tecAMM_INVALID_TOKENS)
        share = adjustFracByTokens(lp_supply, lp_tokens, share)  # the share the rounded LP tokens buy
        return lp_tokens, getRoundedAsset(other_balance, share, IsDeposit=True)

    lp_tokens, amount2 = spend_all_of(tx.Amount, balance1, balance2)
    if amount2 <= tx.Amount2:
        return deposit(pool, tx.Amount, amount2, lp_tokens, min_lp_tokens=tx.LPTokenOut)

    lp_tokens, amount1 = spend_all_of(tx.Amount2, balance2, balance1)
    if amount1 <= tx.Amount:
        return deposit(pool, amount1, tx.Amount2, lp_tokens, min_lp_tokens=tx.LPTokenOut)

    fail(tecAMM_FAILED)
```

## 4.2. equalDepositTokens (tfLPToken)

> "I want X LP tokens, what do I need to deposit?"

Proportional deposit for a requested number of LP tokens.[^equal-deposit-tokens]

[^equal-deposit-tokens]: `AMMDeposit::equalDepositTokens`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L637-L679)

This function handles the reverse calculation from [`equalDepositLimit`](#41-equaldepositlimit-tftwoasset): the user specifies the number of LP tokens they want to receive, and the function calculates the required proportional amounts of both assets. The user provides `LPTokenOut` (LP tokens desired) and optionally `Amount` and `Amount2` as minimum constraints on the deposit amounts. The function first adjusts the requested LP tokens for precision using [`adjustLPTokensOut`](helpers.md#24-adjustlptokensout-deposits), so the user can receive slightly fewer than `LPTokenOut`. It then calculates the pool fraction these tokens represent (`frac = tokensAdj / lptAMMBalance`). Using this fraction, it calculates the required amounts of both assets by multiplying each pool balance by the fraction, rounding up with [`getRoundedAsset`](helpers.md#23-getroundedasset) to ensure the pool receives sufficient assets. If the calculated deposit amounts are less than the optional `Amount` and `Amount2` minimums, the transaction fails with `tecAMM_FAILED`.

**Example:**

Alice created an AMM with 100 USD, 100 EUR, and 100 LP tokens outstanding. Bob wants to receive exactly 25 LP tokens.

- Desired LP tokens: 25
- Fraction of pool: 25 / 100 = 0.25 (25%)
- Required USD: 100 * 0.25 = 25 USD
- Required EUR: 100 * 0.25 = 25 EUR
- Result: Bob deposits 25 USD and 25 EUR, receives 25 LP tokens


### 4.2.1. equalDepositTokens Pseudo-Code

```python
def equalDepositTokens(pool, balance1, balance2, lp_supply, tx, fee):
    lp_tokens = adjustLPTokensOut(lp_supply, tx.LPTokenOut)
    if lp_tokens == 0:
        fail(tecAMM_INVALID_TOKENS)

    share = lp_tokens / lp_supply
    amount1 = getRoundedAsset(balance1, share, IsDeposit=True)
    amount2 = getRoundedAsset(balance2, share, IsDeposit=True)
    return deposit(pool, amount1, amount2, lp_tokens, min_amount1=tx.Amount, min_amount2=tx.Amount2)
```

## 4.3. equalDepositInEmptyState (tfTwoAssetIfEmpty)

Reinitialize an empty AMM pool.[^equal-deposit-in-empty-state]

[^equal-deposit-in-empty-state]: `AMMDeposit::equalDepositInEmptyState`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L994-L1014)

This mode refills an AMM that has no LP tokens left. The withdrawal that removes the last LP tokens normally deletes the AMM. The AMM stays in the ledger only when it has too many trust lines to delete in one transaction (see [AMMDelete](README.md#36-ammdelete-transaction)). Such an AMM holds no assets, because that withdrawal takes out both full balances. The depositor specifies both `Amount` and `Amount2` to set a new pool ratio, similar to [`AMMCreate`](README.md#31-ammcreate-transaction). The initial LP tokens are the geometric mean `sqrt(Amount * Amount2)`, rounded down, as in AMMCreate. The depositor can optionally provide `TradingFee` to set a new fee for the pool. This mode has no minimum fields, because the depositor sets the pool's terms. The [common deposit function](#6-common-deposit-function) then moves both amounts into the pool and issues the LP tokens.

# 5. Single-Asset Deposit Modes

Single-asset deposits allow users to deposit only one asset instead of both assets proportionally. Unlike [multi-asset deposits](#4-multi-asset-deposit-modes) that maintain the pool ratio, single-asset deposits change the pool composition. Because they alter the pool ratio, [trading fees](#3-gettradingfee) apply to single-asset deposits. There are three modes:

- **[singleDeposit](#51-singledeposit-tfsingleasset) (tfSingleAsset)**: User specifies deposit amount, receives calculated LP tokens
- **[singleDepositTokens](#52-singledeposittokens-tfoneassetlptoken) (tfOneAssetLPToken)**: User specifies LP tokens desired, deposits calculated amount
- **[singleDepositEPrice](#53-singledepositeprice-tflimitlptoken) (tfLimitLPToken)**: User specifies deposit amount and effective price limit

## 5.1. singleDeposit (tfSingleAsset)

> "I'll deposit X amount, how many LP tokens will I get?"

Deposit a single asset to receive calculated LP tokens.[^single-deposit]

[^single-deposit]: `AMMDeposit::singleDeposit`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L791-L828)

The user specifies `Amount` (the asset to deposit) and the function calculates how many LP tokens they receive. Since this is a single-asset deposit that changes the pool ratio, a [trading fee](#3-gettradingfee) applies. The function uses [`lpTokensOut`](helpers.md#321-lptokensout-equation-3) to calculate the LP tokens based on the deposit amount and trading fee, then adjusts the result for precision with [`adjustLPTokensOut`](helpers.md#24-adjustlptokensout-deposits). The adjusted tokens are passed to [`adjustAssetInByTokens`](helpers.md#26-adjustassetinbytokens), which prices them in the deposited asset. If rounding makes them cost more than `Amount`, it issues fewer LP tokens and caps the deposit at `Amount`. The optional `LPTokenOut` field sets a minimum. If fewer LP tokens would be issued, the transaction fails with `tecAMM_FAILED`.

### 5.1.1. singleDeposit Pseudo-Code

```python
def singleDeposit(pool, balance1, balance2, lp_supply, tx, fee):
    lp_tokens = adjustLPTokensOut(lp_supply, lpTokensOut(balance1, tx.Amount, lp_supply, fee))
    if lp_tokens == 0:
        fail(tecAMM_INVALID_TOKENS)

    # If rounding makes these LP tokens cost more than Amount, fewer are issued
    # and the deposit is capped at Amount
    lp_tokens, amount = adjustAssetInByTokens(balance1, tx.Amount, lp_supply, lp_tokens, fee)
    if lp_tokens == 0:
        fail(tecAMM_INVALID_TOKENS)

    return deposit(pool, amount, None, lp_tokens, min_lp_tokens=tx.LPTokenOut)
```

## 5.2. singleDepositTokens (tfOneAssetLPToken)

> "I want X LP tokens, how much do I need to deposit (single asset only)?"

Deposit a single asset to receive a requested number of LP tokens.[^single-deposit-tokens]

[^single-deposit-tokens]: `AMMDeposit::singleDepositTokens`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L838-L866)

This function handles the reverse calculation from [`singleDeposit`](#51-singledeposit-tfsingleasset). The user asks for LP tokens with `LPTokenOut`, and the function calculates how much of a single asset they must deposit. `Amount` is the most the user is willing to deposit. The function first rounds the requested LP tokens down to a value the LP token supply can represent, using [`adjustLPTokensOut`](helpers.md#24-adjustlptokensout-deposits), so the user can receive slightly fewer than `LPTokenOut`. It then uses [`ammAssetIn`](helpers.md#322-ammassetin-equation-4) (Equation 4) to calculate the required deposit, rounded up. If that amount exceeds `Amount`, the transaction fails with `tecAMM_FAILED`.

### 5.2.1. singleDepositTokens Pseudo-Code

```python
def singleDepositTokens(pool, balance1, balance2, lp_supply, tx, fee):
    lp_tokens = adjustLPTokensOut(lp_supply, tx.LPTokenOut)
    if lp_tokens == 0:
        fail(tecAMM_INVALID_TOKENS)

    amount = ammAssetIn(balance1, lp_supply, lp_tokens, fee)
    if amount > tx.Amount:
        fail(tecAMM_FAILED)

    return deposit(pool, amount, None, lp_tokens)
```

## 5.3. singleDepositEPrice (tfLimitLPToken)

> "I'll deposit (single asset), but only if the effective price per LP token is reasonable."

Deposit a single asset with an effective price limit.[^single-deposit-eprice]

[^single-deposit-eprice]: `AMMDeposit::singleDepositEPrice`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L894-L991)

This mode lets the user cap the [effective price](README.md#114-effective-price) they pay per LP token: the asset deposited divided by the LP tokens received. The user provides `EPrice`, the maximum effective price, and `Amount`, which can be zero.

If `Amount` is not zero, the function first prices a deposit of all of `Amount`, the same way as [`singleDeposit`](#51-singledeposit-tfsingleasset). If its effective price is within `EPrice`, that deposit goes ahead.

Otherwise, or when `Amount` is zero, the function finds the deposit whose effective price is exactly `EPrice`. The effective price grows with the size of the deposit, because the deposit moves the pool's price against the depositor. So this is the most the user can deposit within the limit. When `Amount` was given, it should be smaller than `Amount`. The function finds it by substituting `t = b / E` into Equation 3 and solving the resulting quadratic for `b`. Rounding can move the final amounts slightly off `EPrice`.

**Example:**

Alice created an AMM with 100 USD and 100 EUR, 100 LP tokens outstanding (0.3% trading fee). Bob wants to deposit USD but doesn't want to pay more than 2.5 USD per LP token.

**Scenario 1: Bob specifies amount**
- Bob provides: Amount = 100 USD, EPrice = 2.5 USD per LP token
- System calculates: Using Equation 3, 100 USD yields ~41.36 LP tokens
- Effective price: 100 / 41.36 = ~2.42 USD per LP token
- Result: 2.42 < 2.5, so the deposit proceeds (100 USD for 41.36 LP tokens)

**Scenario 2: Bob's amount is above the price limit**
- Bob provides: Amount = 200 USD, EPrice = 2.5 USD per LP token
- 200 USD would yield ~73.10 LP tokens, an effective price of ~2.74, which is above the limit
- The system solves for the deposit at 2.5: Bob deposits ~123.88 USD and receives ~49.55 LP tokens
- With Amount = 0, Bob gets the same result

### 5.3.1. singleDepositEPrice Pseudo-Code

```python
def singleDepositEPrice(pool, balance1, balance2, lp_supply, tx, fee):
    if tx.Amount != 0:
        lp_tokens = adjustLPTokensOut(lp_supply, lpTokensOut(balance1, tx.Amount, lp_supply, fee))
        if lp_tokens <= 0:
            fail(tecAMM_INVALID_TOKENS)
        lp_tokens, amount = adjustAssetInByTokens(balance1, tx.Amount, lp_supply, lp_tokens, fee)
        if lp_tokens == 0:
            fail(tecAMM_INVALID_TOKENS)
        if amount / lp_tokens <= tx.EPrice:
            return deposit(pool, amount, None, lp_tokens)

    amount = round_up(balance1 * deposit_ratio_at_price(balance1, lp_supply, tx.EPrice, fee))
    if amount <= 0:
        fail(tecAMM_FAILED)

    lp_tokens = adjustLPTokensOut(lp_supply, round_down(amount / tx.EPrice))
    lp_tokens, amount = adjustAssetInByTokens(balance1, amount, lp_supply, lp_tokens, fee)
    if lp_tokens == 0:
        fail(tecAMM_INVALID_TOKENS)

    return deposit(pool, amount, None, lp_tokens)
```

`deposit_ratio_at_price` solves the quadratic, with `B` the pool balance, `T` the LP token supply and `E` the effective price:

```
f1 = 1 - fee
f2 = (1 - fee/2) / f1
c  = f1 * B / (E * T)
d  = f1 + c*f2 - c

c^2 * R^2 + (c^2*f2^2 + 2c - d^2) * R + (2c*f2^2 + 1 - 2d*f2) = 0

deposit_ratio_at_price = f1 * R, where R is the positive root
```

# 6. Common Deposit Function

The `deposit()` function[^deposit] is called by all deposit modes to perform the actual asset transfers. This function validates minimum constraints for slippage protection, checks the depositor has sufficient funds for the deposit, transfers the assets from the depositor to the AMM account, and issues LP tokens to the depositor (creating a trust line if needed).

[^deposit]: `AMMDeposit::deposit`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L543-L670)

## 6.1. deposit Pseudo-Code

```python
def deposit(pool, amount1, amount2, lp_tokens, min_amount1=0, min_amount2=0, min_lp_tokens=0):
    if (amount1 < min_amount1 or lp_tokens < min_lp_tokens
            or (amount2 is not None and amount2 < min_amount2)):
        fail(tecAMM_FAILED)

    for amount in [amount1, amount2]:
        if amount is None:  # single-asset deposit
            continue
        if amount <= 0:
            fail(temBAD_AMOUNT)
        # For XRP, the balance above the reserve, counting a new LP token trust line if one is needed
        if spendable(depositor, amount.asset) < amount:
            fail(tecUNFUNDED_AMM)
        send(amount, from_=depositor, to=pool.account, transfer_fee=False)

    send(lp_tokens, from_=pool.account, to=depositor)
    return pool.lp_token_supply + lp_tokens
```
