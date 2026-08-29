# Index

- [1. Introduction](#1-introduction)
  - [1.1. Bidding Process](#11-bidding-process)
  - [1.2. Implementation](#12-implementation)
- [2. applyBid](#2-applybid)
  - [2.1. applyBid Pseudo-Code](#21-applybid-pseudo-code)
- [3. updateSlot](#3-updateslot)
  - [3.1. updateSlot Pseudo-Code](#31-updateslot-pseudo-code)
- [4. validOwner](#4-validowner)
  - [4.1. validOwner Pseudo-Code](#41-validowner-pseudo-code)
- [5. getPayPrice](#5-getpayprice)
  - [5.1. getPayPrice Pseudo-Code](#51-getpayprice-pseudo-code)
- [6. Price Calculation](#6-price-calculation)
  - [6.1. Minimum Slot Price](#61-minimum-slot-price)
  - [6.2. Computed Price for Owned Slots](#62-computed-price-for-owned-slots)
  - [6.3. Time Slot Calculation](#63-time-slot-calculation)
- [7. Refund Mechanism](#7-refund-mechanism)
- [8. LP Token Burning](#8-lp-token-burning)

# 1. Introduction

The auction slot is a 24-hour privilege that allows an account to trade through the AMM at a discounted fee (1/10th of the regular trading fee). When an AMM is created, the creator automatically receives the auction slot for free (with price set to 0). Subsequently, accounts can bid LP tokens to win the slot from the current holder, and the winning bid amount (minus any refund to the previous owner) is burned from circulation.

AMM pools cannot directly observe external market prices, so when asset prices diverge between the pool and external markets, arbitrageurs must intervene to rebalance the pool. However, traditional arbitrage faces two problems: (1) arbitrageurs must wait until their profit exceeds the trading fee, creating a window where the pool offers suboptimal prices and experiences reduced trading volume, and (2) multiple arbitrageurs compete in a race, reducing their individual success probability.

The auction slot solves this by offering discounted trading fees to the slot owner, enabling them to execute arbitrage immediately without waiting for profits to exceed fees. This eliminates the race and narrows the window of price inefficiency. The slot owner pays for this advantage by bidding LP tokens, which are burned upon winning - this burn reduces total LP token supply while pool assets remain unchanged, effectively distributing the auction proceeds to all LP token holders through increased ownership percentage.

**LP tokens** are used to bid. The winning bid amount (minus any refund) is burned from the total LP token supply, effectively distributing value to all remaining LP token holders.

The auction slot lasts 24 hours (86,400 seconds). Since the slot can be taken over by a new bidder at any time during this period, the slot owner may not use the full 24 hours.

If a new bidder takes over the slot before it expires, the previous owner receives a **refund** proportional to the remaining time: `refund = (1 - fractionUsed) * pricePurchased`. For example, if the previous owner paid 1,000 LP tokens and used 6 hours (time slot 5, so `(5+1)/20 = 30%` is treated as used), they receive a refund of 700 LP tokens (70% of what they paid). This compensates them for the unused portion of their slot time. The slot's lifecycle is tracked using 20 time intervals of ~1.2 hours each (4,320 seconds) to calculate how much time has been used.

**Price Structure**:
- When no one owns the slot or it has expired, bidders pay the minimum slot price: `TotalLPTokens * TradingFee / 25`. Note that if the `TradingFee` is 0, the minimum slot price is also 0, making the slot free to claim.
- When the slot is owned, new bidders must pay a computed price that starts with a 5% markup over the previous purchase price. This markup decays exponentially as time progresses - early in the slot period, the price is close to `pricePurchased * 1.05 + minSlotPrice`, but as time elapses, the markup portion diminishes and the price approaches just `minSlotPrice`

The slot owner pays only **1/10th** of the regular trading fee when they trade through the AMM. The slot owner can designate up to 4 additional accounts to share the discounted fee benefit.

## 1.1. Bidding Process

When an account submits an AMMBid transaction, they can optionally specify `BidMin` and `BidMax` constraints to control how much they're willing to pay. The system first calculates a computed price based on whether the slot is currently owned and how much time has elapsed since it was purchased. If no one owns the slot or it has expired, the computed price is the minimum slot price. If the slot is owned, the computed price starts with a 5% markup over what the previous owner paid, with this markup decaying exponentially as the slot ages.

The actual pay price is then determined by reconciling the computed price with the bidder's constraints. If the slot is currently owned and the bid succeeds, the previous owner receives a time-based refund proportional to the remaining slot time - this refund comes from the new bidder's payment. The remaining amount (pay price minus any refund) is burned from the LP token supply, reducing total supply while keeping pool assets unchanged. Finally, the auction slot is updated with the new owner, a fresh 24-hour expiration time, and the discounted trading fee.

## 1.2. Implementation

In the `xrpld` C++ implementation (`AMMBid.cpp`), the main transaction handler calls the [`applyBid`](#2-applybid) function with the transaction context, sandbox view, and bidder account. This function retrieves the AMM ledger entry and the bidder's LP token holdings, then calculates the [minimum slot price](#61-minimum-slot-price) and discounted fee based on the AMM's trading fee and total LP token supply.

The function calls [`ammAuctionTimeSlot`](#63-time-slot-calculation) to determine the current time interval (0-19) based on elapsed time since the slot was won. It then defines three lambda functions inline: [`validOwner`](#4-validowner) (checks if the current slot owner is valid and not in the expiring interval), [`updateSlot`](#3-updateslot) (updates auction slot fields and burns LP tokens), and [`getPayPrice`](#5-getpayprice) (determines the actual price to pay given the computed price and bid constraints).

The logic branches into two cases: If no one owns the slot or it has expired (determined by checking the slot's account field and calling [`validOwner`](#4-validowner)), the bidder pays the [minimum slot price](#61-minimum-slot-price) and the entire amount is [burned](#8-lp-token-burning). If the slot is currently owned, the function calculates the [time-based pricing](#62-computed-price-for-owned-slots) using the decay formula `pricePurchased * 1.05 * (1 - fractionUsed^60) + minSlotPrice`, determines the pay price via [`getPayPrice`](#5-getpayprice), transfers a proportional [refund](#7-refund-mechanism) to the previous owner using `accountSend`, and [burns](#8-lp-token-burning) the remaining amount. In both cases, [`updateSlot`](#3-updateslot) is called to finalize the auction slot state.

# 2. applyBid

The `applyBid` function is the main entry point that orchestrates the entire auction slot bidding flow. It handles slot ownership validation, price calculation, refund processing, and LP token burning. See [Implementation](#12-implementation) for a detailed walkthrough of how this function executes.

## 2.1. applyBid Pseudo-Code

```python
def applyBid(ctx: ApplyContext, sb: &Sandbox, account: AccountID):
    """
    Main bidding logic for auction slot.
    Returns (TER, bool) where bool indicates whether to apply the sandbox.
    """

    # Get AMM ledger entry
    ammSle = sb.getAMM(ctx.tx[sfAsset], ctx.tx[sfAsset2])
    if not ammSle:
        return (tecINTERNAL, False)

    lptAMMBalance = ammSle[sfLPTokenBalance]
    lpTokens = ammLPHolds(sb, ammSle, account)

    # Ensure auction slot exists
    # Without fixInnerObjTemplate: Create slot if missing
    # With fixInnerObjTemplate: Slot must already exist
    if not rules.enabled(fixInnerObjTemplate):
        if not ammSle.isFieldPresent(sfAuctionSlot):
            ammSle.makeFieldPresent(sfAuctionSlot)
    else:
        if not ammSle.isFieldPresent(sfAuctionSlot):
            return (tecINTERNAL, False)

    auctionSlot = ammSle.peekFieldObject(sfAuctionSlot)
    current = ctx.view().header().parentCloseTime  # in seconds

    # Calculate fees and prices
    discountedFee = ammSle[sfTradingFee] / kAuctionSlotDiscountedFeeFraction  # Divide by 10
    tradingFee = getFee(ammSle[sfTradingFee])
    minSlotPrice = lptAMMBalance * tradingFee / kAuctionSlotMinFeeFraction  # Divide by 25

    # Determine current time slot (0-19)
    # Returns None if slot is not owned or expired
    timeSlot = ammAuctionTimeSlot(current, auctionSlot)

    # Get bid constraints from transaction
    bidMin = ctx.tx[~sfBidMin]
    bidMax = ctx.tx[~sfBidMax]

    # CASE 1: No one owns slot or slot is expired
    currentOwner = auctionSlot[~sfAccount]
    if not currentOwner or not validOwner(currentOwner, timeSlot, sb):
        # Pay minimum price, no refund
        payPrice = getPayPrice(minSlotPrice, bidMin, bidMax, lpTokens)
        if payPrice is error:
            return (payPrice.error(), False)

        # Update slot with new owner
        result = updateSlot(
            sb,
            ammSle,
            auctionSlot,
            account,
            current,
            discountedFee,
            payPrice,
            payPrice,  # burn entire amount (no refund)
            lpTokens.asset(),
            lptAMMBalance,
            ctx.tx,
        )
        return (result, result == tesSUCCESS)

    # CASE 2: Slot is currently owned
    pricePurchased = auctionSlot[sfPrice]
    fractionUsed = (timeSlot + 1) / kAuctionSlotTimeIntervals  # timeSlot is 0-19, so (timeSlot+1)/20
    fractionRemaining = 1 - fractionUsed

    # Calculate computed price based on time slot
    if timeSlot == 0:
        # First interval: simple 5% markup
        computedPrice = pricePurchased * 1.05 + minSlotPrice
    else:
        # Other intervals: decay function
        computedPrice = pricePurchased * 1.05 * (1 - fractionUsed^60) + minSlotPrice

    payPrice = getPayPrice(computedPrice, bidMin, bidMax, lpTokens)
    if payPrice is error:
        return (payPrice.error(), False)

    # Calculate refund to previous owner
    refund = fractionRemaining * pricePurchased
    if refund > payPrice:
        # This should never happen
        log "AMM Bid: refund exceeds payPrice"
        return (tecINTERNAL, False)

    # Send refund to previous owner
    result = accountSend(
        sb,
        from = account,  # New bidder pays
        to = auctionSlot[sfAccount],  # Previous owner receives
        amount = toSTAmount(lpTokens.asset(), refund),
    )
    if result != tesSUCCESS:
        log "AMM Bid: failed to refund"
        return (result, False)

    # Update slot with new owner
    burn = payPrice - refund
    result = updateSlot(
        sb,
        ammSle,
        auctionSlot,
        account,
        current,
        discountedFee,
        payPrice,
        burn,
        lpTokens.asset(),
        lptAMMBalance,
        ctx.tx,
    )

    return (result, result == tesSUCCESS)
```

# 3. updateSlot

Updates the auction slot with the new bidder and burns LP tokens.

## 3.1. updateSlot Pseudo-Code

```python
def updateSlot(
        sb: &Sandbox,
        ammSle,                  # AMM ledger entry
        auctionSlot,             # Auction slot object reference
        account: AccountID,      # New bidder account
        current: int,            # Current time in seconds
        fee: int,                # Discounted fee
        price: Number,           # Price paid for the slot
        burn: Number,            # Amount to burn
        lpTokenIssue,            # LP token issue
        lptAMMBalance,           # Total outstanding LP tokens
        tx: Transaction,         # Transaction
        ) -> TER:
    """
    Update auction slot fields and burn LP tokens.
    """

    # Update auction slot fields
    auctionSlot.setAccountID(sfAccount, account)
    auctionSlot.setFieldU32(sfExpiration, current + kTotalTimeSlotSecs)  # +86,400 seconds

    if fee != 0:
        auctionSlot.setFieldU16(sfDiscountedFee, fee)
    elif auctionSlot.isFieldPresent(sfDiscountedFee):
        auctionSlot.makeFieldAbsent(sfDiscountedFee)

    auctionSlot.setFieldAmount(sfPrice, toSTAmount(lpTokenIssue, price))

    if tx.isFieldPresent(sfAuthAccounts):
        auctionSlot.setFieldArray(sfAuthAccounts, tx[sfAuthAccounts])
    else:
        auctionSlot.makeFieldAbsent(sfAuthAccounts)

    # Burn LP tokens
    saBurn = adjustLPTokens(lptAMMBalance, toSTAmount(lpTokenIssue, burn), IsDeposit=False) # helpers.md#22-adjustlptokens

    if saBurn >= lptAMMBalance:
        # This should never happen
        log "AMM Bid: LP Token burn exceeds AMM balance"
        return tecINTERNAL

    result = redeemIOU(sb, account, saBurn, lpTokenIssue, journal)
    if result != tesSUCCESS:
        log "AMM Bid: failed to redeem"
        return result

    ammSle.setFieldAmount(sfLPTokenBalance, lptAMMBalance - saBurn)
    sb.update(ammSle)

    return tesSUCCESS
```

# 4. validOwner

Checks if the current slot owner is valid and the slot is not expired.

## 4.1. validOwner Pseudo-Code

```python
def validOwner(account: AccountID, timeSlot: Optional[int], sb: &Sandbox) -> bool:
    """
    Check if account is a valid auction slot owner.
    Valid range is 0-19 but tailing slot (19) pays MinSlotPrice and doesn't refund
    so check is < 19 instead of <= 19 to optimize.
    """

    # Valid range is 0-19 but the tailing slot pays MinSlotPrice
    # and doesn't refund so the check is < instead of <= to optimize.
    # timeSlot is an optional: test presence separately, since slot 0 is a valid owner.
    return timeSlot is not None and timeSlot < 19 and sb.read(keylet.account(account))
```

# 5. getPayPrice

The `getPayPrice` function reconciles the system-computed price with the bidder's optional `BidMin` and `BidMax` constraints. Bidders use these constraints to protect themselves from price volatility: `BidMin` ensures they pay at least a certain amount (useful when they want to guarantee winning the slot even if the computed price is lower), while `BidMax` sets an upper limit they're willing to pay (preventing overpayment if the computed price is unexpectedly high). The function validates that the computed price falls within the bidder's acceptable range and returns an error if it doesn't, or returns the reconciled pay price if it does.

## 5.1. getPayPrice Pseudo-Code

```python
def getPayPrice(
        computedPrice: Number,
        bidMin: Optional[STAmount],
        bidMax: Optional[STAmount],
        lpTokens: STAmount) -> Expected[Number, TER]:
    """
    Determine pay price from computed price and bid constraints.

    Returns either the price to pay or an error code.
    """

    # Both min/max bid price are defined
    if bidMin and bidMax:
        if computedPrice <= bidMax:
            payPrice = max(computedPrice, bidMin)
        else:
            log "AMM Bid: not in range"
            return tecAMM_FAILED

    # Only bidMin defined
    elif bidMin:
        payPrice = max(computedPrice, bidMin)

    # Only bidMax defined
    elif bidMax:
        if computedPrice <= bidMax:
            payPrice = computedPrice
        else:
            log "AMM Bid: not in range"
            return tecAMM_FAILED

    # Neither defined
    else:
        payPrice = computedPrice

    # Final validation: check if payPrice exceeds LP token holdings
    if payPrice > lpTokens:
        return tecAMM_INVALID_TOKENS

    return payPrice
```

# 6. Price Calculation

## 6.1. Minimum Slot Price

The minimum slot price is always:

```
minSlotPrice = (TotalLPTokens * TradingFee) / 25
```

This is the price paid when:
- No one owns the slot
- The slot has expired
- The slot is in the tailing period (slot 19)

## 6.2. Computed Price for Owned Slots

When the slot is owned and not expired, the computed price depends on the time slot:

**For the first interval (timeSlot = 0):**

```
computedPrice = pricePurchased * 1.05 + minSlotPrice
```

This applies a simple 5% markup to the price the current owner paid.

**For other intervals (timeSlot = 1-18):**

```
fractionUsed = (timeSlot + 1) / 20
computedPrice = pricePurchased * 1.05 * (1 - fractionUsed^60) + minSlotPrice
```

The `fractionUsed^60` creates a decay function that makes the price increase more slowly as time progresses.

**Example:**

If the current owner paid 1,000 LP tokens and we're in slot 5:
```
fractionUsed = (5 + 1) / 20 = 0.3 (30% of time elapsed)
computedPrice = 1,000 * 1.05 * (1 - 0.3^60) + minSlotPrice
computedPrice =~ 1,050 + minSlotPrice
```

The `0.3^60` is essentially 0, so the price is close to the full 105% markup early in the slot period.

## 6.3. Time Slot Calculation

The auction slot is divided into 20 time intervals over 24 hours:

```
kTotalTimeSlotSecs = 86,400 seconds (24 hours)
kAuctionSlotTimeIntervals = 20
Each interval = 86,400 / 20 = 4,320 seconds (~1.2 hours)
```

The time slot is calculated as:

```
timeSlot = (currentTime - slotExpiration + 86,400) / 4,320
```

Valid time slots are 0-19, where:
- 0 = first interval (just won the slot)
- 19 = last interval (tailing slot)

The `ammAuctionTimeSlot()` function returns:
- `None` if the slot has expired (or has an invalid expiration)
- A value 0-19 indicating the current time interval

# 7. Refund Mechanism

When a new bidder wins the slot from a current owner:

1. Calculate the fraction of time remaining:
   ```
   fractionRemaining = 1 - (timeSlot + 1) / 20
   ```

2. Calculate refund to previous owner:
   ```
   refund = fractionRemaining * pricePurchased
   ```

3. Transfer refund (in LP tokens) from new bidder to previous owner via `accountSend()`

4. Burn remaining amount:
   ```
   burn = payPrice - refund
   ```

**Example:**

Current owner paid 1,000 LP tokens and we're in slot 8:
```
fractionUsed = (8 + 1) / 20 = 0.45 (45% of time used)
fractionRemaining = 1 - 0.45 = 0.55 (55% of time remaining)
refund = 0.55 * 1,000 = 550 LP tokens
```

If new bidder pays 1,200 LP tokens:
```
burn = 1,200 - 550 = 650 LP tokens
```

The refund compensates the previous owner for the unused portion of their slot time.

# 8. LP Token Burning

The burn amount (bid price minus refund) is removed from the LP token supply:

1. **Adjust burn amount** for LP token precision:
   ```
   saBurn = adjustLPTokens(lptAMMBalance, burn, IsDeposit=False)
   ```

2. **Validate burn amount** doesn't exceed AMM balance:
   ```
   if saBurn >= lptAMMBalance:
       return tecINTERNAL
   ```

3. **Redeem (burn) LP tokens** from bidder's balance:
   ```
   redeemIOU(sb, account, saBurn, lpTokens.get<Issue>())
   ```

4. **Decrease AMM's LPTokenBalance**:
   ```
   ammSle.setFieldAmount(sfLPTokenBalance, lptAMMBalance - saBurn)
   ```

This effectively distributes value to all remaining LP token holders by reducing the total supply while keeping the pool's assets unchanged.