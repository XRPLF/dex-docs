# Index

- [1. Introduction](#1-introduction)
  - [1.1. Bidding Process](#11-bidding-process)
- [2. Bidding Logic](#2-bidding-logic)
  - [2.1. Pseudo-Code](#21-pseudo-code)
- [3. Bid Limits (BidMin and BidMax)](#3-bid-limits-bidmin-and-bidmax)
- [4. Price Calculation](#4-price-calculation)
  - [4.1. Minimum Slot Price](#41-minimum-slot-price)
  - [4.2. Computed Price for Owned Slots](#42-computed-price-for-owned-slots)
  - [4.3. Time Slot Calculation](#43-time-slot-calculation)
- [5. Refund Mechanism](#5-refund-mechanism)
- [6. LP Token Burning](#6-lp-token-burning)

# 1. Introduction

The auction slot is a 24-hour privilege that allows an account to trade through the AMM at a discounted fee (1/10th of the regular trading fee). When an AMM is created, the creator automatically receives the auction slot for free (with price set to 0). Subsequently, accounts can bid LP tokens to win the slot from the current holder, and the winning bid amount (minus any refund to the previous owner) is burned from circulation.

AMM pools cannot directly observe external market prices, so when asset prices diverge between the pool and external markets, arbitrageurs must intervene to rebalance the pool. However, traditional arbitrage faces two problems: (1) arbitrageurs must wait until their profit exceeds the trading fee, creating a window where the pool offers suboptimal prices and experiences reduced trading volume, and (2) multiple arbitrageurs compete in a race, reducing their individual success probability.

The auction slot solves this by offering discounted trading fees to the slot owner, enabling them to execute arbitrage immediately without waiting for profits to exceed fees. This eliminates the race and narrows the window of price inefficiency. The slot owner pays for this advantage by bidding LP tokens, which are burned upon winning - this burn reduces total LP token supply while pool assets remain unchanged, effectively distributing the auction proceeds to all LP token holders through increased ownership percentage.

The auction slot lasts 24 hours (86,400 seconds). Since the slot can be taken over by a new bidder at any time during this period, the slot owner may not use the full 24 hours.

If a new bidder takes over the slot before it expires, the previous owner receives a **refund** proportional to the remaining time: `refund = (1 - fractionUsed) * pricePurchased`. This compensates them for the unused portion of their slot time. The slot's lifecycle is tracked using 20 time intervals of ~1.2 hours each (4,320 seconds) to calculate how much time has been used.

**Price Structure**:
- When no one owns the slot or it has expired, bidders pay the minimum slot price: `TotalLPTokens * TradingFee / 25`. If the `TradingFee` is 0, this formula yields 0, so the `fixCleanup3_4_0` amendment adds a floor (see [Minimum Slot Price](#41-minimum-slot-price)).
- When the slot is owned, a new bidder pays the current owner's price plus 5%, plus the minimum slot price. That price stays almost unchanged for most of the slot's 24 hours and drops only near the end. In the slot's last interval, a bidder pays just the minimum slot price. The current owner gets back the unused share of their price, so the refund shrinks steadily while the price stays flat. For example, in a pool with 10,000,000 LP tokens and a 0.1% fee (a minimum slot price of 400), the current owner bought the slot for 10,000 LP tokens. A new bidder then takes it over at different times (values rounded to two decimals):

| Interval | Time since the owner bought the slot | New bidder pays | Refund to the owner | Burned |
|---|---|---|---|---|
| 0 | 0:00-1:12 | 10,900 | 9,500 | 1,400 |
| 1 | 1:12-2:24 | 10,900 | 9,000 | 1,900 |
| 5 | 6:00-7:12 | 10,900 | 7,000 | 3,900 |
| 10 | 12:00-13:12 | 10,900 | 4,500 | 6,400 |
| 15 | 18:00-19:12 | 10,899.98 | 2,000 | 8,899.98 |
| 17 | 20:24-21:36 | 10,881.13 | 1,000 | 9,881.13 |
| 18 | 21:36-22:48 | 10,416.27 | 500 | 9,916.27 |
| 19 | 22:48-24:00 | 400 | 0 | 400 |
| expired | after 24:00 | 400 | 0 | 400 |

The slot owner pays only **1/10th** of the regular trading fee when they trade through the AMM. The slot owner can designate up to 4 additional accounts to share the discounted fee benefit.

## 1.1. Bidding Process

When an account submits an AMMBid transaction, they can optionally specify `BidMin` and `BidMax` constraints to control how much they're willing to pay. The system first calculates a computed price based on whether the slot is currently owned and how much time has elapsed since it was purchased. If no one owns the slot or it has expired, the computed price is the minimum slot price. If the slot is owned, the computed price is the current owner's price plus 5%, plus the minimum slot price, and it stays almost unchanged until close to expiration.

The actual pay price is then determined by reconciling the computed price with the bidder's constraints. If the slot is currently owned and the bid succeeds, the previous owner receives a time-based refund proportional to the remaining slot time - this refund comes from the new bidder's payment. The remaining amount (pay price minus any refund) is burned from the LP token supply, reducing total supply while keeping pool assets unchanged. Finally, the auction slot is updated with the new owner, a fresh 24-hour expiration time, and the discounted trading fee.

# 2. Bidding Logic

The pseudo-code below follows the entire lifetime of a bid: whether the current slot counts as owned, how the price is computed, how the bid limits apply, and what happens to the LP tokens and the slot.

## 2.1. Pseudo-Code

```python
def bid_for_auction_slot(pool, bidder, bid_min=None, bid_max=None, auth_accounts=None):
    # Validated earlier: the pool has LP tokens (else tecAMM_EMPTY) and so does the
    # bidder (else tecAMM_INVALID_TOKENS, also when their LP tokens are frozen).
    # bid_min and bid_max, when given, are positive amounts of this pool's LP token,
    # at most the bidder's LP tokens, below the LP supply, and bid_min <= bid_max.

    # trading_fee is in units of 0.001%
    min_price = pool.lp_token_supply * pool.trading_fee / 100_000 / 25
    current_time = now()
    interval = pool.auction_slot.interval_at(current_time)  # 0-19, or None outside its period

    owned = (pool.auction_slot.owner and account_exists(pool.auction_slot.owner)
             and interval is not None and interval < 19)
    
    if owned:
        fraction_used = (interval + 1) / 20
        repurchase = pool.auction_slot.price * 1.05
        if interval > 0:
            repurchase *= 1 - fraction_used ** 60
        price = repurchase + min_price
        refund = (1 - fraction_used) * pool.auction_slot.price
    else:
        price = min_price
        refund = 0

    if fixCleanup3_4_0 and pool.trading_fee == 0:
        price = max(price, pool.lp_token_supply * 0.00001 / 25)  # see Minimum Slot Price

    if bid_max is not None and price > bid_max:
        fail(tecAMM_FAILED)
    if bid_min is not None:
        price = max(price, bid_min)
    if price > bidder.lp_tokens:
        fail(tecAMM_INVALID_TOKENS)

    if owned:
        transfer_lp_tokens(refund, from_=bidder, to=pool.auction_slot.owner)
        
    burn_lp_tokens(price - refund, from_=bidder)  # rounded down, may be zero

    pool.auction_slot.owner = bidder
    pool.auction_slot.expiration = current_time + 24 hours
    pool.auction_slot.price = price  # the full price, not the burned part
    pool.auction_slot.discounted_fee = pool.trading_fee // 10
    pool.auction_slot.auth_accounts = auth_accounts or []
```

# 3. Bid Limits (BidMin and BidMax)

A bidder can bound the price with the optional `BidMin` and `BidMax` fields. Bidders use these constraints to protect themselves from price volatility: `BidMin` ensures they pay at least a certain amount (useful when they want to guarantee winning the slot even if the computed price is lower), while `BidMax` sets an upper limit they're willing to pay (preventing overpayment if the computed price is unexpectedly high). If the computed price is above `BidMax`, the bid fails with `tecAMM_FAILED`. If it is below `BidMin`, the bidder pays `BidMin` instead.

Under the `fixCleanup3_4_0` amendment, a zero-fee AMM's computed price is first raised to the [minimum slot price floor](#41-minimum-slot-price) before these limits apply.

# 4. Price Calculation

## 4.1. Minimum Slot Price

The minimum slot price is:

```
minSlotPrice = (TotalLPTokens * TradingFee) / 25
```

`TradingFee` here is the fee as a fraction, the integer field value divided by 100,000.

This is the price paid when:
- No one owns the slot
- The slot has expired
- The slot is in the tailing period (slot 19)

Under the `fixCleanup3_4_0` amendment, the slot should never be free. When the trading fee is 0, the price is at least what the formula above gives for the smallest possible fee, 1 fee unit, which is 1/100,000 or 0.001%:

```
minSlotPrice = (TotalLPTokens * TradingFee) / 25
             = (TotalLPTokens * 1/100,000) / 25
             = TotalLPTokens / (100,000 * 25)
             = TotalLPTokens / 2,500,000
```

Before the amendment, a zero-fee pool gave its slot away for nothing. That mattered because LP token holders can later vote the fee up, and the slot holder's discount follows.

## 4.2. Computed Price for Owned Slots

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

The `fractionUsed^60` term keeps the price close to the full 105% for most of the slot and lowers it only near the end.

The [example table](#1-introduction) in the introduction shows the resulting price at different points of the slot.

## 4.3. Time Slot Calculation

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

# 5. Refund Mechanism

When a new bidder wins the slot from a current owner:

1. Calculate the fraction of time remaining:
   ```
   fractionRemaining = 1 - (timeSlot + 1) / 20
   ```

2. Calculate the refund to the current owner:
   ```
   refund = fractionRemaining * pricePurchased
   ```

3. Transfer the refund (in LP tokens) from the new bidder to the current owner

4. Burn remaining amount:
   ```
   burn = payPrice - refund
   ```

**Example:**

The current owner paid 10,000 LP tokens, and a new bidder takes over in interval 5 (the interval 5 row of the [example table](#1-introduction)):
```
fractionUsed = (5 + 1) / 20 = 0.3 (30% of time used)
fractionRemaining = 1 - 0.3 = 0.7 (70% of time remaining)
refund = 0.7 * 10,000 = 7,000 LP tokens
```

The new bidder pays 10,900 LP tokens:
```
burn = 10,900 - 7,000 = 3,900 LP tokens
```

The refund compensates the owner for the unused portion of their slot time.

# 6. LP Token Burning

The burn amount (the price minus any refund) is removed from the bidder's LP tokens and from the pool's total LP token supply, while the pool's assets stay unchanged. This raises the share of the pool held by every remaining LP token holder. The burned amount is rounded down to the precision of the LP token supply (see [adjustLPTokens](helpers.md#22-adjustlptokens)), so it can be slightly less than the price.