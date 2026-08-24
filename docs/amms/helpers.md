# Index

- [1. Introduction](#1-introduction)
- [2. Precision and Rounding](#2-precision-and-rounding)
  - [2.1. getRoundedLPTokens](#21-getroundedlptokens)
    - [2.1.1. getRoundedLPTokens Pseudo-Code](#211-getroundedlptokens-simple-pseudo-code)
  - [2.2. adjustLPTokens](#22-adjustlptokens)
    - [2.2.1. adjustLPTokens Pseudo-Code](#221-adjustlptokens-pseudo-code)
  - [2.3. getRoundedAsset](#23-getroundedasset)
    - [2.3.1. getRoundedAsset Pseudo-Code](#231-getroundedasset-simple-pseudo-code)
  - [2.4. adjustLPTokensOut (deposits)](#24-adjustlptokensout-deposits)
    - [2.4.1. adjustLPTokensOut Pseudo-Code](#241-adjustlptokensout-pseudo-code)
  - [2.5. adjustLPTokensIn (withdrawals)](#25-adjustlptokensin-withdrawals)
    - [2.5.1. adjustLPTokensIn Pseudo-Code](#251-adjustlptokensin-pseudo-code)
  - [2.6. adjustAssetInByTokens](#26-adjustassetinbytokens)
    - [2.6.1. adjustAssetInByTokens Pseudo-Code](#261-adjustassetinbytokens-pseudo-code)
- [3. AMM Formulas](#3-amm-formulas)
  - [3.1. Swap Formulas](#31-swap-formulas)
    - [3.1.1. swapAssetIn](#311-swapassetin)
      - [3.1.1.1. swapAssetIn Pseudo-Code](#3111-swapassetin-pseudo-code)
    - [3.1.2. swapAssetOut](#312-swapassetout)
      - [3.1.2.1. swapAssetOut Pseudo-Code](#3121-swapassetout-pseudo-code)
    - [3.1.3. Slippage and Quality Degradation](#313-slippage-and-quality-degradation)
    - [3.1.4. changeSpotPriceQuality](#314-changespotpricequality)
  - [3.2. Deposit Formulas](#32-deposit-formulas)
    - [3.2.1. lpTokensOut (Equation 3)](#321-lptokensout-equation-3)
      - [3.2.1.1. lpTokensOut Pseudo-Code](#3211-lptokensout-pseudo-code)
    - [3.2.2. ammAssetIn (Equation 4)](#322-ammassetin-equation-4)
      - [3.2.2.1. ammAssetIn Pseudo-Code](#3221-ammassetin-pseudo-code)
  - [3.3. Withdrawal Formulas](#33-withdrawal-formulas)
    - [3.3.1. lpTokensIn (Equation 7)](#331-lptokensin-equation-7)
      - [3.3.1.1. lpTokensIn Pseudo-Code](#3311-lptokensin-pseudo-code)
    - [3.3.2. ammAssetOut (Equation 8)](#332-ammassetout-equation-8)
      - [3.3.2.1. ammAssetOut Pseudo-Code](#3321-ammassetout-pseudo-code)

# 1. Introduction

This document describes helper functions used throughout the AMM implementation in `xrpld`. These functions are called by the transaction handlers documented in [deposit.md](deposit.md) and [withdraw.md](withdraw.md), as well as by the payment path finding code, specifically [BookStep](../flow/steps.md#5-bookstep) which integrates [AMM liquidity](../flow/steps.md#54-amm-integration) into the Payment Engine.

The functions fall into two categories: [precision and rounding](#2-precision-and-rounding) functions that handle decimal precision to prevent value leakage, and [AMM formulas](#3-amm-formulas) that implement the mathematical equations for calculating LP tokens and asset amounts based on the weighted geometric mean market maker model.

**Terminology**: 

Throughout the document we refer to equations by their number in the [XLS-30 specification](https://github.com/XRPLF/XRPL-Standards/tree/master/XLS-0030-automated-market-maker).
AMM helpers use **token** as a subject in many function names. This refers to any supported [currency](../glossary.md#currency) in the system, not only a particular token implementation, like trust lines or MPTs.

# 2. Precision and Rounding

Functions for handling precision and rounding with the [fixAMMv1_3](https://xrpl.org/resources/known-amendments#fixammv1_3) amendment. These functions ensure that floating-point calculations do not introduce precision errors that could be exploited or cause inconsistencies.

The `checkAMMPrecisionLoss` helper (requires `fixCleanup3_3_0` together with `fixAMMv1_3`) verifies the pool product invariant, comparing `sqrt(asset1 * asset2)` against the new LP token balance. A failed check returns `tecPRECISION_LOSS`.[^check-precision-loss]

[^check-precision-loss]: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L436-L471)

## 2.1. getRoundedLPTokens

Calculate LP tokens with proper rounding and precision adjustment.

This function is used throughout the AMM implementation when we need to calculate LP token amounts based on a fraction of the pool. It is called by both deposit and withdrawal operations to ensure that the LP tokens issued or redeemed are calculated with proper rounding to protect the pool from precision exploits. 

Before the `fixAMMv1_3` amendment, simple multiplication could lead to precision issues where adding small amounts to large balances would lose precision. The directional rounding - down for deposits, up for withdrawals - ensures the pool is always protected: depositors get slightly fewer LP tokens, and withdrawers redeem slightly more LP tokens, both favoring the pool.

There are two overloaded versions:
1. **Simple version**: Takes a direct fraction value[^get-rounded-lp-tokens-simple] - used by [`equalDepositLimit`](deposit.md#411-equaldepositlimit-pseudo-code), [`equalWithdrawLimit`](withdraw.md#421-equalwithdrawlimit-pseudo-code)
2. **Callback version**: Takes lambda callbacks for delayed evaluation[^get-rounded-lp-tokens-callback] - used by [`singleDepositEPrice`](deposit.md#531-singledepositeprice-pseudo-code), [`singleWithdrawEPrice`](withdraw.md#531-singlewithdraweprice-pseudo-code)

[^get-rounded-lp-tokens-simple]: Simple version of `getRoundedLPTokens`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L319-L331)
[^get-rounded-lp-tokens-callback]: Callback version of `getRoundedLPTokens`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L334-L354)

The callback version exists to avoid redundant calculations when the input depends on complex intermediate calculations. The lambdas delay evaluation so that:
- If `fixAMMv1_3` is disabled, only `noRoundCb()` is called
- If `fixAMMv1_3` is enabled, only `productCb()` is called
This prevents computing expensive formulas twice. For simple proportional operations, the simple version is more straightforward.

The `Number` class uses thread-local global state for its rounding mode[^number-thread-local-mode]. When `NumberRoundModeGuard` sets the rounding mode, all subsequent `Number` arithmetic operations automatically use that mode. The callbacks perform operations like `amountDeposit / ePrice` where both operands are `STAmount` values that implicitly convert to `Number`[^stamount-to-number-conversion], so the division respects the global rounding mode without needing to pass the mode as a parameter.

[^number-thread-local-mode]: Thread-local rounding mode in `Number` class: [`Number.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/basics/Number.h#L545)
[^stamount-to-number-conversion]: `STAmount` to `Number` conversion operator: [`STAmount.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STAmount.h#L524-L533)

### 2.1.1. getRoundedLPTokens (Simple) Pseudo-Code

```python
def getRoundedLPTokens(
        rules, # Ledger rules 
        balance, # LP token balance
        frac, # Fraction of the pool being deposited/withdrawan
        isDeposit # True for deposits, False for withdrawals
    ):
    # Before fixAMMv1_3: Simple multiplication, no special rounding
    if not rules.enabled(fixAMMv1_3):
        return toSTAmount(balance.asset, balance * frac)

    tokens = multiply(balance, frac, 'downward' if isDeposit else 'upward')
    return adjustLPTokens(balance, tokens, isDeposit)
```

### 2.1.2. getRoundedLPTokens (Callback) Pseudo-Code

```python
def getRoundedLPTokens(
        rules, # Ledger rules (to check if fixAMMv1_3 is enabled)
        noRoundCb, # Lambda that returns the tokens without fixAMMv1_3 rounding
        lptAMMBalance, # Total outstanding LP tokens
        productCb, # Lambda that returns the multiplier/fraction
        isDeposit # True for deposits, False for withdrawals
    ):
    # Before fixAMMv1_3: Use noRoundCb (no special rounding)
    if not rules.enabled(fixAMMv1_3):
        return toSTAmount(lptAMMBalance.asset, noRoundCb())

    # Rounding mode
    rm = 'downward' if isDeposit else 'upward'
    
    if isDeposit:
        # For deposits: convert productCb() result with rounding
        with NumberRoundModeGuard(rm):
            tokens = toSTAmount(lptAMMBalance.asset, productCb(), rm)
    else:
        # For withdrawals: multiply balance by productCb() result
        tokens = multiply(lptAMMBalance, productCb(), 'downward' if isDeposit else 'upward')

    # Adjust tokens to account for precision loss
    return adjustLPTokens(lptAMMBalance, tokens, isDeposit)
```

## 2.2. adjustLPTokens

Adjust LP tokens to account for precision loss when updating the balance.[^adjust-lp-tokens]

[^adjust-lp-tokens]: `adjustLPTokens`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L182-L190)

This is a low-level helper function that handles a precision issue. When adding or removing a small amount to a very large pool balance, precision is lost when using `Number` - the resulting balance may be less than the actual sum would be with infinite precision.

This function preemptively calculates what the balance will actually be after the operation completes, accounting for this precision loss. For deposits, it computes `(lptAMMBalance + lpTokens) - lptAMMBalance` which simulates adding the tokens to the balance and then calculating what the actual change was. For withdrawals, it computes `(lpTokens - lptAMMBalance) + lptAMMBalance`. By using the actual representable result, the function ensures that the tokens issued or redeemed exactly match what will be reflected in the balance after the ledger applies the operation.

### 2.2.1. adjustLPTokens Pseudo-Code

```python
def adjustLPTokens(lptAMMBalance, lpTokens, isDeposit):
    # Force downward rounding to ensure adjusted tokens <= requested tokens
    with rounding_mode('downward'):
        if isDeposit:
            return (lptAMMBalance + lpTokens) - lptAMMBalance
        else:
            return (lpTokens - lptAMMBalance) + lptAMMBalance
```

## 2.3. getRoundedAsset

Calculate asset amount with proper rounding and precision adjustment.

This function is similar to [`getRoundedLPTokens`](#21-getroundedlptokens), but operates on pool assets (XRP, USD, etc.) rather than LP tokens. The key difference is that the rounding direction is opposite: for deposits, assets round up to ensure the pool receives enough, while for withdrawals, assets round down to ensure the pool retains enough.

There are two overloaded versions:
1. **Simple version**: Takes a direct fraction value[^get-rounded-asset-simple] - used for proportional deposits/withdrawals
2. **Callback version**: Takes lambda callbacks for delayed evaluation[^get-rounded-asset-callback] - used for complex single-asset calculations

[^get-rounded-asset-simple]: Simple version of `getRoundedAsset`: [`AMMHelpers.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/ledger/helpers/AMMHelpers.h#L623-L638)
[^get-rounded-asset-callback]: Callback version of `getRoundedAsset`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L301-L316)

Please refer to [`getRoundedLPTokens`](#21-getroundedlptokens) for the reasoning and the explanation behind the callback version.

### 2.3.1. getRoundedAsset (Simple) Pseudo-Code

```python
def getRoundedAsset(
        rules, # Ledger rules (to check if fixAMMv1_3 is enabled)
        balance, # Current pool balance of the asset
        frac, # Fraction of the pool being deposited/withdrawn (Number)
        isDeposit # True for deposits, False for withdrawals
    ):
    # Before fixAMMv1_3: Simple multiplication, no special rounding
    if not rules.enabled(fixAMMv1_3):
        return toSTAmount(balance.asset, balance * frac)

    return multiply(balance, frac, "upward" if isDeposit else "downward")
```

### 2.3.2. getRoundedAsset (Callback) Pseudo-Code

```python
def getRoundedAsset(
        rules, # Ledger rules (to check if fixAMMv1_3 is enabled)
        noRoundCb, # Lambda that returns the amount without fixAMMv1_3 rounding
        balance, # Current pool balance of the asset
        productCb, # Lambda that returns the multiplier/fraction
        isDeposit # True for deposits, False for withdrawals
    ):
    # Before fixAMMv1_3: Use noRoundCb (no special rounding)
    if not rules.enabled(fixAMMv1_3):
        return toSTAmount(balance.asset, noRoundCb())

    # Rounding mode
    rm = "upward" if isDeposit else "downward"

    if isDeposit:
        # For deposits: multiply balance by productCb() result
        return multiply(balance, productCb(), rm)
    else:
        # For withdrawals: convert productCb() result directly with rounding
        with NumberRoundModeGuard(rm):
            return toSTAmount(balance.asset, productCb(), rm)
```

## 2.4. adjustLPTokensOut (deposits)

Adjust LP tokens for deposits (outgoing from AMM).[^adjust-lp-tokens-out]

[^adjust-lp-tokens-out]: `adjustLPTokensOut`: [`AMMDeposit.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L623-L631)

This wrapper function applies precision adjustments specifically for deposit operations. It calls [`adjustLPTokens`](#22-adjustlptokens) with `isDeposit=True`, ensuring downward rounding that slightly reduces the LP tokens issued to depositors.

### 2.4.1. adjustLPTokensOut Pseudo-Code

```python
def adjustLPTokensOut(rules, lptAMMBalance, lpTokensDeposit):
    if not rules.enabled(fixAMMv1_3):
        return lpTokensDeposit

    # Deposit uses IsDeposit=True, which rounds tokens DOWNWARD
    return adjustLPTokens(lptAMMBalance, lpTokensDeposit, IsDeposit=True)
```

## 2.5. adjustLPTokensIn (withdrawals)

Adjust LP tokens for withdrawals (incoming to AMM).[^adjust-lp-tokens-in]

[^adjust-lp-tokens-in]: `adjustLPTokensIn`: [`AMMWithdraw.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L712-L721)

This wrapper function applies precision adjustments specifically for withdrawal operations. It calls [`adjustLPTokens`](#22-adjustlptokens) with `isDeposit=False`.

### 2.5.1. adjustLPTokensIn Pseudo-Code

```python
def adjustLPTokensIn(rules, lptAMMBalance, lpTokensWithdraw, withdrawAll):
    if not rules.enabled(fixAMMv1_3) or withdrawAll == WithdrawAll.Yes:
        return lpTokensWithdraw

    # Withdrawal uses IsDeposit=False, which rounds tokens UPWARD
    return adjustLPTokens(lptAMMBalance, lpTokensWithdraw, IsDeposit=False)
```

## 2.6. adjustAssetInByTokens

Adjust asset deposit amount and LP tokens to handle rounding edge cases.[^adjust-asset-in-by-tokens]

[^adjust-asset-in-by-tokens]: `adjustAssetInByTokens`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L357-L380)

This function is used by single-asset deposit operations to handle a precision edge case. When a user specifies a deposit amount, the system calculates LP tokens using [`lpTokensOut`](#321-lptokensout-equation-3) (Equation 3), then adjusts those tokens with [`adjustLPTokens`](#22-adjustlptokens). However, when we reverse the calculation using [`ammAssetIn`](#322-ammassetin-equation-4) (Equation 4) to verify the deposit amount, rounding can cause the calculated amount to exceed the user's original deposit amount.

If this happens, the function reduces the deposit amount by the difference and recalculates the LP tokens with the adjusted deposit amount. This ensures the final deposit amount never exceeds what the user specified, protecting them from overpaying due to rounding artifacts.

### 2.6.1. adjustAssetInByTokens Pseudo-Code

```python
def adjustAssetInByTokens(
        rules,          # Ledger rules
        balance,        # Current pool balance of the asset
        amount,         # User's specified deposit amount
        lptAMMBalance,  # Total outstanding LP tokens
        tokens,         # Adjusted LP tokens (from adjustLPTokens)
        tfee):          # Trading fee
    """
    Assumes fixAMMv1_3 is enabled.
    """
    # Calculate what deposit amount is needed for the adjusted tokens
    assetAdj = ammAssetIn(balance, lptAMMBalance, tokens, tfee)
    tokensAdj = tokens

    # Check if reverse calculation exceeds user's amount due to rounding
    if assetAdj > amount:
        # Reduce deposit amount by the difference
        adjAmount = amount - (assetAdj - amount)

        # Recalculate LP tokens with the reduced amount
        t = lpTokensOut(balance, adjAmount, lptAMMBalance, tfee)
        tokensAdj = adjustLPTokens(lptAMMBalance, t, isDeposit=True)

        # Recalculate asset amount with the new tokens
        assetAdj = ammAssetIn(balance, lptAMMBalance, tokensAdj, tfee)

    # Return adjusted tokens and the minimum of original or calculated amount
    return (tokensAdj, min(amount, assetAdj))
```

# 3. AMM Formulas

Core AMM mathematical formulas for calculating LP tokens and asset amounts.

## 3.1. Swap Formulas

Swap formulas are used when trading assets through the AMM pool.

### 3.1.1. swapAssetIn

Calculate output amount given a specific input amount.[^swap-asset-in] It answers the question: "If I provide X amount of input asset, how much output asset do I receive?"

[^swap-asset-in]: `swapAssetIn`: [`AMMHelpers.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/ledger/helpers/AMMHelpers.h#L424-L477)

This function uses the formula `out = B - (A * B) / (A + in * (1 - fee))`, but with rounding at every step to protect the AMM pool. Used by [`BookStep`](../flow/steps.md#54-amm-integration) in the payment engine when consuming AMM liquidity. The function rounds downward, ensuring the pool gives out slightly less than the theoretical amount to protect against precision-based value leakage.

#### 3.1.1.1. swapAssetIn Pseudo-Code

```python
def swapAssetIn(
        pool,          # TAmounts<TIn, TOut> with pool.in and pool.out balances
        assetIn,       # Amount of input asset to swap in
        tfee):         # Trading fee in 1/10 basis point units
    fee = tfee / 100000

    with rounding_mode('upward'):
        numerator = pool.in * pool.out

    with rounding_mode('downward'):
        denom = pool.in + assetIn * (1 - fee)

    if denom <= 0:
        return 0

    with rounding_mode('upward'):
        ratio = numerator / denom

    with rounding_mode('downward'):
        swapOut = pool.out - ratio

    if swapOut < 0:
        return 0

    with rounding_mode('downward'):
        return swapOut
```

### 3.1.2. swapAssetOut

Calculate required input amount for a specific desired output.[^swap-asset-out] It answers the question: "If I want to receive Y amount of output asset, how much input asset must I provide?"

[^swap-asset-out]: `swapAssetOut`: [`AMMHelpers.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/ledger/helpers/AMMHelpers.h#L490-L543)

This is the inverse of [`swapAssetIn`](#311-swapassetin). It uses the formula `in = ((A * B) / (B - out) - A) / (1 - fee)` to determine how much input is needed to extract a specific output amount while maintaining the pool ratio. Used by [`BookStep`](../flow/steps.md#54-amm-integration) when the payment engine needs to deliver a specific amount.

#### 3.1.2.1. swapAssetOut Pseudo-Code

```python
def swapAssetOut(
        pool,          # TAmounts<TIn, TOut> with pool.in and pool.out balances
        assetOut,      # Desired amount of output asset
        tfee):         # Trading fee in 1/10 basis point units
    fee = tfee / 100000

    with rounding_mode('upward'):
        numerator = pool.in * pool.out

    with rounding_mode('downward'):
        denom = pool.out - assetOut

    if denom <= 0:
        return MAX_AMOUNT  # Cannot extract more than pool has

    with rounding_mode('upward'):
        ratio = numerator / denom
        numerator2 = ratio - pool.in

    with rounding_mode('downward'):
        feeMult = 1 - fee

    with rounding_mode('upward'):
        swapIn = numerator2 / feeMult

    if swapIn < 0:
        return 0

    with rounding_mode('upward'):
        return swapIn
```

### 3.1.3. Slippage and Quality Degradation

The **quality** of a swap is defined as `input / output` (how much you pay per unit received). Lower quality is better for the trader.

For `swapAssetOut`, we can derive:

```
quality = in / out
        = [((A*B) / (B-out)-A) / (1-fee)] / out
        = [A * out] / [(B-out) * (1-fee) * out]
        = A / [(B-out) * (1-fee)]
```

As `out` **increases**, the denominator `(B-out)` **decreases**, making quality **increase** (worse for trader).

**Example: Quality degradation in a 1,000 USD / 10,000 EUR pool with 0.3% fee**

| Desired Output (EUR) | ~Required Input (USD) | ~Quality |
|----------------------|-----------------------|----------|
| 100                  | 10.13                 | 0.101    |
| 500                  | 52.79                 | 0.106    |
| 1,000                | 111.45                | 0.111    |
| 2,000                | 250.75                | 0.125    |
| 5,000                | 1,003.01              | 0.201    |

In `xrpld`, this can be verified using `xrpld --unittest=AMMCalc` and appropriate logging, e.g.: 
```bash
xrpld --unittest=AMMCalc --unittest-arg "swapout,A(USD(1000),EUR(10000)),EUR(5000),300"
```

### 3.1.4. changeSpotPriceQuality

Calculate an AMM offer size such that after the swap, the pool's reserve ratio equals a target quality (typically the best CLOB offer quality).[^change-spot-price-quality]

[^change-spot-price-quality]: `changeSpotPriceQuality`: [`AMMHelpers.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/ledger/helpers/AMMHelpers.h#L302-L399)

This function is called from `AMMLiquidity::getOffer()` during single-path pathfinding when integrating AMM and CLOB (Central Limit Order Book) liquidity.[^amm-liquidity-get-offer] When a CLOB quality is available for comparison, this function sizes the AMM offer so that consuming it would move the AMM's spot price to match the best CLOB offer quality.

[^amm-liquidity-get-offer]: AMMLiquidity getOffer usage: [`AMMLiquidity.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/AMMLiquidity.cpp#L211-L212)

When one side of the pool is XRP, the algorithm calculates the XRP side first (whether that's takerGets or takerPays), then derives the other side from it. This differs from IOU-only or MPT pools where takerPays is calculated first. 


#### 3.1.4.1. changeSpotPriceQuality Pseudo-Code

```python
def changeSpotPriceQuality(
        pool,           # TAmounts<TIn, TOut> with pool.in and pool.out balances
        quality,        # Target quality (typically best CLOB offer quality)
        tfee,           # Trading fee in units of 1/100,000
        rules):         # Ledger rules (for amendment checks)
    """
    This pseudocode assumes fixAMMv1_1 is enabled
    """

    if isXRP(getAsset(pool.out)):
        # TakerGets is XRP - calculate it first
        amounts = getAMMOfferStartWithTakerGets(pool, quality, tfee)
    else:
        # TakerPays is XRP (or both are IOUs) - calculate takerPays first
        amounts = getAMMOfferStartWithTakerPays(pool, quality, tfee)

    # Even after sizing, the resulting offer may still be worse than the target.
    if not amounts or Quality(amounts) < quality:
        return None
    return amounts
```

#### 3.1.4.2. getAMMOfferStartWithTakerGets

Calculate AMM offer starting with takerGets (output) when takerGets is XRP.[^get-amm-offer-start-with-taker-gets]

[^get-amm-offer-start-with-taker-gets]: `getAMMOfferStartWithTakerGets`: [`AMMHelpers.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/ledger/helpers/AMMHelpers.h#L175-L215)

**Two scenarios considered:**

The algorithm solves for two different scenarios and selects the smallest takerGets to maximize quality (smaller offer means less slippage):

**Scenario A:** Spot Price Quality after consumption equals target quality
- Condition: `Qsp = (O - o) / (I + i) = targetQuality`
- Where: O = pool.out, I = pool.in, o = takerGets, i = takerPays
- Substitute `i` from swap equation: `i = (I * o) / ((O - o) * f)` where `f = 1 - tfee/100000`
- Results in quadratic: `o^2 + o * (I * (1 - 1/f) / Qt - 2*O) + O^2 - I * O / Qt = 0`
- NB: The code uses `targetQuality.rate()` which returns `1/Qt`, so the code divides by `targetQuality.rate()` where the formula in the `xrpld` comment multiplies by Qt (comment and XLS-30 use Qt = targetQuality). Here, we adopt that Qt = 1 / targetQuality.

**Scenario B:** Effective Price Quality equals target quality
- Condition: `Qep = o / i = targetQuality`
- Substitute swap equation and solve for o: `o = O - I / (Qt * f)`

##### 3.1.4.2.1. getAMMOfferStartWithTakerGets Pseudo-Code

```python
def getAMMOfferStartWithTakerGets(pool, targetQuality, tfee):
    if targetQuality == 0:
        return None

    f = 1 - (tfee / 100000)  # Fee multiplier
    Qt = 1 / targetQuality

    # Scenario A: Solve quadratic where spot price after = target quality
    # o^2 + o * (I * (1 - 1/f) / Qt - 2*O) + O^2 - I * O / Qt = 0
    a = 1
    b = pool.in * (1 - 1/f) / Qt - 2 * pool.out
    c = pool.out * pool.out - pool.in * pool.out / Qt

    takerGetsA = solveQuadraticEqSmallest(a, b, c)
    if takerGetsA is None or takerGetsA <= 0:
        return None

    # Scenario B: Solve where effective price = target quality
    # o = O - I / (Qt * f)
    takerGetsB = pool.out - pool.in / (Qt * f)
    if takerGetsB <= 0:
        return None

    # Select smallest to maximize quality
    takerGets = min(takerGetsA, takerGetsB)

    # Round takerGets downward (minimizes offer, maximizes quality)
    # This has most impact when takerGets is XRP
    takerGets = toAmount(getAsset(pool.out), takerGets, roundingMode=downward)

    # Calculate takerPays using swapAssetOut
    takerPays = swapAssetOut(pool, takerGets, tfee)

    amounts = Amounts(in=takerPays, out=takerGets)

    # Try to reduce offer size to improve quality if needed
    if Quality(amounts) < targetQuality:
        # Reduce takerGets by 0.9999x and recalculate
        reducedTakerGets = takerGets * 0.9999
        reducedTakerGets = toAmount(getAsset(pool.out), reducedTakerGets, roundingMode=downward)
        reducedTakerPays = swapAssetOut(pool, reducedTakerGets, tfee)
        amounts = TAmounts(in=reducedTakerPays, out=reducedTakerGets)

    return amounts


def solveQuadraticEqSmallest(a, b, c):
    """
    Solve quadratic equation and return smallest positive root.

    Uses numerically stable citardauq formula:
    https://people.csail.mit.edu/bkph/articles/Quadratics.pdf
    """
    discriminant = b * b - 4 * a * c
    if discriminant < 0:
        return None

    # Citardauq formula: choose formula based on sign of b
    if b > 0:
        return (2 * c) / (-b - sqrt(discriminant))
    else:
        return (2 * c) / (-b + sqrt(discriminant))
```

#### 3.1.4.3. getAMMOfferStartWithTakerPays

Similar to `getAMMOfferStartWithTakerGets`, but solves for takerPays (i) instead of takerGets (o):

**Scenario A:** Spot Price Quality after consumption equals target quality
- Equation: `Qsp = (O - o) / (I + i) = targetQuality`
- Substitute `o` from swap equation: `o = (O * i * f) / (I + i * f)` where `f = 1 - tfee/100000`
- Results in quadratic: `i^2 * f + i * I * (1 + f) + I^2 - I * O / Qt = 0`
- NB: The code uses `targetQuality.rate()` which returns `1/Qt`, so the code multiplies by `targetQuality.rate()` where the formula divides by Qt. The `xrpld` comment uses Qt and we keep it for consistency.

**Scenario B:** Effective Price Quality equals target quality
- Equation: `Qep = o / i = targetQuality`
- Substitute swap equation and solve for i: `i = O / Qt - I / f`

##### 3.1.4.3.1. getAMMOfferStartWithTakerPays Pseudo-Code

```python
def getAMMOfferStartWithTakerPays(pool, targetQuality, tfee):
    """
    Generate AMM offer starting with takerPays when pool.in is XRP
    (or both assets are IOUs).

    Solves two scenarios and selects smallest takerPays to maximize quality.
    """
    if targetQuality == 0:
        return None

    f = 1 - (tfee / 100000)  # Fee multiplier
    Qt = targetQuality

    # Scenario A: Solve quadratic where spot price after = target quality
    # i^2 * f + i * I * (1 + f) + I^2 - I * O / Qt = 0
    a = f
    b = pool.in * (1 + f)
    c = pool.in * pool.in - pool.in * pool.out / Qt

    # Use numerically stable citardauq formula
    takerPaysA = solveQuadraticEqSmallest(a, b, c)
    if takerPaysA is None or takerPaysA <= 0:
        return None

    # Scenario B: Solve where effective price = target quality
    # i = O / Qt - I / f
    takerPaysB = pool.out / Qt - pool.in / f
    if takerPaysB <= 0:
        return None

    # Select smallest to maximize quality
    takerPays = min(takerPaysA, takerPaysB)

    # Round takerPays downward (minimizes offer, maximizes quality)
    # This has most impact when takerPays is XRP
    takerPays = toAmount(getAsset(pool.in), takerPays, roundingMode=downward)

    # Calculate takerGets using swapAssetIn
    takerGets = swapAssetIn(pool, takerPays, tfee)

    amounts = TAmounts(in=takerPays, out=takerGets)

    # Try to reduce offer size to improve quality if needed
    if Quality(amounts) < targetQuality:
        # Reduce takerPays by 0.9999x and recalculate
        reducedTakerPays = takerPays * 0.9999
        reducedTakerPays = toAmount(getAsset(pool.in), reducedTakerPays, roundingMode=downward)
        reducedTakerGets = swapAssetIn(pool, reducedTakerPays, tfee)
        amounts = TAmounts(in=reducedTakerPays, out=reducedTakerGets)

    return amounts
```

## 3.2. Deposit Formulas

### 3.2.1. lpTokensOut (Equation 3)

Calculate LP tokens to receive for a single-asset deposit.[^lp-tokens-out]

[^lp-tokens-out]: `lpTokensOut`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L63-L82)

```
t1 = [R - (sqrt(f2^2 + R/f1) - f2)] / [1 + (sqrt(f2^2 + R/f1) - f2)]
```

Where:
- `t1` = LP token ratio (fraction of total supply: `t / T`)
- `t` = `lpTokens` (LP tokens being issued to the depositor)
- `T` = `lptAMMBalance` (total outstanding LP tokens in the AMM)
- `R` = `assetDeposit / assetBalance` (deposit ratio)
- `f1` = `feeMult = 1 - tfee` (fee multiplier)
- `f2` = `feeMultHalf = (1 - tfee/2) / f1` (half-fee factor)

The final result is: `lpTokens = lptAMMBalance * t1`

For example, Alice creates an AMM with 100 USD and 100 EUR (0.3% trading fee). 

If now Bob deposits a proportional amount of 50 USD and 50 EUR, the new LP token amount for the pool would be SQRT(150 * 150) = 150, and since he is increasing it by 1/3, Bob would receive 50 LP tokens.

However, if Bob deposited only 100 USD, while AMM instance considers 1 USD = 1 EUR, he would get less:

- Deposit ratio: R = 100/100 = 1.0
- Fee multipliers: f1 = 1 - 0.003 = 0.997, f2 = (1 - 0.0015) / 0.997 ~= 1.001505
- Calculate t1: t1 = (R - sqrt(f2^2 + R/f1) + f2) / (1 + sqrt(f2^2 + R/f1) - f2) ~= 0.413591
- LP tokens = 100 * 0.413591 ~= **41.36 LP tokens**

The single-asset deposit of 100 USD yields 41.36 LP tokens, while a proportional deposit of 50 USD + 50 EUR (same total value of 100 units) would yield 50 LP tokens. The single-asset deposit is less capital-efficient because it creates an imbalance in the pool ratio, and the trading fee is applied to account for this inefficiency.

### 3.2.1.1. lpTokensOut Pseudo-Code 

```python
def lpTokensOut(
        assetBalance,      # Current pool balance of the asset
        assetDeposit,      # Amount of asset to deposit
        lptAMMBalance,     # Total outstanding LP tokens
        tradingFee):       # Trading fee in 1/10 basis point units

    f1 = 1 - (tradingFee / 100000)
    f2 = (1 - (tradingFee / 200000)) / f1

    # Calculate the deposit ratio
    R = assetDeposit / assetBalance

    # Calculate c using Equation 3:
    c = sqrt((f2 * f2) + (R / f1)) - f2

    frac = (R - c) / (1 + c)
    return multiply(lptAMMBalance, frac, roundingMode="downward")
```

### 3.2.2. ammAssetIn (Equation 4)

Calculate required asset deposit for desired LP tokens.[^amm-asset-in]

[^amm-asset-in]: `ammAssetIn`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L95-L118)

Equation 4 is the inverse of Equation 3 (lpTokensOut). We solve Equation 3 for the asset deposit ratio `R = assetDeposit / assetBalance`.

Starting from Equation 3:
```
t1 = [R - (sqrt(f2^2 + R/f1) - f2)] / [1 + (sqrt(f2^2 + R/f1) - f2)]
```

Where the variables represent:
- `t1` = LP token ratio (what fraction of total supply the new tokens represent)
- `t` = `lpTokens` (the LP tokens being issued to the depositor)
- `T` = `lptAMMBalance` (total outstanding LP tokens in the AMM)
- `R` = `assetDeposit / assetBalance` (deposit ratio)
- `f1` = `feeMult = 1 - tfee` (fee multiplier)
- `f2` = `feeMultHalf = (1 - tfee/2) / f1` (half-fee factor)

For convenience, we define:
- `t2 = 1 + t1` (intermediate value)
- `d = f2 - t1/t2` (delta term used in derivation)

Algebraic steps:

1. Multiply both sides by denominator:
   ```
   t1 * [1 + sqrt(f2^2 + R/f1) - f2] = R - sqrt(f2^2 + R/f1) + f2
   ```

2. Expand and collect sqrt terms:
   ```
   sqrt(f2^2 + R/f1) * (t1 + 1) = R + f2 + t1*f2 - t1
   ```

3. Substitute `t2 = 1 + t1`:
   ```
   sqrt(f2^2 + R/f1) * t2 = R + t2*f2 - t1
   ```

4. Divide by `t2`:
   ```
   sqrt(f2^2 + R/f1) = R/t2 + f2 - t1/t2
   ```

5. Let `d = f2 - t1/t2`:
   ```
   sqrt(f2^2 + R/f1) = R/t2 + d
   ```

6. Square both sides:
   ```
   f2^2 + R/f1 = (R/t2)^2 + 2*d*R/t2 + d^2
   ```

7. Rearrange to quadratic form:
   ```
   (R/t2)^2 + R*(2*d/t2 - 1/f1) + (d^2 - f2^2) = 0
   ```

This gives us the quadratic coefficients:
- `a = 1/t2^2`
- `b = 2*d/t2 - 1/f1`
- `c = d^2 - f2^2`

We then solve using the quadratic formula to find `R`, and multiply by `assetBalance` to get the actual deposit amount.

#### 3.2.2.1. ammAssetIn Pseudo-Code

```python
def ammAssetIn(
        assetBalance,      # Current pool balance of the asset (B)
        lptAMMBalance,     # Total outstanding LP tokens (T)
        lpTokensDesired,   # LP tokens desired (t)
        tradingFee):       # Trading fee in 1/10 basis points units
    """
    Assumes fixAMMv1_3 is enabled
    """

    # Fee multipliers (see derivation above for explanation)
    f1 = 1 - (tradingFee / 100000)
    f2 = (1 - (tradingFee / 200000)) / f1

    # Calculate LP token fraction
    # Example: 5,000 / 50,000 = 0.1 (wanting 10% of pool)
    t1 = lpTokensDesired / lptAMMBalance  

    # Intermediate value
    t2 = 1 + t1 

    # Delta term (see derivation step 5)
    d = f2 - t1 / t2

    # Solve quadratic equation: a*R^2 + b*R + c = 0
    # where R = assetDeposit / assetBalance (what we're solving for)

    # Quadratic coefficients (see derivation step 7)
    a = 1 / (t2 * t2)
    b = 2 * d / t2 - 1 / f1
    c = d * d - f2 * f2

    # Solve using quadratic formula: R = (-b + sqrt(b^2 - 4ac)) / (2a)
    discriminant = b * b - 4 * a * c
    R = (-b + sqrt(discriminant)) / (2 * a)

    # Convert fraction R to actual deposit amount and round upward (protect pool)
    return multiply(assetBalance, R, roundingMode=upward)
```

## 3.3. Withdrawal Formulas

### 3.3.1. lpTokensIn (Equation 7)

Calculate LP tokens to redeem for a single-asset withdrawal.[^lp-tokens-in]

[^lp-tokens-in]: `lpTokensIn`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L124-L143)

```
t = T * (c - sqrt(c^2 - 4R)) / 2
```

Where:
- `t` = LP tokens to redeem
- `T` = `lptAMMBalance` (total outstanding LP tokens in the AMM)
- `R` = `assetWithdraw / assetBalance` (withdrawal ratio)
- `c` = `R * fee + 2 - fee`
- `fee` = `tradingFee / 100000` (raw fee value, e.g., 0.003 for 0.3% fee)

### 3.3.1.1. lpTokensIn Pseudo-Code

```python
def lpTokensIn(
        assetBalance,      # Current pool balance of the asset (B)
        assetWithdraw,     # Amount of asset to withdraw (b)
        lptAMMBalance,     # Total outstanding LP tokens (T)
        tradingFee):       # Trading fee in units of 1/100,000
    # Calculate withdrawal ratio
    R = assetWithdraw / assetBalance

    # Get raw fee value (not a multiplier)
    fee = tradingFee / 100000  # e.g., 0.003 for 0.3% fee

    # Calculate intermediate value c
    c = R * fee + 2 - fee

    # Calculate discriminant
    discriminant = c * c - 4 * R

    # Calculate LP tokens using the formula
    # With fixAMMv1_3: round upward (maximize tokens in, protect pool)
    frac = (c - sqrt(discriminant)) / 2
    return multiply(lptAMMBalance, frac, roundingMode=upward)
```

### 3.3.2. ammAssetOut (Equation 8)

Calculate asset withdrawal for redeeming LP tokens.[^amm-asset-out]

[^amm-asset-out]: `ammAssetOut`: [`AMMHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/AMMHelpers.cpp#L155-L173)

Equation 8 is the inverse of Equation 7 (lpTokensIn). We solve Equation 7 for the asset withdrawal amount.

Starting from Equation 7:
```
t = T * (c - sqrt(c^2 - 4R)) / 2
t1 = (c - sqrt(c^2 - 4R)) / 2
```

Where:
- `c = R*fee + 2 - fee`
- `R` = `assetWithdraw / assetBalance` (withdrawal ratio)
- `t1` = `lpTokensRedeem / lptAMMBalance` (LP token fraction - what we know)
- `fee` = `tradingFee / 100000` (raw fee value, same as in lpTokensIn)

The derivation solves for the withdrawal ratio `R`, then multiplies by `assetBalance` to get the actual asset withdrawal amount.

Algebraic steps:

1. Rearrange to isolate the square root:
   ```
   c - 2*t1 = sqrt(c^2 - 4R)
   ```

2. Square both sides:
   ```
   c^2 - 4*c*t1 + 4*t1^2 = c^2 - 4R
   ```

3. Simplify (c^2 cancels):
   ```
   -4*c*t1 + 4*t1^2 = -4R
   ```

4. Divide by -4:
   ```
   -c*t1 + t1^2 = -R
   ```

5. Substitute `c = R*fee + 2 - fee`:
   ```
   -(R*fee + 2 - fee)*t1 + t1^2 = -R
   ```

6. Expand:
   ```
   -t1*R*fee - 2*t1 + t1*fee + t1^2 = -R
   ```

7. Solve for R:
   ```
   R = (t1^2 - t1*(2 - fee)) / (t1*fee - 1)
   ```

### 3.3.2.1. ammAssetOut Pseudo-Code

```python
def ammAssetOut(
        assetBalance,      # Current pool balance of the asset (B)
        lptAMMBalance,     # Total outstanding LP tokens (T)
        lpTokensRedeem,    # LP tokens to redeem (t)
        tradingFee):       # Trading fee in units of 1/100,000
    """
    Assumes fixAMMv1_3 is enabled
    """

    # Calculate LP token fraction
    t1 = lpTokensRedeem / lptAMMBalance

    # Get raw fee value (same as in lpTokensIn)
    fee = tradingFee / 100000  # e.g., 0.003 for 0.3% fee

    # Calculate R using the derived formula (see derivation step 7 above)
    R = (t1 * t1 - t1 * (2 - fee)) / (t1 * fee - 1)

    # Convert ratio R to actual withdrawal amount and round downward (protect pool)
    return multiply(assetBalance, R, roundingMode=downward)
```
