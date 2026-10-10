# Index

- [1. Introduction](#1-introduction)
- [2. Withdraw Logic](#2-withdraw-logic)
  - [2.1. Pseudo-Code](#21-pseudo-code)
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

# 2. Withdraw Logic

The `applyGuts` function[^applyGuts] is the main entry point for processing AMMWithdraw transactions. It retrieves the AMM ledger entry and the withdrawer's LP token balance, determines how many LP tokens to redeem (all tokens for `tfWithdrawAll`/`tfOneAssetWithdrawAll`, or `LPTokenIn` for the modes that take it), then adjusts the LP token balance for precision if needed. The function gets the current pool balances and determines which trading fee applies to the withdrawer (regular or discounted for [auction slot holders](#3-gettradingfee)). Based on the transaction flags and provided fields, it dispatches to one of five withdrawal mode handlers (implementing seven total modes): two [multi-asset modes](#4-multi-asset-withdrawal-modes) that maintain proportional withdrawals, and three [single-asset modes](#5-single-asset-withdrawal-modes) that perform single-sided withdrawals. Each mode handler calculates the withdrawal amounts and LP tokens to burn, then calls the [common withdraw function](#6-common-withdraw-function) to execute the actual asset transfers and update the pool state. After the withdrawal, if the pool is empty (zero LP tokens), the function attempts to delete the AMM account - if successful, the AMM is fully removed; if incomplete due to remaining trust lines, the AMM remains in an empty state with the LP token balance set to zero.

Under the `fixCleanup3_3_0` amendment, the freeze rules relax as described in the [failure conditions](README.md#332-failure-conditions).

Under the `fixCleanup3_4_0` amendment, the mode dispatch is wrapped in the same guard as in [deposit](deposit.md#2-deposit-logic). This one is defense in depth: no known withdrawal path can produce an out-of-range amount, since `preclaim` bounds requested amounts by the pool balances.

[^applyGuts]: AMMWithdraw::applyGuts: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.4.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L337-L486)

## 2.1. Pseudo-Code

```python
def withdraw_from_pool(pool, withdrawer, tx):
    held = pool.lp_tokens_of(withdrawer)
    if tx.flags & (tfWithdrawAll | tfOneAssetWithdrawAll):
        lp_tokens_in = held
    else:
        lp_tokens_in = tx.LPTokenIn  # absent for the modes that work it out

    # Rounding can leave the last LP's tokens slightly off the pool's LP token supply
    if pool.is_only_lp(withdrawer):
        if not within_relative_distance(held, pool.lp_token_supply, 0.001):
            fail(tecAMM_INVALID_TOKENS)
        pool.lp_token_supply = held

    fee = getTradingFee(pool, withdrawer)

    # Frozen reserves and MPTs the AMM may not hold read as zero. When the withdrawer
    # issues either pool asset, both reserves are read through the freeze, because
    # an issuer can always receive its own token.
    balance1, balance2, lp_supply = pool.balances(
        tx.Amount, tx.Amount2,
        ignore_freeze=withdrawer in (pool.Asset.issuer, pool.Asset2.issuer))

    withdraw_mode = {
        tfLPToken: equalWithdrawTokens,
        tfWithdrawAll: equalWithdrawTokens,
        tfTwoAsset: equalWithdrawLimit,
        tfSingleAsset: singleWithdraw,
        tfOneAssetLPToken: singleWithdrawTokens,
        tfOneAssetWithdrawAll: singleWithdrawTokens,
        tfLimitLPToken: singleWithdrawEPrice,
    }[tx.withdraw_mode_flag]

    try:
        new_lp_supply = withdraw_mode(pool, balance1, balance2, lp_supply, lp_tokens_in, tx, fee)
    except AmountOutOfRange:
        if fixCleanup3_4_0:
            fail(tecAMM_FAILED)
        fail(tefEXCEPTION)

    # The remaining LP tokens must not exceed the geometric mean of the pool's actual reserves
    pool_mean = sqrt(pool.reserve1 * pool.reserve2)
    if (new_lp_supply > pool_mean
            and not within_relative_distance(pool_mean, new_lp_supply, 1e-11)):
        fail(tecPRECISION_LOSS)

    pool.lp_token_supply = new_lp_supply
    if new_lp_supply == 0:
        # Deletes what it can. With too many trust lines, the empty AMM stays for AMMDelete.
        pool.delete()
```

# 3. getTradingFee

Determines the trading fee for the withdrawer, accounting for auction slot discounts. Same implementation as AMMDeposit, see [deposit.md](deposit.md#3-gettradingfee).

# 4. Multi-Asset Withdrawal Modes

Multi-asset withdrawal modes maintain proportional withdrawals by removing both pool assets simultaneously. This preserves the pool's price (asset ratio) while decreasing liquidity. Since these withdrawals maintain proportional ratios, they incur no trading fees.

The modes are:
- **[equalWithdrawTokens](#41-equalwithdrawtokens-tflptoken-tfwithdrawall) (tfLPToken, tfWithdrawAll)** - Redeem LP tokens or all LP tokens, receive proportional amounts of both assets
- **[equalWithdrawLimit](#42-equalwithdrawlimit-tftwoasset) (tfTwoAsset)** - Specify maximum amounts for both assets, system calculates LP tokens to burn

## 4.1. equalWithdrawTokens (tfLPToken, tfWithdrawAll)

> "I want to redeem X LP tokens, how much of both assets do I get?" (tfLPToken)
> "Redeem all my LP tokens for both assets." (tfWithdrawAll)

Proportional withdrawal of pool assets for the amount of LP tokens.[^equalWithdrawTokens]

[^equalWithdrawTokens]: AMMWithdraw::equalWithdrawTokens: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.4.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L903-L990)

This function handles two related modes. With `tfLPToken`, the user requests a number of LP tokens using `LPTokenIn`. For a partial withdrawal, the amount redeemed can be rounded down slightly for precision. The function calculates the proportional amounts of both assets to withdraw. With `tfWithdrawAll`, the user redeems their entire LP token balance without specifying an amount. The function handles a special case when withdrawing all LP tokens from the pool (`lpTokensWithdraw == lptAMMBalance`), which empties the pool completely. For partial withdrawals, it calculates the pool fraction (`frac = tokensAdj / lptAMMBalance`), then multiplies each asset balance by this fraction, rounding down with [`getRoundedAsset`](helpers.md#23-getroundedasset) to ensure the pool retains sufficient assets.

**Example:**

A pool has 150 USD, 150 EUR, and 150 LP tokens outstanding. Bob holds 30 LP tokens (20% of the pool).

**Case 1: Bob redeems exactly 15 LP tokens (tfLPToken)**
- LP tokens to redeem: 15
- Fraction of pool: 15 / 150 = 0.1 (10%)
- USD withdrawn: 150 * 0.1 = 15 USD
- EUR withdrawn: 150 * 0.1 = 15 EUR
- Result: Bob redeems 15 LP tokens, receives 15 USD + 15 EUR
- Bob now holds: 15 LP tokens (10% of the original 150, or 1/9 of the remaining 135)

**Case 2: Bob redeems all his LP tokens (tfWithdrawAll)**
- LP tokens to redeem: 30 (all Bob's holdings)
- Fraction of pool: 30 / 150 = 0.2 (20%)
- USD withdrawn: 150 * 0.2 = 30 USD
- EUR withdrawn: 150 * 0.2 = 30 EUR
- Result: Bob redeems 30 LP tokens, receives 30 USD + 30 EUR
- Bob now holds: 0 LP tokens

### 4.1.1 equalWithdrawTokens Pseudo-Code

```python
def equalWithdrawTokens(pool, balance1, balance2, lp_supply, lp_tokens_in, tx, fee):
    if lp_tokens_in == lp_supply:
        return withdraw(pool, balance1, balance2, lp_tokens_in, withdraw_all=True)

    withdraw_all = tx.flags & tfWithdrawAll
    lp_tokens = lp_tokens_in if withdraw_all else adjustLPTokensIn(lp_supply, lp_tokens_in)
    if lp_tokens == 0:
        fail(tecAMM_INVALID_TOKENS)

    share = lp_tokens / lp_supply
    amount1 = getRoundedAsset(balance1, share, IsDeposit=False)
    amount2 = getRoundedAsset(balance2, share, IsDeposit=False)
    # With too few LP tokens, one side rounds to zero and the withdrawal would take only the other asset
    if amount1 == 0 or amount2 == 0:
        fail(tecAMM_FAILED)

    return withdraw(pool, amount1, amount2, lp_tokens, withdraw_all=withdraw_all)
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
def equalWithdrawLimit(pool, balance1, balance2, lp_supply, lp_tokens_in, tx, fee):
    def take_all_of(amount, balance, other_balance):
        share = amount / balance
        lp_tokens = getRoundedLPTokens(lp_supply, share, IsDeposit=False)
        if lp_tokens == 0:
            fail(tecAMM_INVALID_TOKENS)
        share = adjustFracByTokens(lp_supply, lp_tokens, share)  # the share the rounded LP tokens redeem
        return lp_tokens, getRoundedAsset(other_balance, share, IsDeposit=False)

    lp_tokens, amount2 = take_all_of(tx.Amount, balance1, balance2)
    if amount2 <= tx.Amount2:
        return withdraw(pool, tx.Amount, amount2, lp_tokens)

    lp_tokens, amount1 = take_all_of(tx.Amount2, balance2, balance1)
    if amount1 <= tx.Amount:
        return withdraw(pool, amount1, tx.Amount2, lp_tokens)

    fail(tecAMM_FAILED)
```

# 5. Single-Asset Withdrawal Modes

Single-asset withdrawal modes allow users to withdraw only one asset instead of both assets proportionally. Unlike [multi-asset withdrawals](#4-multi-asset-withdrawal-modes) that maintain the pool ratio, single-asset withdrawals change the pool composition. Because they alter the pool ratio, [trading fees](#3-gettradingfee) apply to single-asset withdrawals. There are three modes:

- **[singleWithdraw](#51-singlewithdraw-tfsingleasset) (tfSingleAsset)** - User specifies withdrawal amount, system calculates LP tokens to redeem
- **[singleWithdrawTokens](#52-singlewithdrawtokens-tfoneassetlptoken-tfoneassetwithdrawall) (tfOneAssetLPToken, tfOneAssetWithdrawAll)** - User specifies LP tokens to redeem or all LP tokens, withdraws calculated amount of single asset
- **[singleWithdrawEPrice](#53-singlewithdraweprice-tflimitlptoken) (tfLimitLPToken)** - User specifies minimum effective price limit

## 5.1. singleWithdraw (tfSingleAsset)

> "I want to withdraw X amount, how many LP tokens must I redeem?"

The user specifies `Amount` (the asset amount to withdraw) and the function calculates how many LP tokens must be redeemed.[^singleWithdraw] Since this is a single-asset withdrawal that changes the pool ratio, a [trading fee](#3-gettradingfee) applies. The function uses [`lpTokensIn`](helpers.md#331-lptokensin-equation-7) (Equation 7) to calculate the LP tokens based on the withdrawal amount and trading fee, then adjusts the result for precision with [`adjustLPTokensIn`](helpers.md#25-adjustlptokensin-withdrawals). The adjusted tokens are passed to `adjustAssetOutByTokens`, which prices them in the withdrawn asset. If the result exceeds `Amount`, it recomputes the LP tokens from a smaller target amount. The withdrawal is capped at `Amount`.

[^singleWithdraw]: AMMWithdraw::singleWithdraw: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L969-L1007)

### 5.1.1. singleWithdraw Pseudo-Code

```python
def singleWithdraw(pool, balance1, balance2, lp_supply, lp_tokens_in, tx, fee):
    lp_tokens = adjustLPTokensIn(lp_supply, lpTokensIn(balance1, tx.Amount, lp_supply, fee))
    if lp_tokens == 0:
        fail(tecAMM_INVALID_TOKENS)

    # If rounding makes these LP tokens worth more than Amount, they are recomputed
    # from a smaller amount, and the withdrawal is capped at Amount
    lp_tokens, amount = adjustAssetOutByTokens(balance1, tx.Amount, lp_supply, lp_tokens, fee)
    if lp_tokens == 0:
        fail(tecAMM_INVALID_TOKENS)

    return withdraw(pool, amount, None, lp_tokens)
```

## 5.2. singleWithdrawTokens (tfOneAssetLPToken, tfOneAssetWithdrawAll)

> "I'll redeem X LP tokens, how much asset (single asset only) do I get?" (tfOneAssetLPToken)
> "Redeem all my LP tokens for a single asset." (tfOneAssetWithdrawAll)

Withdraw a single asset by redeeming specified LP tokens.[^singleWithdrawTokens]

[^singleWithdrawTokens]: AMMWithdraw::singleWithdrawTokens: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L1020-L1051)

This function handles the reverse calculation from [`singleWithdraw`](#51-singlewithdraw-tfsingleasset). The user names the LP tokens to redeem: `LPTokenIn` for tfOneAssetLPToken, or all their LP tokens for tfOneAssetWithdrawAll. The function calculates how much of a single asset they receive. `Amount` names that asset. When it is not zero, it is also the minimum the user accepts.

For tfOneAssetLPToken, the function first rounds the LP tokens down for precision using [`adjustLPTokensIn`](helpers.md#25-adjustlptokensin-withdrawals), so it can redeem slightly fewer than `LPTokenIn`. It then uses [`ammAssetOut`](helpers.md#332-ammassetout-equation-8) (Equation 8) to calculate the withdrawal amount by solving the inverse single-asset withdrawal problem. If the calculated amount is less than the user's `Amount` constraint (when non-zero), the transaction fails with `tecAMM_FAILED`.

### 5.2.1. singleWithdrawTokens Pseudo-Code

```python
def singleWithdrawTokens(pool, balance1, balance2, lp_supply, lp_tokens_in, tx, fee):
    withdraw_all = tx.flags & tfOneAssetWithdrawAll
    lp_tokens = lp_tokens_in if withdraw_all else adjustLPTokensIn(lp_supply, lp_tokens_in)
    if lp_tokens == 0:
        fail(tecAMM_INVALID_TOKENS)

    amount = ammAssetOut(balance1, lp_supply, lp_tokens, fee)
    if tx.Amount != 0 and amount < tx.Amount:
        fail(tecAMM_FAILED)

    return withdraw(pool, amount, None, lp_tokens, withdraw_all=withdraw_all)
```

## 5.3. singleWithdrawEPrice (tfLimitLPToken)

> "I'll withdraw (single asset), but only if the effective price per LP token is reasonable."

Withdraw a single asset with an effective price constraint.[^singleWithdrawEPrice]

This mode allows users to control the effective price when redeeming LP tokens, where effective price is defined as the ratio of LP tokens redeemed to asset withdrawn. The user provides `EPrice` (maximum effective price) and `Amount`, which names the asset. When `Amount` is not zero, it is the minimum the user accepts. As with deposits, `EPrice` is an upper bound. The trade is sized so the effective price equals `EPrice`, and rounding can move the final amounts slightly off it. A lower effective price means a better deal for the withdrawer (fewer LP tokens per asset withdrawn).

The function solves a derived formula from Equation 8 to calculate the LP tokens that achieve exactly the specified effective price. It then calculates the withdrawal amount as `tokensAdj / ePrice`. If the calculated amount is less than a non-zero `Amount`, the transaction fails with `tecAMM_FAILED`. Under the `fixCleanup3_3_0` amendment, a denominator (`T*f - B*E`) of exactly zero also fails with `tecAMM_FAILED`. Without the amendment, that division throws. The transaction then fails with `tefEXCEPTION`, or with `tecAMM_FAILED` under `fixCleanup3_4_0`.

[^singleWithdrawEPrice]: AMMWithdraw::singleWithdrawEPrice: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L1119-L1179)

### 5.3.1. singleWithdrawEPrice Pseudo-Code

```python
def singleWithdrawEPrice(pool, balance1, balance2, lp_supply, lp_tokens_in, tx, fee):
    f = fee / 100_000
    denominator = lp_supply * f - balance1 * tx.EPrice
    if denominator == 0:
        fail(tecAMM_FAILED)

    # Equation 8 solved for the LP tokens at an effective price of exactly EPrice, before rounding
    lp_tokens = adjustLPTokensIn(lp_supply, round_up(
        lp_supply * (lp_supply + balance1 * tx.EPrice * (f - 2)) / denominator))
    if lp_tokens <= 0:
        fail(tecAMM_INVALID_TOKENS)

    amount = round_down(lp_tokens / tx.EPrice)
    if tx.Amount != 0 and amount < tx.Amount:
        fail(tecAMM_FAILED)

    return withdraw(pool, amount, None, lp_tokens)
```

# 6. Common Withdraw Function

The `withdraw()` function[^withdraw] serves as the final common pathway for all withdrawal modes, executing the actual asset transfers after mode-specific handlers determine the withdrawal amounts.

Under `fixCleanup3_3_0` together with `fixAMMv1_3`, `applyGuts` runs the pool product check described in [Precision and Rounding](helpers.md#2-precision-and-rounding) after the mode handlers return. See the [failure conditions](README.md#332-failure-conditions) for the resulting `tecPRECISION_LOSS`.

[^withdraw]: AMMWithdraw::withdraw: [AMMWithdraw.cpp](https://github.com/XRPLF/rippled/blob/3.4.0/src/libxrpl/tx/transactors/dex/AMMWithdraw.cpp#L537-L818)

This function orchestrates a sequenced validation and execution flow. It begins by verifying the withdrawer holds sufficient LP tokens to redeem, then enforces pool integrity constraints that prevent malformed states. 

The function prohibits one-sided pool withdrawals - situations where all of one asset would be withdrawn while the other remains. When a withdrawal would redeem all outstanding LP tokens, the function mandates that all pool assets must also be withdrawn, preventing orphaned assets in an empty pool.

Before executing transfers, the function checks whether the withdrawer has adequate XRP reserves if new trust lines or MPTokens need creation. For MPTs, regular withdrawals also verify authorization from the issuer. With `fixCleanup3_4_0`, the AMMClawback path skips the reserve check: clawback takes priority over the reserve, so a holder cannot avoid it by deleting the trust line or MPToken and keeping a low spendable balance. Two other 3.4.0 changes affect MPT withdrawals. The clawback path recreates a missing holder MPToken instead of failing, so the issuer can recover MPT locked in the pool, and auto-authorizes it only for the clawback issuer's own asset, since granting authorization for another issuer's asset would bypass that issuer's `lsfMPTRequireAuth`. The reserve comparison uses the higher of the pre-transaction and current XRP balance for MPTs as well, counting XRP already received earlier in the same withdrawal, as the trust line path always did. The function then transfers each withdrawn asset from the AMM account to the withdrawer, waiving transfer fees as XLS-30 specifies. Finally, it burns the redeemed LP tokens by calling `redeemIOU`, which reduces both the withdrawer's LP token balance and the total outstanding token supply.

## 6.1. withdraw Pseudo-Code

```python
def withdraw(pool, amount1, amount2, lp_tokens, withdraw_all=False, clawback_issuer=None):
    clawback = clawback_issuer is not None
    # Read as in section 2. Clawback reads through freezes and missing authorization.
    balance1, balance2 = pool.balances(amount1.asset)

    if lp_tokens <= 0 or lp_tokens > pool.lp_tokens_of(withdrawer):
        fail(tecAMM_INVALID_TOKENS)
    # Never empty one side of the pool while the other keeps a balance
    if (amount1 == balance1) != (amount2 == balance2):
        fail(tecAMM_BALANCE)
    # Redeeming every LP token must take both full balances
    if lp_tokens == pool.lp_token_supply and (amount1 != balance1 or amount2 != balance2):
        fail(tecAMM_BALANCE)
    if amount1 > balance1 or (amount2 is not None and amount2 > balance2):
        fail(tecAMM_BALANCE)

    # With MPTokensV2: afterwards, the balances and the LP tokens are all zero or all non-zero.
    # A single-asset withdrawal ignores the other side, which can read as zero when frozen.
    if MPTokensV2:
        emptied = [amount1 == balance1, lp_tokens == pool.lp_token_supply]
        if amount2 is not None:
            emptied.append(amount2 == balance2)
        if any(emptied) and not all(emptied):
            fail(tecAMM_BALANCE)

    for amount in [amount1, amount2]:
        if amount is None:  # single-asset withdrawal
            continue
        if not amount.is_xrp and not withdrawer.has_trust_line_or_mptoken(amount.asset):
            # Clawback takes priority over the reserve, so a holder cannot avoid it
            # by deleting the trust line or MPToken and keeping a low balance
            if not (fixCleanup3_4_0 and clawback):
                # The first two owned objects need no reserve, as in TrustSet.
                # The higher balance counts XRP received earlier in this withdrawal.
                if (withdrawer.owner_count >= 2 and
                        max(withdrawer.xrp_before_tx, withdrawer.xrp) < reserve(withdrawer.owner_count + 1)):
                    fail(tecINSUFFICIENT_RESERVE)
            if amount.is_mpt and withdrawer != amount.issuer:
                flags = 0
                if requireAuth(withdrawer, amount.asset) != tesSUCCESS:
                    # Only clawback gets here. An issuer can authorize only its own asset,
                    # so another issuer's MPToken is recreated unauthorized.
                    if amount.issuer == clawback_issuer:
                        flags = lsfMPTAuthorized
                create_mptoken(withdrawer, amount.asset, flags)
        send(amount, from_=pool.account, to=withdrawer, transfer_fee=False)

    burn(lp_tokens, from_=withdrawer)
    return pool.lp_token_supply - lp_tokens
```
