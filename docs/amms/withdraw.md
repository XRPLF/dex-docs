# Index

- [1. Introduction](#1-introduction)
- [2. applyGuts](#2-applyguts)
  - [2.1. applyGuts Pseudo-Code](#21-applyguts-pseudo-code)
- [3. getTradingFee](#3-gettradingfee)
- [4. Multi-Asset Withdrawal Modes](#4-multi-asset-withdrawal-modes)
  - [4.1. equalWithdrawTokens (tfLPToken, tfWithdrawAll)](#41-equalwithdrawtokens-tflptoken-tfwithdrawall)
    - [4.1.1 equalWithdrawTokens Pseudo-Code](#411-equalwithdrawtokens-pseudo-code)
  - [4.2. equalWithdrawLimit (tfTwoAsset)](#42-equalwithdrawlimit-tftwoasset)
    - [4.2.1 equalWithdrawLimit Pseudo-Code](#421-equalwithdrawlimit-pseudo-code)
- [5. Single-Asset Withdrawal Modes](#5-single-asset-withdrawal-modes)
  - [5.1. singleWithdraw (tfSingleAsset)](#51-singlewithdraw-tfsingleasset)
    - [5.1.1. singleWithdraw Pseudo-Code](#511-singlewithdraw-pseudo-code)
  - [5.2. singleWithdrawTokens (tfOneAssetLPToken, tfOneAssetWithdrawAll)](#52-singlewithdrawtokens-tfoneassetlptoken-tfoneassetwithdrawall)
    - [5.2.1. singleWithdrawTokens Pseudo-Code](#521-singlewithdrawtokens-pseudo-code)
  - [5.3. singleWithdrawEPrice (tfLimitLPToken)](#53-singlewithdraweprice-tflimitlptoken)
    - [5.3.1. singleWithdrawEPrice Pseudo-Code](#531-singlewithdraweprice-pseudo-code)
- [6. Common Withdraw Function](#6-common-withdraw-function)
  - [6.1. withdraw Pseudo-Code](#61-withdraw-pseudo-code)

# 1. Introduction

The AMMWithdraw transaction allows liquidity providers to redeem their LP tokens for underlying pool assets. This document provides technical implementation details for the withdrawal logic in `xrpld`.

The [AMM documentation](README.md#33-ammwithdraw-transaction) describes the high-level business logic of withdrawals, including different [withdrawal modes](README.md#331-withdrawal-modes), their purposes, and user-facing behavior. This document focuses on the implementation: how the code validates constraints, calculates withdrawal amounts, and executes asset transfers.

**Terminology Notes**:

Throughout the document we refer to equations by their number in the [XLS-30 specification](https://github.com/XRPLF/XRPL-Standards/tree/master/XLS-0030-automated-market-maker).
AMM helpers use **token** as a subject in many function names. This refers to any supported currency in the system, not only a particular token implementation, like trust lines or MPTs.

# 2. applyGuts

The `applyGuts` function[^applyGuts] is the main entry point for processing AMMWithdraw transactions. It retrieves the AMM ledger entry and the withdrawer's LP token balance, determines how many LP tokens to redeem (all tokens for `tfWithdrawAll`/`tfOneAssetWithdrawAll`, or the specified amount from `LPTokenIn`), then adjusts the LP token balance for precision if needed. The function gets the current pool balances and determines which trading fee applies to the withdrawer (regular or discounted for [auction slot holders](#3-gettradingfee)). Based on the transaction flags and provided fields, it dispatches to one of five withdrawal mode handlers (implementing seven total modes): two [multi-asset modes](#4-multi-asset-withdrawal-modes) that maintain proportional withdrawals, and three [single-asset modes](#5-single-asset-withdrawal-modes) that perform single-sided withdrawals. Each mode handler calculates the withdrawal amounts and LP tokens to burn, then calls the [common withdraw function](#6-common-withdraw-function) to execute the actual asset transfers and update the pool state. After the withdrawal, if the pool is empty (zero LP tokens), the function attempts to delete the AMM account - if successful, the AMM is fully removed; if incomplete due to remaining trust lines, the AMM remains in an empty state with the LP token balance set to zero.

Under the `fixCleanup3_3_0` amendment, the freeze rules relax as described in the [failure conditions](README.md#332-failure-conditions). In this path, an issuer withdrawing its own frozen token reads the pool balances with `IgnoreFreeze` instead of the `ZeroIfFrozen` shown in the pseudo-code below.

[^applyGuts]: AMMWithdraw::applyGuts: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L336-L462)

## 2.1. applyGuts Pseudo-Code

```python
def applyGuts(sb: &Sandbox, tx: Transaction):
    amount = tx[~sfAmount]
    amount2 = tx[~sfAmount2]
    ePrice = tx[~sfEPrice]
    ammSle = sb.getAMM(tx[sfAsset], tx[sfAsset2])
    if not ammSle:
        return (tecINTERNAL, False)

    ammAccountID = ammSle[sfAccount]

    # Get withdrawer's LP token balance
    lpTokens = ammLPHolds(view, ammSle, accountID_)

    # Determine LP tokens to withdraw
    # For tfWithdrawAll and tfOneAssetWithdrawAll: use all LP tokens
    # Otherwise: use tx[sfLPTokenIn]
    lpTokensWithdraw = tokensWithdraw(lpTokens, tx[~sfLPTokenIn], tx.getFlags())

    # Adjust LP token balance for precision (with fixAMMv1_1)
    # This handles rounding issues for the last LP
    if rules.enabled(fixAMMv1_1):
        if not verifyAndAdjustLPTokenBalance(sb, lpTokens, ammSle, accountID_):
            return (tecAMM_INVALID_TOKENS, False)

    # Get current trading fee (with potential discount)
    tfee = getTradingFee(view, ammSle, accountID_)

    # Get current pool balances
    # FreezeHandling.ZeroIfFrozen: treat frozen assets as having zero balance
    # AuthHandling.ZeroIfUnauthorized: treat unauthorized MPT holders as having zero balance
    currentBalances = ammHolds(sb, ammSle, amount.asset(), amount2.asset(), FreezeHandling.ZeroIfFrozen, AuthHandling.ZeroIfUnauthorized)
    if not currentBalances:
        return (currentBalances.error(), False)

    amountBalance, amount2Balance, lptAMMBalance = currentBalances

    subTxType = tx.getFlags() & tfWithdrawSubTx

    # Dispatch to appropriate withdrawal mode handler
    # Returns (result_code, new_lp_token_balance)
    if subTxType & tfTwoAsset:
        # Proportional withdrawal with max constraints on both assets
        result, newLPTokenBalance = equalWithdrawLimit(
            sb,
            ammSle,
            ammAccountID,
            amountBalance,
            amount2Balance,
            lptAMMBalance,
            amount,              # max amount1 to withdraw
            amount2,             # max amount2 to withdraw
            tfee
        )

    elif subTxType & (tfOneAssetLPToken | tfOneAssetWithdrawAll):
        # Single asset withdrawal for specified LP tokens
        result, newLPTokenBalance = singleWithdrawTokens(
            sb,
            ammSle,
            ammAccountID,
            amountBalance,
            lptAMMBalance,
            amount,              # min amount or asset specifier
            lpTokensWithdraw,    # LP tokens to redeem
            tfee
        )

    elif subTxType & tfLimitLPToken:
        # Single asset withdrawal with effective price constraint
        result, newLPTokenBalance = singleWithdrawEPrice(
            sb,
            ammSle,
            ammAccountID,
            amountBalance,
            lptAMMBalance,
            amount,              # min amount
            ePrice,              # min effective price
            tfee
        )

    elif subTxType & tfSingleAsset:
        # Single asset withdrawal for specified amount
        result, newLPTokenBalance = singleWithdraw(
            sb,
            ammSle,
            ammAccountID,
            amountBalance,
            lptAMMBalance,
            amount,              # amount to withdraw
            tfee
        )

    elif subTxType & (tfLPToken | tfWithdrawAll):
        # Proportional withdrawal for LP tokens
        result, newLPTokenBalance = equalWithdrawTokens(
            sb,
            ammSle,
            ammAccountID,
            amountBalance,
            amount2Balance,
            lptAMMBalance,
            lpTokens,
            lpTokensWithdraw,    # LP tokens to redeem
            tfee
        )

    else:
        # Should not happen (validated in preflight)
        return (tecINTERNAL, False)

    if result != tesSUCCESS:
        return (result, False)

    # Delete AMM if empty, or update LP token balance
    res = deleteAMMAccountIfEmpty(
        sb,
        ammSle,
        newLPTokenBalance,
        tx[sfAsset],
        tx[sfAsset2],
        journal
    )

    if not res.second:
        return (res.first, False)

    return (tesSUCCESS, True)
```

# 3. getTradingFee

Determines the trading fee for the withdrawer, accounting for auction slot discounts. Same implementation as AMMDeposit, see [deposit.md](deposit.md#3-gettradingfee).

# 4. Multi-Asset Withdrawal Modes

Multi-asset withdrawal modes maintain proportional withdrawals by removing both pool assets simultaneously. This preserves the pool's price (asset ratio) while decreasing liquidity. Since these withdrawals maintain proportional ratios, they incur no trading fees.

The modes are:
- **[equalWithdrawTokens](#41-equalwithdrawtokens-tflptoken-tfwithdrawall) (tfLPToken, tfWithdrawAll)** - Redeem exact LP tokens or all LP tokens, receive proportional amounts of both assets
- **[equalWithdrawLimit](#42-equalwithdrawlimit-tftwoasset) (tfTwoAsset)** - Specify maximum amounts for both assets, system calculates LP tokens to burn

## 4.1. equalWithdrawTokens (tfLPToken, tfWithdrawAll)

> "I want to redeem exactly X LP tokens, how much of both assets do I get?" (tfLPToken)
> "Redeem all my LP tokens for both assets." (tfWithdrawAll)

Proportional withdrawal of pool assets for the amount of LP tokens.[^equalWithdrawTokens]

[^equalWithdrawTokens]: AMMWithdraw::equalWithdrawTokens: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L790-L871)

This function handles two related modes. With `tfLPToken`, the user specifies the exact number of LP tokens to redeem using `LPTokenIn`, and the function calculates the proportional amounts of both assets to withdraw. With `tfWithdrawAll`, the user redeems their entire LP token balance without specifying an amount. The function handles a special case when withdrawing all LP tokens from the pool (`lpTokensWithdraw == lptAMMBalance`), which empties the pool completely. For partial withdrawals, it calculates the pool fraction (`frac = tokensAdj / lptAMMBalance`), then multiplies each asset balance by this fraction, rounding down with [`getRoundedAsset`](helpers.md#23-getroundedasset) to ensure the pool retains sufficient assets.

**Example:**

A pool has 150 USD, 150 EUR, and 150 LP tokens outstanding. Bob holds 30 LP tokens (20% of the pool).

**Case 1: Bob redeems exactly 15 LP tokens (tfLPToken)**
- LP tokens to redeem: 15
- Fraction of pool: 15 / 150 = 0.1 (10%)
- USD withdrawn: 150 * 0.1 = 15 USD
- EUR withdrawn: 150 * 0.1 = 15 EUR
- Result: Bob redeems 15 LP tokens, receives 15 USD + 15 EUR
- Bob now holds: 15 LP tokens (10% of remaining pool)

**Case 2: Bob redeems all his LP tokens (tfWithdrawAll)**
- LP tokens to redeem: 30 (all Bob's holdings)
- Fraction of pool: 30 / 150 = 0.2 (20%)
- USD withdrawn: 150 * 0.2 = 30 USD
- EUR withdrawn: 150 * 0.2 = 30 EUR
- Result: Bob redeems 30 LP tokens, receives 30 USD + 30 EUR
- Bob now holds: 0 LP tokens

### 4.1.1 equalWithdrawTokens Pseudo-Code

```python
def equalWithdrawTokens(
        view,                 # Ledger view (sandbox)
        ammSle,               # AMM ledger entry
        account,              # Withdrawer account ID
        ammAccount,           # AMM pseudo-account ID
        amountBalance,        # Current pool balance of asset1
        amount2Balance,       # Current pool balance of asset2
        lptAMMBalance,        # Total outstanding LP tokens
        lpTokens,             # Withdrawer's LP token balance
        lpTokensWithdraw,     # LP tokens to redeem
        tfee,                 # Trading fee (not used for proportional)
        freezeHandling,       # How to handle frozen assets
        withdrawAll,          # Whether this is tfWithdrawAll
        priorBalance,         # Withdrawer's prior XRP balance
        journal):             # Debug journal
    # CASE 1: Withdrawing all LP tokens in the pool
    if lpTokensWithdraw == lptAMMBalance:
        # Withdraw all assets, empty the pool
        return withdraw(
            view,
            ammSle,
            ammAccount,
            account,
            amountBalance,
            amountBalance,       # withdraw all of asset1
            amount2Balance,      # withdraw all of asset2
            lptAMMBalance,
            lpTokensWithdraw,
            tfee,
            freezeHandling,
            WithdrawAll=True,    # special handling for complete withdrawal
            priorBalance,
            journal
        )

    # CASE 2: Partial withdrawal
    # Adjust LP tokens for precision (with fixAMMv1_3)
    tokensAdj = adjustLPTokensIn(rules, lptAMMBalance, lpTokensWithdraw, withdrawAll) # helpers.md#25-adjustlptokensin-withdrawals

    if rules.enabled(fixAMMv1_3) and tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, STAmount{}, STAmount{}, None)

    # Calculate the fraction of the pool being withdrawn
    # Example: 5,000 LP tokens / 50,000 total = 0.1 (10% of pool)
    frac = tokensAdj / lptAMMBalance

    # Calculate withdrawal amounts for both assets
    # With fixAMMv1_3: Round DOWN (conservative, ensures pool keeps enough)
    amountWithdraw = getRoundedAsset(rules, amountBalance, frac, IsDeposit=False) # helpers.md#231-getroundedasset-simple-pseudo-code
    amount2Withdraw = getRoundedAsset(rules, amount2Balance, frac, IsDeposit=False) # helpers.md#231-getroundedasset-simple-pseudo-code

    # Prevent one-sided pool withdrawal due to rounding
    # If either amount rounds to zero, fail so user withdraws more tokens
    if amountWithdraw == 0 or amount2Withdraw == 0:
        return (tecAMM_FAILED, STAmount{}, STAmount{}, STAmount{})

    return withdraw(
        view,
        ammSle,
        ammAccount,
        account,
        amountBalance,
        amountWithdraw,
        amount2Withdraw,
        lptAMMBalance,
        tokensAdj,
        tfee,
        freezeHandling,
        withdrawAll,
        priorBalance,
        journal
    )
```

## 4.2. equalWithdrawLimit (tfTwoAsset)

> "I want to withdraw up to X and Y, how many LP tokens do I burn?"

The user specifies maximum amounts they want to withdraw for both assets (`Amount` and `Amount2`).[^equalWithdrawLimit] Since the withdrawal must maintain the pool's ratio, the function cannot simply use both maximum amounts - one will typically be limiting while the other has excess.

[^equalWithdrawLimit]: AMMWithdraw::equalWithdrawLimit: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L899-L961)

The function tries two strategies to maximize the withdrawal within the user's constraints. First, it attempts to withdraw all of `Amount` by calculating the pool fraction this represents (`frac = Amount / amountBalance`), converting this to LP tokens with proper rounding, then recalculating the fraction from the rounded LP tokens (`frac = adjustFracByTokens(...)`) to ensure precision consistency. Using this adjusted fraction, it calculates the proportional amount2 needed. If this amount2 fits within `Amount2`, the withdrawal proceeds immediately.

Only if the first strategy fails does it try the second strategy: starting with all of `Amount2`, going through the same fraction -> LP tokens -> adjusted fraction → amount1 calculation. If the calculated amount1 exceeds `Amount`, the entire withdrawal fails.

**Example:**

There is an AMM with 100 USD and 100 EUR (1:1 ratio), 100 LP tokens outstanding. Bob holds 50 LP tokens and wants to make a proportional withdrawal.

**Case 1: Bob tries to withdraw up to 30 USD + 20 EUR**
- Strategy 1: Use all 30 USD and needs 30 EUR to maintain 1:1 ratio - FAILS (only wants 20 EUR)
- Strategy 2: Use all 20 EUR and needs 20 USD to maintain 1:1 ratio - SUCCESS (wants up to 30 USD)
- Result: Withdraws 20 USD + 20 EUR, redeems 20 LP tokens

**Case 2: Bob tries to withdraw up to 20 USD + 30 EUR**
- Strategy 1: Use all 20 USD and needs 20 EUR to maintain 1:1 ratio - SUCCESS (wants up to 30 EUR)
- Result: Withdraws 20 USD + 20 EUR, redeems 20 LP tokens

### 4.2.1 equalWithdrawLimit Pseudo-Code

```python
def equalWithdrawLimit(
        view,                 # Ledger view (sandbox)
        ammSle,               # AMM ledger entry
        ammAccount,           # AMM pseudo-account ID
        amountBalance,        # Current pool balance of asset1
        amount2Balance,       # Current pool balance of asset2
        lptAMMBalance,        # Total outstanding LP tokens
        amount,               # User's max amount1 to withdraw
        amount2,              # User's max amount2 to withdraw
        tfee):                # Trading fee (not used for proportional)
    # STRATEGY 1: Try using all of amount (asset1)
    # Calculate what fraction of the pool this represents
    frac = amount / amountBalance

    # Calculate LP tokens for this fraction
    # Using simple version of getRoundedLPTokens (direct fraction)
    tokensAdj = getRoundedLPTokens(rules, lptAMMBalance, frac, IsDeposit=False) # helpers.md#211-getroundedlptokens-simple-pseudo-code

    if rules.enabled(fixAMMv1_3) and tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, STAmount{})

    # Adjust fraction based on rounded tokens (for precision consistency)
    frac = adjustFracByTokens(rules, lptAMMBalance, tokensAdj, frac)

    # Calculate how much asset2 would be withdrawn for this fraction
    # Using simple version of getRoundedAsset (direct fraction)
    amount2Withdraw = getRoundedAsset(rules, amount2Balance, frac, IsDeposit=False) # helpers.md#231-getroundedasset-simple-pseudo-code

    # Check if calculated amount2 fits within user's max constraint
    if amount2Withdraw <= amount2:
        # Success! Use all of amount, calculated amount2Withdraw
        return withdraw(
            view,
            ammSle,
            ammAccount,
            amountBalance,
            amount,              # withdraw all of asset1
            amount2Withdraw,     # calculated asset2
            lptAMMBalance,
            tokensAdj,
            tfee
        )

    # STRATEGY 2: Strategy 1 failed, try using all of amount2
    # Calculate what fraction of the pool amount2 represents
    frac = amount2 / amount2Balance

    # Calculate how much asset1 would be withdrawn for this fraction (preliminary)
    # Using simple version of getRoundedAsset (direct fraction)
    amountWithdraw = getRoundedAsset(rules, amountBalance, frac, IsDeposit=False) # helpers.md#231-getroundedasset-simple-pseudo-code

    # Calculate LP tokens for this fraction
    # Using simple version of getRoundedLPTokens (direct fraction)
    tokensAdj = getRoundedLPTokens(rules, lptAMMBalance, frac, IsDeposit=False) # helpers.md#211-getroundedlptokens-simple-pseudo-code

    if rules.enabled(fixAMMv1_3) and tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, STAmount{})

    # Adjust fraction based on rounded tokens (for precision consistency)
    frac = adjustFracByTokens(rules, lptAMMBalance, tokensAdj, frac)

    # Recalculate asset1 amount with adjusted fraction
    # Using simple version of getRoundedAsset (direct fraction)
    amountWithdraw = getRoundedAsset(rules, amountBalance, frac, IsDeposit=False) # helpers.md#231-getroundedasset-simple-pseudo-code

    # Check if calculated amount fits within user's max constraint
    if rules.enabled(fixAMMv1_3):
        if amountWithdraw > amount:
            return (tecAMM_FAILED, STAmount{})

    # Success! Use calculated amountWithdraw, all of amount2
    return withdraw(
        view,
        ammSle,
        ammAccount,
        amountBalance,
        amountWithdraw,      # calculated asset1
        amount2,             # withdraw all of asset2
        lptAMMBalance,
        tokensAdj,
        tfee
    )
```

# 5. Single-Asset Withdrawal Modes

Single-asset withdrawal modes allow users to withdraw only one asset instead of both assets proportionally. Unlike [multi-asset withdrawals](#4-multi-asset-withdrawal-modes) that maintain the pool ratio, single-asset withdrawals change the pool composition. Because they alter the pool ratio, [trading fees](#3-gettradingfee) apply to single-asset withdrawals. There are three modes:

- **[singleWithdraw](#51-singlewithdraw-tfsingleasset) (tfSingleAsset)** - User specifies withdrawal amount, system calculates LP tokens to redeem
- **[singleWithdrawTokens](#52-singlewithdrawtokens-tfoneassetlptoken-tfoneassetwithdrawall) (tfOneAssetLPToken, tfOneAssetWithdrawAll)** - User specifies exact LP tokens to redeem or all LP tokens, withdraws calculated amount of single asset
- **[singleWithdrawEPrice](#53-singlewithdraweprice-tflimitlptoken) (tfLimitLPToken)** - User specifies minimum effective price limit

## 5.1. singleWithdraw (tfSingleAsset)

> "I want to withdraw X amount, how many LP tokens must I redeem?"

The user specifies `Amount` (the asset amount to withdraw) and the function calculates how many LP tokens must be redeemed.[^singleWithdraw] Since this is a single-asset withdrawal that changes the pool ratio, a [trading fee](#3-gettradingfee) applies. The function uses [`lpTokensIn`](helpers.md#331-lptokensin-equation-7) (Equation 7) to calculate the LP tokens based on the withdrawal amount and trading fee, then adjusts the result for precision with [`adjustLPTokensIn`](helpers.md#25-adjustlptokensin-withdrawals). The adjusted tokens are passed to `adjustAssetOutByTokens` to recalculate the withdrawal amount, ensuring the reverse calculation produces consistent results and doesn't underpay the user due to rounding.

[^singleWithdraw]: AMMWithdraw::singleWithdraw: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L969-L1007)

### 5.1.1. singleWithdraw Pseudo-Code

```python
def singleWithdraw(
        view,              # Ledger view (sandbox)
        ammSle,            # AMM ledger entry
        ammAccount,        # AMM pseudo-account ID
        amountBalance,     # Current pool balance of the asset
        lptAMMBalance,     # Total outstanding LP tokens
        amount,            # Amount to withdraw
        tfee):             # Trading fee (for single-asset withdrawal)
    # Calculate LP tokens using the single-asset withdrawal formula
    # lpTokensIn solves: "How many LP tokens to redeem to get `amount` assets?"
    tokens = lpTokensIn(amountBalance, amount, lptAMMBalance, tfee) # helpers.md#331-lptokensin-equation-7

    # Adjust LP tokens for precision (with fixAMMv1_3)
    tokensAdj = adjustLPTokensIn(rules, lptAMMBalance, tokens, isWithdrawAll(tx)) # helpers.md#25-adjustlptokensin-withdrawals

    if tokensAdj == 0:
        if not rules.enabled(fixAMMv1_3):
            return (tecAMM_FAILED, STAmount{})
        else:
            return (tecAMM_INVALID_TOKENS, STAmount{})

    # Adjust withdrawal amount based on adjusted tokens
    # This ensures the reverse calculation produces consistent results
    tokensAdj, amountWithdrawAdj = adjustAssetOutByTokens(
        rules, amountBalance, amount, lptAMMBalance, tokensAdj, tfee)

    if rules.enabled(fixAMMv1_3) and tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, STAmount{})

    return withdraw(
        view,
        ammSle,
        ammAccount,
        amountBalance,
        amountWithdrawAdj,   # adjusted: actual amount to withdraw
        None,                # single-asset withdrawal (no asset2)
        lptAMMBalance,
        tokensAdj,           # calculated: LP tokens to redeem
        tfee
    )
```

## 5.2. singleWithdrawTokens (tfOneAssetLPToken, tfOneAssetWithdrawAll)

> "I'll redeem exactly X LP tokens, how much asset (single asset only) do I get?" (tfOneAssetLPToken)
> "Redeem all my LP tokens for a single asset." (tfOneAssetWithdrawAll)

Withdraw a single asset by redeeming specified LP tokens.[^singleWithdrawTokens]

[^singleWithdrawTokens]: AMMWithdraw::singleWithdrawTokens: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L1020-L1051)

This function handles the reverse calculation from [`singleWithdraw`](#51-singlewithdraw-tfsingleasset): the user specifies the exact number of LP tokens to redeem (using `LPTokenIn` for tfOneAssetLPToken, or all LP tokens for tfOneAssetWithdrawAll), and the function calculates the withdrawal amount of a single asset. The user can provide `Amount` as a minimum constraint on how much they expect to receive.

The function first adjusts the LP tokens for precision using [`adjustLPTokensIn`](helpers.md#25-adjustlptokensin-withdrawals), then uses [`ammAssetOut`](helpers.md#332-ammassetout-equation-8) (Equation 8) to calculate the withdrawal amount by solving the inverse single-asset withdrawal problem. If the calculated amount is less than the user's `Amount` constraint (when non-zero), the transaction fails with `tecAMM_FAILED`.

### 5.2.1. singleWithdrawTokens Pseudo-Code

```python
def singleWithdrawTokens(
        view,              # Ledger view (sandbox)
        ammSle,            # AMM ledger entry
        ammAccount,        # AMM pseudo-account ID
        amountBalance,     # Current pool balance of the asset
        lptAMMBalance,     # Total outstanding LP tokens
        amount,            # Min asset to receive (or 0 for no min, or asset specifier)
        lpTokensWithdraw,  # LP tokens to redeem
        tfee):             # Trading fee (for single-asset withdrawal)
    # Adjust LP tokens for precision (with fixAMMv1_3)
    tokensAdj = adjustLPTokensIn(rules, lptAMMBalance, lpTokensWithdraw, isWithdrawAll(tx)) # helpers.md#25-adjustlptokensin-withdrawals

    if rules.enabled(fixAMMv1_3) and tokensAdj == 0:
        return (tecAMM_INVALID_TOKENS, STAmount{})

    # Calculate withdrawal amount using ammAssetOut formula
    amountWithdraw = ammAssetOut(amountBalance, lptAMMBalance, tokensAdj, tfee) # helpers.md#332-ammassetout-equation-8

    # Check if calculated amount meets user's minimum (if specified)
    if amount == 0 or amountWithdraw >= amount:
        # Either no minimum specified, or calculated amount meets minimum
        return withdraw(
            view,
            ammSle,
            ammAccount,
            amountBalance,
            amountWithdraw,      # calculated: amount to withdraw
            None,                # single-asset withdrawal
            lptAMMBalance,
            tokensAdj,           # exact: LP tokens to redeem
            tfee
        )

    # Calculated amount is less than user's minimum
    return (tecAMM_FAILED, STAmount{})
```

## 5.3. singleWithdrawEPrice (tfLimitLPToken)

> "I'll withdraw (single asset), but only if the effective price per LP token is reasonable."

Withdraw a single asset with an effective price constraint.[^singleWithdrawEPrice]

This mode allows users to control the effective price when redeeming LP tokens, where effective price is defined as the ratio of LP tokens redeemed to asset withdrawn. The user provides `EPrice` (maximum effective price) and optionally `Amount` (minimum withdrawal amount). As with deposits, `EPrice` is an upper bound: the trade is sized so the effective price does not exceed `EPrice`. A lower effective price means a better deal for the withdrawer (fewer LP tokens per asset withdrawn).

The function solves a derived formula from Equation 8 to calculate the LP tokens that achieve exactly the specified effective price. It then calculates the withdrawal amount as `tokensAdj / ePrice`. If the calculated amount is less than the user's optional `Amount` constraint, the transaction fails with `tecAMM_FAILED`. Under the `fixCleanup3_3_0` amendment, a denominator (`T*f - B*E`) of exactly zero also fails with `tecAMM_FAILED`. Without the amendment that division throws and the transaction fails with `tefEXCEPTION`.

[^singleWithdrawEPrice]: AMMWithdraw::singleWithdrawEPrice: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L1119-L1179)

### 5.3.1. singleWithdrawEPrice Pseudo-Code

```python
def singleWithdrawEPrice(
        view,              # Ledger view (sandbox)
        ammSle,            # AMM ledger entry
        ammAccount,        # AMM pseudo-account ID
        amountBalance,     # Current pool balance of the asset
        lptAMMBalance,     # Total outstanding LP tokens
        amount,            # Min asset to receive (or 0 for no min)
        ePrice,            # Min effective price (LPTokenIn / AssetOut)
        tfee):             # Trading fee (for single-asset withdrawal)
    # Calculate intermediate value: B * E (balance * effective price)
    ae = amountBalance * ePrice

    # Get fee multiplier
    f = getFee(tfee)  # fee in units of 1/100,000 (e.g., 30 -> 0.0003)

    # Calculate LP tokens using derived formula
    # t = T*(T + B*E*(f-2)) / (T*f - B*E)
    tokNoRoundCb = lambda: (
        lptAMMBalance * (lptAMMBalance + ae * (f - 2)) / (lptAMMBalance * f - ae)
    )
    tokProdCb = lambda: (
        (lptAMMBalance + ae * (f - 2)) / (lptAMMBalance * f - ae)
    )

    tokensAdj = getRoundedLPTokens(
        rules, tokNoRoundCb, lptAMMBalance, tokProdCb, IsDeposit=False) # helpers.md#212-getroundedlptokens-callback-pseudo-code

    if tokensAdj <= 0:
        if not rules.enabled(fixAMMv1_3):
            return (tecAMM_FAILED, STAmount{})
        else:
            return (tecAMM_INVALID_TOKENS, STAmount{})

    # Calculate withdrawal amount from tokens and effective price
    # amountWithdraw = tokensAdj / ePrice
    amtNoRoundCb = lambda: tokensAdj / ePrice
    amtProdCb = lambda: tokensAdj / ePrice

    amountWithdraw = getRoundedAsset(
        rules, amtNoRoundCb, amount, amtProdCb, IsDeposit=False) # helpers.md#232-getroundedasset-callback-pseudo-code

    # Check if calculated amount meets user's minimum (if specified)
    if amount == 0 or amountWithdraw >= amount:
        return withdraw(
            view,
            ammSle,
            ammAccount,
            amountBalance,
            amountWithdraw,      # calculated: amount to withdraw
            None,                # single-asset withdrawal
            lptAMMBalance,
            tokensAdj,           # calculated: LP tokens to redeem
            tfee
        )

    # Calculated amount is less than user's minimum
    return (tecAMM_FAILED, STAmount{})
```

# 6. Common Withdraw Function

The `withdraw()` function[^withdraw] serves as the final common pathway for all withdrawal modes, executing the actual asset transfers after mode-specific handlers determine the withdrawal amounts.

Under `fixCleanup3_3_0` together with `fixAMMv1_3`, the common path also runs the pool product check described in [Precision and Rounding](helpers.md#2-precision-and-rounding). See the [failure conditions](README.md#332-failure-conditions) for the resulting `tecPRECISION_LOSS`.

[^withdraw]: AMMWithdraw::withdraw: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L479-L749)

This function orchestrates a sequenced validation and execution flow. It begins by verifying the withdrawer holds sufficient LP tokens to redeem, then enforces pool integrity constraints that prevent malformed states. 

The function prohibits one-sided pool withdrawals - situations where all of one asset would be withdrawn while the other remains. When a withdrawal would redeem all outstanding LP tokens, the function mandates that all pool assets must also be withdrawn, preventing orphaned assets in an empty pool.

Before executing transfers, the function checks whether the withdrawer has adequate XRP reserves if new trust lines or MPTokens need creation. For MPTs, this includes verifying proper authorization from the issuer. The function then transfers each withdrawn asset from the AMM account to the withdrawer, waiving transfer fees as AMM operations are privileged. Finally, it burns the redeemed LP tokens by calling `redeemIOU`, which reduces both the withdrawer's LP token balance and the total outstanding token supply.

## 6.1. withdraw Pseudo-Code

```python
def withdraw(
        view,              # Ledger view (sandbox)
        ammSle,            # AMM ledger entry
        ammAccount,        # AMM pseudo-account ID
        account,           # Withdrawer account ID
        amountBalance,     # Current pool balance of asset1
        amountWithdraw,    # Amount1 to withdraw
        amount2Withdraw,   # Optional: amount2 to withdraw
        lpTokensAMMBalance,  # Total outstanding LP tokens
        lpTokensWithdraw,  # LP tokens to redeem
        tfee,              # Trading fee
        freezeHandling,    # How to handle frozen assets
        authHandling,      # How to handle unauthorized MPT holders
        withdrawAll,       # Whether this is a complete withdrawal
        priorBalance:      # Withdrawer's prior XRP balance
    # Get withdrawer's current LP token balance
    lpTokens = ammLPHolds(view, ammSle, account, journal)

    # Get current pool balances (accounting for freezes and authorization)
    currentBalances = ammHolds(view, ammSle, amountWithdraw.issue, None, freezeHandling, authHandling)
    if not currentBalances:
        return (currentBalances.error(), STAmount{}, STAmount{}, STAmount{})

    curBalance, curBalance2, _ = currentBalances

    # Adjust amounts for precision (with fixAMMv1_3, this shouldn't be needed as we have already adjusted and rounded all numbers properly)
    # When withdrawing all, skip adjustment and use exact values
    if withdrawAll == No:
        amountWithdrawActual, amount2WithdrawActual, lpTokensWithdrawActual = \
            adjustAmountsByLPTokens(
                amountBalance,
                amountWithdraw,
                amount2Withdraw,
                lpTokensAMMBalance,
                lpTokensWithdraw,
                tfee,
                IsDeposit=False
            )
    else:
        amountWithdrawActual = amountWithdraw
        amount2WithdrawActual = amount2Withdraw
        lpTokensWithdrawActual = lpTokensWithdraw

    # Validate LP tokens
    if lpTokensWithdrawActual <= 0 or lpTokensWithdrawActual > lpTokens:
        return (tecAMM_INVALID_TOKENS, STAmount{}, STAmount{}, STAmount{})

    # With fixAMMv1_1: Additional validation
    if rules.enabled(fixAMMv1_1) and lpTokensWithdrawActual > lpTokensAMMBalance:
        return (tecINTERNAL, STAmount{}, STAmount{}, STAmount{})

    # Prevent one-sided pool withdrawal
    # If withdrawing all of one asset but not the other, fail
    # This ensures pools are always balanced (or completely empty)
    if (amountWithdrawActual == curBalance and amount2WithdrawActual != curBalance2) or \
       (amount2WithdrawActual == curBalance2 and amountWithdrawActual != curBalance):
        return (tecAMM_BALANCE, STAmount{}, STAmount{}, STAmount{})

    # If redeeming all LP tokens, must withdraw all assets
    # This prevents situations where LP tokens are zero but assets remain
    if lpTokensWithdrawActual == lpTokensAMMBalance and \
       (amountWithdrawActual != curBalance or amount2WithdrawActual != curBalance2):
        return (tecAMM_BALANCE, STAmount{}, STAmount{}, STAmount{})

    # Check withdrawal doesn't exceed pool balance
    if amountWithdrawActual > curBalance or amount2WithdrawActual > curBalance2:
        return (tecAMM_BALANCE, STAmount{}, STAmount{}, STAmount{})

    # With featureMPTokensV2: the post-withdrawal pool state must be consistent
    # (all balances zero or all non-zero, agreeing with the LP token total)
    if rules.enabled(featureMPTokensV2):
        newBalanceZero = (curBalance - amountWithdrawActual) == 0
        newBalance2Zero = (curBalance2 - amount2WithdrawActual) == 0
        newLPTokensZero = (lpTokensAMMBalance - lpTokensWithdrawActual) == 0
        if amount2WithdrawActual is None:
            valid = (newBalanceZero == newLPTokensZero)
        else:
            valid = (newBalanceZero == newBalance2Zero and newBalance2Zero == newLPTokensZero)
        if not valid:
            return (tecAMM_BALANCE, STAmount{}, STAmount{}, STAmount{})

    # Helper function to check reserve requirements (with fixAMMv1_2)
    # Checks if withdrawer has sufficient XRP reserve for trust line or MPToken creation
    def sufficientReserve(asset):
        if not rules.enabled(fixAMMv1_2) or isXRP(asset):
            return (tesSUCCESS, None)

        # Check if trust line (for IOUs) or MPToken (for MPTs) exists
        if isIOU(asset):
            assetExists = view.exists(keylet.line(account, asset.issue))
            mptokenKey = None
        else:  # MPT
            issuanceKey = keylet.mptIssuance(asset.mptID)
            mptokenKey = keylet.mptoken(issuanceKey, account)
            assetExists = view.exists(mptokenKey)
            if assetExists:
                mptokenKey = None  # Already exists, no need to create

        if not assetExists:
            sleAccount = view.peek(keylet.account(account))
            if not sleAccount:
                return (tecINTERNAL, None)

            balance = sleAccount[sfBalance].xrp
            ownerCount = sleAccount[sfOwnerCount]

            reserve = view.fees().accountReserve(ownerCount + 1) if ownerCount >= 2 else 0

            # For IOUs: use max of prior and current balance
            # For MPTs: use prior balance only
            balanceToCheck = max(priorBalance, balance) if isIOU(asset) else priorBalance

            if balanceToCheck < reserve:
                return (tecINSUFFICIENT_RESERVE, None)

        return (tesSUCCESS, mptokenKey)

    # Helper function to create MPToken if needed
    def createMPToken(asset, mptokenKey):
        if mptokenKey and account != asset.getIssuer():
            # Must authorize MPToken
            if requireAuth(view, asset.mptIssue, account, WeakAuth) != tesSUCCESS:
                return err

            if checkCreateMPT(view, asset.mptIssue, account, journal) != tesSUCCESS:
                return err

        return tesSUCCESS

    # Check reserve and create MPToken for asset1
    result, mptokenKey = sufficientReserve(amountWithdrawActual.asset)
    if result != tesSUCCESS:
        return (result, STAmount{}, STAmount{}, STAmount{})

    result = createMPToken(amountWithdrawActual.asset, mptokenKey)
    if result != tesSUCCESS:
        return (result, STAmount{}, STAmount{}, STAmount{})

    # Transfer asset1 from AMM to withdrawer
    result = accountSend(
        view,
        ammAccount,           # from: AMM pseudo-account
        account,              # to: withdrawer
        amountWithdrawActual, # amount
        WaiveTransferFee=Yes  # AMM withdrawals don't pay transfer fees
    )
    if result != tesSUCCESS:
        return (result, STAmount{}, STAmount{}, STAmount{})

    # If two-asset withdrawal, check reserve, create MPToken, and transfer asset2
    if amount2WithdrawActual:
        # Check reserve and create MPToken for asset2
        result, mptokenKey = sufficientReserve(amount2WithdrawActual.asset)
        if result != tesSUCCESS:
            return (result, STAmount{}, STAmount{}, STAmount{})

        result = createMPToken(amount2WithdrawActual.asset, mptokenKey)
        if result != tesSUCCESS:
            return (result, STAmount{}, STAmount{}, STAmount{})

        result = accountSend(
            view,
            ammAccount,
            account,
            amount2WithdrawActual,
            WaiveTransferFee=Yes
        )
        if result != tesSUCCESS:
            return (result, STAmount{}, STAmount{}, STAmount{})

    # Redeem (burn) LP tokens
    # This decreases the trust line balance and may delete the trust line
    result = redeemIOU(
        view,
        account,
        lpTokensWithdrawActual,
        lpTokensWithdrawActual.issue,
        journal
    )
    if result != tesSUCCESS:
        return (result, STAmount{}, STAmount{}, STAmount{})

    # Return success with new LP token balance and actual withdrawal amounts
    return (
        tesSUCCESS,
        lpTokensAMMBalance - lpTokensWithdrawActual,
        amountWithdrawActual,
        amount2WithdrawActual
    )
```
