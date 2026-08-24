# Index

- [1. Introduction](#1-introduction)
- [2. applyGuts](#2-applyguts)
  - [2.1. applyGuts Pseudo-Code](#21-applyguts-pseudo-code)
- [3. getTradingFee](#3-gettradingfee)
  - [3.1. getTradingFee Pseudo-Code](#31-gettradingfee-pseudo-code)
- [4. Multi-Asset Deposit Modes](#4-multi-asset-deposit-modes)
  - [4.1. equalDepositLimit (tfTwoAsset)](#41-equaldepositlimit-tftwoasset)
    - [4.1.1. equalDepositLimit Pseudo-Code](#411-equaldepositlimit-pseudo-code)
  - [4.2. equalDepositTokens (tfLPToken)](#42-equaldeposittokens-tflptoken)
    - [4.2.1. equalDepositTokens Pseudo-Code](#421-equaldeposittokens-pseudo-code)
  - [4.3. equalDepositInEmptyState (tfTwoAssetIfEmpty)](#43-equaldepositinemptystate-tftwoassetifempty)
    - [4.3.1. equalDepositInEmptyState Pseudo-Code](#431-equaldepositinemptystate-pseudo-code)
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

# 2. applyGuts

The `applyGuts` function[^apply-guts] is the main entry point for processing AMMDeposit transactions.

It retrieves the AMM ledger entry and current pool balances, then determines which trading fee applies to the depositor (regular or discounted for [auction slot holders](#3-gettradingfee)). Based on the transaction flags and provided fields, it dispatches to one of deposit mode handlers: three [multi-asset modes](#4-multi-asset-deposit-modes) that maintain proportional deposits or reinitialize empty pools, and three [single-asset modes](#5-single-asset-deposit-modes) that perform single-sided deposits. Each mode handler calculates the deposit amounts and LP tokens to issue, then calls the [common deposit function](#6-common-deposit-function) to execute the actual asset transfers and update the pool state.

Under the `fixCleanup3_3_0` amendment, `preclaim` runs the freeze checks for both pool assets whether or not they are deposited. See the [failure conditions](README.md#322-failure-conditions) for the resulting behavior change.

[^apply-guts]: `AMMDeposit::applyGuts`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L412-L526) 

## 2.1. applyGuts Pseudo-Code

```python
def applyGuts(sb: &Sandbox, tx: Transaction):
    amount = tx[sfAmount]
    amount2 = tx[sfAmount2]
    ePrice = tx[sfEPrice]
    ammSle = sb.getAMM(amount, amount2)
    if not ammSle:
        return tecINTERNAL

    ammAccountId = ammSle[sfAccount]

    # In `xrpld`, this is called "expected", but probably only to reflect the returned type. A better name is "existing" or "currentBalances"
    # FreezeHandling.ZeroIfFrozen: treat frozen assets as having zero balance
    # AuthHandling.ZeroIfUnauthorized: treat unauthorized MPT holders as having zero balance
    currentBalances = ammHolds(sb, ammSle, amount, amount2, FreezeHandling.ZeroIfFrozen, AuthHandling.ZeroIfUnauthorized)
    if not currentBalances:
        return currentBalances.error()

    # Current pool balance of asset1
    # Current pool balance of asset2
    # Total outstanding LP Tokens
    amountBalance, amount2Balance, lptAMMBalance = currentBalances

    # Determine which trading fee to use
    if lptAMMBalance == 0:
        # Empty pool: depositor can set new trading fee (like AMMCreate)
        tfee = tx[sfTradingFee] if tx.has(sfTradingFee) else 0
    else:
        # Normal pool: get current trading fee (with potential discount)
        # Returns discounted fee if account is auction slot holder or authorized
        # Otherwise returns regular AMM trading fee
        tfee = getTradingFee(view, ammSle, accountID_)

    subTxType = tx.getFlags() & tfDepositSubTx

    # Dispatch to appropriate deposit mode handler
    # Returns (result_code, new_lp_token_balance)
    if subTxType == tfTwoAsset:
        # Proportional deposit with max constraints on both assets
        result, newLPTokenBalance = equalDepositLimit(
            sb,
            ammAccountID,
            amountBalance,
            amount2Balance,
            lptAMMBalance,
            amount,              # max amount1 to deposit
            amount2,             # max amount2 to deposit
            lpTokensDeposit,     # optional: min LP tokens expected
            tfee
        )

    elif subTxType == tfOneAssetLPToken:
        # Single asset deposit for exact LP tokens
        result, newLPTokenBalance = singleDepositTokens(
            sb,
            ammAccountID,
            amountBalance,
            amount,              # max amount to pay
            lptAMMBalance,
            lpTokensDeposit,     # exact LP tokens desired
            tfee
        )

    elif subTxType == tfLimitLPToken:
        # Single asset deposit with effective price limit
        result, newLPTokenBalance = singleDepositEPrice(
            sb,
            ammAccountID,
            amountBalance,
            amount,              # amount to deposit
            lptAMMBalance,
            ePrice,              # max effective price
            tfee
        )

    elif subTxType == tfSingleAsset:
        # Single asset deposit for calculated LP tokens
        result, newLPTokenBalance = singleDeposit(
            sb,
            ammAccountID,
            amountBalance,
            lptAMMBalance,
            amount,              # amount to deposit
            lpTokensDeposit,     # optional: min LP tokens expected
            tfee
        )

    elif subTxType == tfLPToken:
        # Proportional deposit for exact LP tokens
        result, newLPTokenBalance = equalDepositTokens(
            sb,
            ammAccountID,
            amountBalance,
            amount2Balance,
            lptAMMBalance,
            lpTokensDeposit,     # exact LP tokens desired
            amount,              # optional: min amount1 constraint
            amount2,             # optional: min amount2 constraint
            tfee
        )

    elif subTxType == tfTwoAssetIfEmpty:
        # Empty pool reinitialization (like AMMCreate)
        result, newLPTokenBalance = equalDepositInEmptyState(
            sb,
            ammAccountID,
            amount,              # amount1 to deposit
            amount2,             # amount2 to deposit
            lptAMMBalance.asset(),
            tfee
        )

    else:
        # Should not happen (validated in preflight)
        return (tecINTERNAL, False)

    # Update AMM state if deposit succeeded
    if result == tesSUCCESS:
        assert newLPTokenBalance > 0

        # Update LP token balance in AMM object
        ammSle.set(sfLPTokenBalance, newLPTokenBalance)

        # If pool was empty, initialize auction/vote slots
        if lptAMMBalance == 0:
            initializeFeeAuctionVote(
                sb,
                ammSle,
                accountID_,
                lptAMMBalance.asset(),
                tfee
            )

        sb.update(ammSle)

    return (result, result == tesSUCCESS)

```

# 3. getTradingFee

The `getTradingFee` function[^get-trading-fee] is called by [`applyGuts`](#2-applyguts) to determine which trading fee rate applies to the depositor before calculating deposit amounts.

It checks if the depositor holds the [auction slot](README.md#121-auction-slot) or is listed in the slot's authorized accounts and if the auction slot has not expired. If so, it returns the discounted fee (1/10th of the regular fee). Otherwise, it returns the AMM's standard trading fee.

[^get-trading-fee]: `getTradingFee`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L602-L627)  

## 3.1. getTradingFee Pseudo-Code

```python
def getTradingFee(view, ammSle, account):
    # Check if auction slot exists
    if not ammSle.isFieldPresent(sfAuctionSlot):
        return ammSle[sfTradingFee]

    auctionSlot = ammSle.peekAtField(sfAuctionSlot)

    # Check if auction slot is NOT expired
    currentTime = view.parentCloseTime()
    expiration = auctionSlot[~sfExpiration]

    if currentTime < expiration:
        # Auction slot is active, check if account gets discount

        # Is account the auction slot holder?
        if auctionSlot[~sfAccount] == account:
            return auctionSlot[sfDiscountedFee]

        # Is account in the authorized accounts list?
        if auctionSlot.isFieldPresent(sfAuthAccounts):
            for authAccount in auctionSlot.getFieldArray(sfAuthAccounts):
                if authAccount[~sfAccount] == account:
                    return auctionSlot[sfDiscountedFee]

    # Auction slot expired or account not authorized, return regular fee
    return ammSle[sfTradingFee]
```

# 4. Multi-Asset Deposit Modes

Multi-asset deposit modes maintain proportional deposits by adding both pool assets simultaneously. This preserves the pool's asset ratio while increasing liquidity. Since these deposits maintain proportional ratios, they incur no trading fees.

The three modes are:
- [`equalDepositLimit`](#41-equaldepositlimit-tftwoasset) - Specify maximum amounts for both assets, receive proportional LP tokens
- [`equalDepositTokens`](#42-equaldeposittokens-tflptoken) - Specify exact LP tokens to receive, system calculates required asset amounts
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
def equalDepositLimit(
        view,                 # Ledger view (sandbox)
        ammAccount,           # AMM pseudo-account ID
        amountBalance,        # Current pool balance of asset1
        amount2Balance,       # Current pool balance of asset2
        lptAMMBalance,        # Total outstanding LP tokens
        amount,               # User's max amount1 to deposit (from tx[sfAmount])
        amount2,              # User's max amount2 to deposit (from tx[sfAmount2])
        lpTokensDepositMin,   # Optional: min LP tokens expected (from tx[sfLPTokenOut])
        tfee):                # Trading fee (regular or discounted)
    # STRATEGY 1: Try using all of amount (asset1)
    frac = amount / amountBalance

    # Calculate LP tokens for this fraction (with precision adjustment)
    tokensAdj = getRoundedLPTokens(rules, lptAMMBalance, frac, IsDeposit=True) # helpers.md#211-getroundedlptokens-simple-pseudo-code

    if tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, None)

    # Adjust fraction based on rounded tokens (for precision consistency)
    frac = adjustFracByTokens(rules, lptAMMBalance, tokensAdj, frac)

    # Calculate how much asset2 is needed for this fraction
    amount2Deposit = getRoundedAsset(rules, amount2Balance, frac, IsDeposit=True) # helpers.md#231-getroundedasset-simple-pseudo-code

    # Check if calculated amount2 fits within user's max constraint
    if amount2Deposit <= amount2:
        # Success! Use all of amount, calculated amount2Deposit
        return deposit(
            view,
            ammAccount,
            amountBalance,
            amount,              # deposit all of asset1
            amount2Deposit,      # calculated asset2
            lptAMMBalance,
            tokensAdj,
            None,                # no min constraint on amount
            None,                # no min constraint on amount2
            lpTokensDepositMin,  # check LP token minimum
            tfee
        )

    # STRATEGY 2: Strategy 1 failed, try using all of amount2
    frac = amount2 / amount2Balance

    # Calculate LP tokens for this fraction
    # Using simple version of getRoundedLPTokens (direct fraction)
    tokensAdj = getRoundedLPTokens(rules, lptAMMBalance, frac, IsDeposit=True) # helpers.md#211-getroundedlptokens-simple-pseudo-code

    if tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, None)

    # Adjust fraction based on rounded tokens
    frac = adjustFracByTokens(rules, lptAMMBalance, tokensAdj, frac)

    # Calculate how much asset1 is needed for this fraction
    amountDeposit = getRoundedAsset(rules, amountBalance, frac, IsDeposit=True) # helpers.md#231-getroundedasset-simple-pseudo-code

    # Check if calculated amount fits within user's max constraint
    if amountDeposit <= amount:
        # Success! Use calculated amountDeposit, all of amount2
        return deposit(
            view,
            ammAccount,
            amountBalance,
            amountDeposit,       # calculated asset1
            amount2,             # deposit all of asset2
            lptAMMBalance,
            tokensAdj,
            None,                # no min constraint on amount
            None,                # no min constraint on amount2
            lpTokensDepositMin,  # check LP token minimum
            tfee
        )

    # Both options failed - cannot satisfy constraints
    return (tecAMM_FAILED, None)
```

## 4.2. equalDepositTokens (tfLPToken)

> "I want exactly X LP tokens, what do I need to deposit?"

Proportional deposit for exact LP tokens.[^equal-deposit-tokens]

[^equal-deposit-tokens]: `AMMDeposit::equalDepositTokens`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L637-L679)

This function handles the reverse calculation from [`equalDepositLimit`](#41-equaldepositlimit-tftwoasset): the user specifies the exact number of LP tokens they want to receive, and the function calculates the required proportional amounts of both assets. The user provides `LPTokenOut` (exact LP tokens desired) and optionally `Amount` and `Amount2` as minimum constraints on the deposit amounts. The function first adjusts the requested LP tokens for precision using [`adjustLPTokensOut`](helpers.md#24-adjustlptokensout-deposits), then calculates the pool fraction these tokens represent (`frac = tokensAdj / lptAMMBalance`). Using this fraction, it calculates the required amounts of both assets by multiplying each pool balance by the fraction, rounding up with [`getRoundedAsset`](helpers.md#23-getroundedasset) to ensure the pool receives sufficient assets. If the calculated deposit amounts are less than the optional `Amount` and `Amount2` minimums, the transaction fails with `tecAMM_FAILED`.

**Example:**

Alice created an AMM with 100 USD, 100 EUR, and 100 LP tokens outstanding. Bob wants to receive exactly 25 LP tokens.

- Desired LP tokens: 25
- Fraction of pool: 25 / 100 = 0.25 (25%)
- Required USD: 100 * 0.25 = 25 USD
- Required EUR: 100 * 0.25 = 25 EUR
- Result: Bob deposits 25 USD and 25 EUR, receives 25 LP tokens


### 4.2.1. equalDepositTokens Pseudo-Code

```python
def equalDepositTokens(
        view,              # Ledger view (sandbox)
        ammAccount,        # AMM pseudo-account ID
        amountBalance,     # Current pool balance of asset1
        amount2Balance,    # Current pool balance of asset2
        lptAMMBalance,     # Total outstanding LP tokens
        lpTokensDeposit,   # Exact LP tokens desired (from tx[sfLPTokenOut])
        depositMin,        # Optional: min amount1 constraint (from tx[sfAmount])
        deposit2Min,       # Optional: min amount2 constraint (from tx[sfAmount2])
        tfee):             # Trading fee (not used for proportional deposits)
    # Adjust LP tokens for precision (with fixAMMv1_3)
    tokensAdj = adjustLPTokensOut(rules, lptAMMBalance, lpTokensDeposit) # helpers.md#24-adjustlptokensout-deposits

    if tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, None)

    # Calculate the fraction of the pool these tokens represent
    frac = tokensAdj / lptAMMBalance

    # Calculate required deposit amounts for both assets
    # Both assets must be deposited proportionally to maintain the pool ratio
    amountDeposit = getRoundedAsset(rules, amountBalance, frac, IsDeposit=True) # helpers.md#231-getroundedasset-simple-pseudo-code
    amount2Deposit = getRoundedAsset(rules, amount2Balance, frac, IsDeposit=True) # helpers.md#231-getroundedasset-simple-pseudo-code

    # Execute the deposit
    return deposit(
        view,
        ammAccount,
        amountBalance,
        amountDeposit,       # calculated: amount1 to deposit
        amount2Deposit,      # calculated: amount2 to deposit
        lptAMMBalance,
        tokensAdj,           # exact: LP tokens to issue
        depositMin,          # optional: min amount1 expected to deposit
        deposit2Min,         # optional: min amount2 expected to deposit
        None,                # no min LP tokens (exact amount specified)
        tfee
    )
```

## 4.3. equalDepositInEmptyState (tfTwoAssetIfEmpty)

Reinitialize an empty AMM pool.[^equal-deposit-in-empty-state]

[^equal-deposit-in-empty-state]: `AMMDeposit::equalDepositInEmptyState`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L994-L1014)

This is a special mode that handles the case where all LP tokens have been withdrawn from an AMM, which means both asset balances are also zero (enforced by the [withdrawal system](README.md#13-ammwithdraw)). The depositor specifies both `Amount` and `Amount2` to set a new pool ratio, similar to [`AMMCreate`](README.md#11-ammcreate). The function calculates initial LP tokens using the geometric mean formula (`sqrt(amount * amount2)`), the same formula used when creating a new AMM. The depositor can optionally provide `TradingFee` to set a new fee for the pool. Unlike other deposit modes, there are no minimum constraints since the depositor is establishing the initial terms for the reinitialized pool.

### 4.3.1. equalDepositInEmptyState Pseudo-Code

```python
def equalDepositInEmptyState(
        view,              # Ledger view (sandbox)
        ammAccount,        # AMM pseudo-account ID
        amount,            # Amount of asset1 to deposit (from tx[sfAmount])
        amount2,           # Amount of asset2 to deposit (from tx[sfAmount2])
        lptIssue,          # LP token issue (currency/issuer)
        tfee):             # Optional: trading fee to set (from tx[sfTradingFee])
    lpTokens = sqrt(amount * amount2)

    # Execute the deposit with special empty-pool handling
    # Note the unusual parameters:
    #   - amountBalance = amount (not the current balance, since pool is empty)
    #   - lptAMMBalance = 0 (pool is empty, and asset balances are also 0)
    #   - No minimum constraints (depositor sets the terms)
    return deposit(
        view,
        ammAccount,
        amount,              # amountBalance: used as reference (pool is empty)
        amount,              # amountDeposit: actual amount to deposit
        amount2,             # amount2Deposit: actual amount2 to deposit
        STAmount(lptIssue, 0),  # lptAMMBalance: 0 (empty pool)
        lpTokens,            # lpTokensDeposit: initial LP tokens to mint
        None,                # no min constraint on amount
        None,                # no min constraint on amount2
        None,                # no min constraint on LP tokens
        tfee
    )
```

# 5. Single-Asset Deposit Modes

Single-asset deposits allow users to deposit only one asset instead of both assets proportionally. Unlike [multi-asset deposits](#4-multi-asset-deposit-modes) that maintain the pool ratio, single-asset deposits change the pool composition. Because they alter the pool ratio, [trading fees](#3-gettradingfee) apply to single-asset deposits. There are three modes:

- **[singleDeposit](#51-singledeposit-tfsingleasset) (tfSingleAsset)**: User specifies deposit amount, receives calculated LP tokens
- **[singleDepositTokens](#52-singledeposittokens-tfoneassetlptoken) (tfOneAssetLPToken)**: User specifies exact LP tokens desired, deposits calculated amount
- **[singleDepositEPrice](#53-singledepositeprice-tflimitlptoken) (tfLimitLPToken)**: User specifies deposit amount and effective price limit

## 5.1. singleDeposit (tfSingleAsset)

> "I'll deposit X amount, how many LP tokens will I get?"

Deposit a single asset to receive calculated LP tokens.[^single-deposit]

[^single-deposit]: `AMMDeposit::singleDeposit`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L791-L828)

The user specifies `Amount` (the asset to deposit) and the function calculates how many LP tokens they receive. Since this is a single-asset deposit that changes the pool ratio, a [trading fee](#3-gettradingfee) applies. The function uses [`lpTokensOut`](helpers.md#321-lptokensout-equation-3) to calculate the LP tokens based on the deposit amount and trading fee, then adjusts the result for precision with [`adjustLPTokensOut`](helpers.md#24-adjustlptokensout-deposits). The adjusted tokens are passed to [`adjustAssetInByTokens`](helpers.md#26-adjustassetinbytokens) to recalculate the deposit amount, ensuring the reverse calculation doesn't exceed the user's specified amount due to rounding. The optional `LPTokenOut` field provides a minimum constraint - if the calculated LP tokens are less than this value, the transaction fails with `tecAMM_FAILED`.

### 5.1.1. singleDeposit Pseudo-Code

```python
def singleDeposit(
        view,              # Ledger view (sandbox)
        ammAccount,        # AMM pseudo-account ID
        amountBalance,     # Current pool balance of the asset
        lptAMMBalance,     # Total outstanding LP tokens
        amount,            # User's deposit amount (from tx[sfAmount])
        lpTokensDepositMin,  # Optional: min LP tokens expected (from tx[sfLPTokenOut])
        tfee):             # Trading fee (for single-asset deposit)
    tokens = lpTokensOut(amountBalance, amount, lptAMMBalance, tfee) # helpers.md#321-lptokensout-equation-3

    # Adjust tokens for precision (with fixAMMv1_3)
    tokensAdj = adjustLPTokensOut(rules, lptAMMBalance, tokens) # helpers.md#24-adjustlptokensout-deposits

    if tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, None)

    # Adjust deposit amount based on adjusted tokens
    tokensAdj, amountDepositAdj = adjustAssetInByTokens(
        rules, amountBalance, amount, lptAMMBalance, tokensAdj, tfee)

    if tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, None)

    # Execute the deposit
    # lpTokensDepositMin provides slippage protection
    return deposit(
        view,
        ammAccount,
        amountBalance,
        amountDepositAdj,    # adjusted: actual amount to deposit
        None,                # single-asset deposit (no asset2)
        lptAMMBalance,
        tokensAdj,           # calculated: LP tokens to issue
        None,                # no min constraint on asset amount
        None,                # no asset2
        lpTokensDepositMin,  # optional: min LP tokens constraint
        tfee
    )
```

## 5.2. singleDepositTokens (tfOneAssetLPToken)

> "I want exactly X LP tokens, how much do I need to deposit (single asset only)?"

Deposit a single asset to receive exact LP tokens.[^single-deposit-tokens]

[^single-deposit-tokens]: `AMMDeposit::singleDepositTokens`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L838-L866)

This function handles the reverse calculation from [`singleDeposit`](#51-singledeposit-tfsingleasset): the user specifies the exact number of LP tokens they want to receive using `LPTokenOut`, and the function calculates the required deposit amount of a single asset. The user provides `Amount` as a maximum constraint on how much they're willing to deposit. The function first adjusts the requested LP tokens for precision using [`adjustLPTokensOut`](helpers.md#24-adjustlptokensout-deposits), then uses [`ammAssetIn`](helpers.md#322-ammassetin-equation-4) (Equation 4) to calculate the required deposit amount by solving the inverse single-asset deposit problem. If the calculated amount exceeds the user's `Amount` constraint, the transaction fails with `tecAMM_FAILED`.

### 5.2.1. singleDepositTokens Pseudo-Code

```python
def singleDepositTokens(
        view,              # Ledger view (sandbox)
        ammAccount,        # AMM pseudo-account ID
        amountBalance,     # Current pool balance of the asset
        amount,            # User's max amount willing to pay (from tx[sfAmount])
        lptAMMBalance,     # Total outstanding LP tokens
        lpTokensDeposit,   # Exact LP tokens desired (from tx[sfLPTokenOut])
        tfee):             # Trading fee (for single-asset deposit)
    # Adjust LP tokens for precision (with fixAMMv1_3)
    tokensAdj = adjustLPTokensOut(rules, lptAMMBalance, lpTokensDeposit) # helpers.md#24-adjustlptokensout-deposits

    if tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, None)

    # Calculate how much asset is needed to get these LP tokens
    amountDeposit = ammAssetIn(amountBalance, lptAMMBalance, tokensAdj, tfee) # helpers.md#322-ammassetin-equation-4

    # Check if calculated deposit exceeds user's maximum
    if amountDeposit > amount:
        # User's maximum is too low for desired LP tokens
        return (tecAMM_FAILED, None)

    # Execute the deposit
    return deposit(
        view,
        ammAccount,
        amountBalance,
        amountDeposit,    # calculated: how much asset to deposit
        None,             # single-asset deposit (no asset2)
        lptAMMBalance,
        tokensAdj,        # exact LP tokens to issue
        None,             # no min constraint on asset
        None,             # no asset2
        None,             # no min LP tokens (exact amount specified)
        tfee
    )
```

## 5.3. singleDepositEPrice (tfLimitLPToken)

> "I'll deposit (single asset), but only if the effective price per LP token is reasonable."

Deposit a single asset with an effective price limit.[^single-deposit-eprice]

[^single-deposit-eprice]: `AMMDeposit::singleDepositEPrice`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L894-L991)

This mode allows users to control the maximum [effective price](README.md#114-effective-price) they're willing to pay per LP token (effective price = asset deposited / LP tokens received). The user provides `EPrice` (maximum effective price) and `Amount` (deposit amount, or zero). 

There are two scenarios: If `Amount` is non-zero, the function calculates the LP tokens using [`lpTokensOut`](helpers.md#321-lptokensout-equation-3), adjusts for precision, and checks if the resulting effective price is within the limit - if acceptable, the deposit proceeds; otherwise it falls through to scenario 2. 

If `Amount` is zero (or the effective price check failed), the function solves a quadratic equation derived from Equation 3 to calculate the optimal deposit amount that achieves exactly the specified effective price. The derivation inverts the single-asset deposit formula to find the deposit amount that results in the target effective price. 

**Example:**

Alice created an AMM with 100 USD and 100 EUR, 100 LP tokens outstanding (0.3% trading fee). Bob wants to deposit USD but doesn't want to pay more than 2.5 USD per LP token.

**Scenario 1: Bob specifies amount**
- Bob provides: Amount = 100 USD, EPrice = 2.5 USD per LP token
- System calculates: Using Equation 3, 100 USD yields ~41.36 LP tokens
- Effective price: 100 / 41.36 = ~2.42 USD per LP token
- Result: 2.42 < 2.5, so the deposit proceeds (100 USD for 41.36 LP tokens)

**Scenario 2: Bob specifies only effective price**
- Bob provides: Amount = 0, EPrice = 2.5 USD per LP token
- System calculates: Solves quadratic to find optimal deposit for exactly 2.5 USD/LP
- Result: System calculates the exact deposit amount that yields 2.5 USD per LP token

### 5.3.1. singleDepositEPrice Pseudo-Code

```python
def singleDepositEPrice(
        view,              # Ledger view (sandbox)
        ammAccount,        # AMM pseudo-account ID
        amountBalance,     # Current pool balance of the asset
        amount,            # User's deposit amount (from tx[sfAmount], can be 0)
        lptAMMBalance,     # Total outstanding LP tokens
        effectivePrice,    # Max effective price (from tx[sfEPrice])
        tfee):             # Trading fee (for single-asset deposit)
    # SCENARIO 1: User provided a specific deposit amount
    # Check if the effective price for this deposit is acceptable
    if amount != 0:
        # Calculate LP tokens we'd get for this deposit
        # lpTokensOut uses Equation 3 (single-asset deposit formula)
        tokens = lpTokensOut(amountBalance, amount, lptAMMBalance, tfee) # helpers.md#321-lptokensout-equation-3
        tokens = adjustLPTokensOut(rules, lptAMMBalance, tokens) # helpers.md#24-adjustlptokensout-deposits

        if tokens <= 0:
            return (tecAMM_INVALID_TOKENS, None)

        # Adjust the deposit amount if rounding caused issues
        # This ensures we don't try to deposit more than the user's maximum
        tokensAdj, amountDepositAdj = adjustAssetInByTokens(
            rules, amountBalance, amount, lptAMMBalance, tokens, tfee) # helpers.md#26-adjustassetinbytokens

        if tokensAdj == 0:
            return (tecAMM_INVALID_TOKENS, None)

        # Calculate the actual effective price
        # effectivePriceActual = asset deposited / LP tokens received
        # Example: 1,000 XRP / 95 LP tokens = 10.526 XRP per LP token
        effectivePriceActual = amountDepositAdj / tokensAdj

        # Check if the effective price is within user's limit
        if effectivePriceActual <= effectivePrice:
            # Good price! Proceed with deposit
            return deposit(
                view,
                ammAccount,
                amountBalance,
                amountDepositAdj,
                None,             # single-asset deposit
                lptAMMBalance,
                tokensAdj,
                None, None, None, # no min constraints
                tfee
            )
        # else: Price too high, fall through to scenario 2

    # SCENARIO 2: User didn't provide amount OR price was too high
    # Calculate the optimal deposit for the given effective price limit
    #
    # We need to solve for the deposit amount that gives exactly the effective price.
    # Given: effectivePrice (E), find: assetDeposit (b) and lpTokens (t)
    # Such that: E = b / t
    #
    # Fee multipliers (same as ammAssetIn)
    feeMult = 1 - (tfee / 100000)
    feeMultHalf = (1 - (tfee / 200000)) / feeMult

    # Calculate intermediate values for the quadratic equation
    c = feeMult * amountBalance / (effectivePrice * lptAMMBalance)
    d = feeMult + c * feeMultHalf - c

    # Set up quadratic equation: a1*R^2 + b1*R + c1 = 0
    # Where R is the deposit ratio (see derivation in code)
    quadraticA = c * c
    quadraticB = (c * c * feeMultHalf * feeMultHalf) + (2 * c) - (d * d)
    quadraticC = (2 * c * feeMultHalf * feeMultHalf) + 1 - (2 * d * feeMultHalf)

    # Solve for R using quadratic formula: (-b + sqrt(b^2 - 4ac)) / (2a)
    discriminant = quadraticB * quadraticB - 4 * quadraticA * quadraticC
    R = (-quadraticB + sqrt(discriminant)) / (2 * quadraticA)

    # Apply rounding (upward for deposits with fixAMMv1_3)
    # Using callback version of getRoundedAsset for delayed evaluation
    assetDeposit = [getRoundedAsset](helpers.md#232-getroundedasset-callback-pseudo-code)(
        rules,
        lambda: feeMult * amountBalance * R,  # noRoundCb
        amountBalance,                         # balance
        lambda: feeMult * R,                   # productCb
        IsDeposit=True
    )

    if assetDeposit <= 0:
        return (tecAMM_FAILED, None)

    # Calculate LP tokens from the deposit and effective price
    # Using callback version of getRoundedLPTokens for delayed evaluation
    lpTokensCalculated = [getRoundedLPTokens](helpers.md#212-getroundedlptokens-callback-pseudo-code)(
        rules,
        lambda: assetDeposit / effectivePrice,  # noRoundCb
        lptAMMBalance,                           # lptAMMBalance
        lambda: assetDeposit / effectivePrice,   # productCb
        IsDeposit=True
    )

    # Adjust for rounding issues (same as scenario 1)
    tokensAdj, amountDepositAdj = [adjustAssetInByTokens](helpers.md#26-adjustassetinbytokens)(
        rules, amountBalance, assetDeposit, lptAMMBalance, lpTokensCalculated, tfee)

    if tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, None)

    # Execute the deposit
    return deposit(
        view,
        ammAccount,
        amountBalance,
        amountDepositAdj,
        None,             # single-asset deposit
        lptAMMBalance,
        tokensAdj,
        None, None, None, # no min constraints
        tfee
    )
```

# 6. Common Deposit Function

The `deposit()` function[^deposit] is called by all deposit modes to perform the actual asset transfers. This function validates minimum constraints for slippage protection, checks the depositor has sufficient funds for the deposit, transfers the assets from the depositor to the AMM account, and issues LP tokens to the depositor (creating a trust line if needed).

[^deposit]: `AMMDeposit::deposit`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L543-L670)

## 6.1. deposit Pseudo-Code

```python
def deposit(
        view,              # Ledger view (sandbox)
        ammAccount,        # AMM pseudo-account ID
        amountBalance,     # Current pool balance of asset1
        amountDeposit,     # Calculated amount1 to deposit
        amount2Deposit,    # Optional: calculated amount2 to deposit
        lptAMMBalance,     # Total outstanding LP tokens
        lpTokensDeposit,   # LP tokens to issue
        depositMin,        # Optional: min amount1 constraint
        deposit2Min,       # Optional: min amount2 constraint
        lpTokensDepositMin,  # Optional: min LP tokens constraint
        tfee):             # Trading fee
    # Adjust amounts for precision. With fixAMMv1_3 this should not change the values, because all rounding and precision calculations have been done already
    amountDepositActual, amount2DepositActual, lpTokensDepositActual = \
        adjustAmountsByLPTokens(
            amountBalance,
            amountDeposit,
            amount2Deposit,
            lptAMMBalance,
            lpTokensDeposit,
            tfee,
            isDeposit=True
        )

    # Validate adjusted LP tokens
    if lpTokensDepositActual <= 0:
        return (tecAMM_INVALID_TOKENS, None)

    # Check minimum constraints (slippage protection)
    if amountDepositActual < depositMin or \
       amount2DepositActual < deposit2Min or \
       lpTokensDepositActual < lpTokensDepositMin:
        return (tecAMM_FAILED, None)

    # Check balance for asset1 deposit
    # checkBalance returns error code (TER) if deposit would fail, tesSUCCESS otherwise
    # For XRP: checks liquid balance (accounting for reserve and potential LP token trustline creation)
    # For other assets: checks if amount <= 0, or if account holds sufficient amount (or is the issuer)
    ter = checkBalance(amountDepositActual)
    if ter != tesSUCCESS:
        return (ter, None)  # returns temBAD_AMOUNT if <= 0, tecUNFUNDED_AMM if insufficient

    # Transfer asset1 from depositor to AMM
    result = accountSend(
        view,
        account,              # from: depositor
        ammAccount,           # to: AMM pseudo-account
        amountDepositActual,  # amount
        WaiveTransferFee=Yes, # AMM deposits don't pay transfer fees
        AllowMPTOverflow=No   # Prevent MPT OutstandingAmount from exceeding MaximumAmount
    )
    if result != tesSUCCESS:
        return (result, None)

    # If two-asset deposit, check balance and transfer asset2
    if amount2DepositActual:
        ter = checkBalance(amount2DepositActual)
        if ter != tesSUCCESS:
            return (ter, None)

        result = accountSend(
            view,
            account,
            ammAccount,
            amount2DepositActual,
            WaiveTransferFee=Yes,
            AllowMPTOverflow=No
        )
        if result != tesSUCCESS:
            return (result, None)

    # Transfer LP tokens from AMM to depositor
    #    This creates a trust line if depositor doesn't have one
    result = accountSend(
        view,
        ammAccount,            # from: AMM pseudo-account
        account,               # to: depositor
        lpTokensDepositActual  # amount
    )
    if result != tesSUCCESS:
        return (result, None)

    # Return new total LP token balance
    return (tesSUCCESS, lptAMMBalance + lpTokensDepositActual)
```
