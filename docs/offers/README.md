# Index

- [1. Introduction](#1-introduction)
    - [1.1. Offers](#11-offers)
    - [1.2. Offer Crossing](#12-offer-crossing)
        - [1.2.1. Self-Crossing](#121-self-crossing)
        - [1.2.2. Sell vs Buy Offers](#122-sell-vs-buy-offers)
        - [1.2.3. Auto-bridging](#123-auto-bridging)
        - [1.2.4. Creating the Residual Offer](#124-creating-the-residual-offer)
    - [1.3. Rate Calculation](#13-rate-calculation)
        - [1.3.1. TickSize Rounding](#131-ticksize-rounding)
    - [1.4. Offer Deletion](#14-offer-deletion)
    - [1.5. Domain and Hybrid Offers](#15-permissioned-dex)
        - [1.5.1. Domain Offers](#151-domain-offers)
        - [1.5.2. Hybrid Offers](#152-hybrid-offers)
- [2. Ledger Entries](#2-ledger-entries)
    - [2.1. Offer Ledger Entry](#21-offer-ledger-entry)
        - [2.1.1. Object Identifier](#211-object-identifier)
        - [2.1.2. Fields](#212-fields)
            - [2.1.2.1. Asset-Specific Fields](#2121-asset-specific-fields)
            - [2.1.2.2. Domain-Specific Fields](#2122-domain-specific-fields)
            - [2.1.2.3. Flags](#2123-flags)
        - [2.1.3. Pseudo-accounts](#213-pseudo-accounts)
        - [2.1.4. Ownership](#214-ownership)
        - [2.1.5. Reserves](#215-reserves)
    - [2.2. DirectoryNode Ledger Entry](#22-directorynode-ledger-entry)
        - [2.2.1. Object Identifier](#221-object-identifier)
        - [2.2.2. Directory Pages](#222-directory-pages)
        - [2.2.3. Fields](#223-fields)
- [3. Transactions](#3-transactions)
    - [3.1. Offer Transactions](#31-offer-transactions)
        - [3.1.1. OfferCreate Transaction](#311-offercreate-transaction)
            - [3.1.1.1. Failure Conditions](#3111-failure-conditions)
            - [3.1.1.2. State Changes](#3112-state-changes)
        - [3.1.2. OfferCancel Transaction](#312-offercancel-transaction)
            - [3.1.2.1. Failure Conditions](#3121-failure-conditions)
            - [3.1.2.2. State Changes](#3122-state-changes)

# 1. Introduction

Offers are limit orders on the XRP Ledger decentralized exchange (DEX). They execute only at an exchange rate that is as favorable as, or better than, the rate the offer creator specifies.
As part of the decentralized exchange, users can submit offers between any combination of asset types: [XRP](../glossary.md#xrp), [IOUs](../glossary.md#iou), and [MPTs](../glossary.md#mpt). MPTs must have the `lsfMPTCanTrade` flag set on their MPTokenIssuance to be tradable on the DEX. See [MPT Flags](../mpts/README.md#2121-flags) for details on MPT capability flags.

The DEX supports both open order books (accessible to all accounts) and domain-specific order books (restricted to credential holders). During offer crossing and payments, domain offers only match within their domain, while hybrid offers can match in both environments. This Permissioned DEX functionality enables regulated trading for securities, institutional venues, and other scenarios requiring verified participants. See [PermissionedDomains documentation](../permissioned_domains/README.md) and [Domain and Hybrid Offers](#15-permissioned-dex) for details.

When `xrpld` applies an `OfferCreate`, it first invokes the payment [flow engine](../flow/README.md) to try crossing with existing book depth; only any leftover remainder is placed as a resting order.

## 1.1. Offers

An offer is defined in terms of `takerGets` and `takerPays` parameters, which are named from the perspective of the taker - the party accepting the offer. If Alice creates an offer with `takerGets = 100 XRP` and `takerPays = 10 USD`, it means she is offering 100 XRP and wants to receive 10 USD in return.

Alice has to have a positive amount in `takerGets` currency, unless she is the issuer of that asset (IOU issuers can issue trust line tokens on demand; MPT issuers can mint MPTs into circulation). She does not have to have the full amount to cover the offer. If her balance is lower than `takerGets`, the offer may still partially fill.

## 1.2. Offer Crossing

Because offers are limit orders, a new offer is first matched against existing offers on the book during crossing. It can be filled fully, partially, or not at all. Only the unfilled remainder is then placed on the order book as a resting offer.

Crossing is done by calling the [flow engine](../flow/README.md) and passing a set of paths. All crossings will contain
the [default path](../path_finding/README.md#24-default-paths). If neither taker pays or taker gets is XRP, then an additional
`xrpCurrency` path is added to achieve auto-bridging.  

If transaction has `tfPassive` flag, it will only cross offers with strictly better quality than its own.
It will not cross offers of equal quality, making it more likely to remain on the order book.

If the offer is `tfImmediateOrCancel` it will never be placed in the order book. It can be fully or partially filled during crossing, or not filled at all, but `Offer` ledger entry will never be created for it.

If the offer is `tfFillOrKill` it will never be placed in the order book. It can either *fully* fill immediately or fail.

During crossing, the new offer may be fully filled, partially filled, or not filled at all by existing offers in the order book.

```mermaid
---
title: Offer Creation
---
flowchart LR
    A((Offer Transaction)) --> B[Calculate Rate]
    B --> C[Add default path to Paths]
    C --> D{Can be<br/>autobridged?}
    D -->|yes| E[Add XRP currency to Paths]
    D -->|no| F[Try crossing]
    E --> F

    F --> G{Can fully fill?}

    %% Fully filled path
    G -->|yes| X[Fill offer]
    X --> Z((Transaction Done))

    %% NOT fully filled: handle flags and partials
    G -->|no| H{tfImmediateOrCancel?}
    H -->|yes| X
    H -->|no| I{tfFillOrKill?}
    I -->|yes| Z
    I -->|no| J{Can partially fill?}
    J -->|yes| P[Fill partially]
    P --> R[Create offer for remainder]
    J -->|no| R[Create offer for remainder]
    R --> Z
```

**Atomicity:**

The fee and sequence number are applied to the base ledger view by the transactor before offer crossing begins. Both sandboxes below are built over that base view, so the fee is recorded outside of them and persists regardless of which one is applied:
- `sb`: the crossing results, the deletions of offers consumed or removed during crossing, and the new resting offer
- `sbCancel`: the deletion of offers marked for permanent removal during crossing, such as expired, already-unfunded, invalid, no-longer-in-domain, or directly self-crossable offers

When the offer will not be placed (a `tfFillOrKill` offer that cannot fully cross, or a `tfImmediateOrCancel` offer that crosses nothing), `sbCancel` is applied instead of `sb`. This discards the crossing and placement work while keeping the fee and permanent offer cleanup. See [Ledger Views and Sandboxes](../transactions/README.md#5-ledger-views-and-sandboxes) for how sandboxes provide atomic state changes.

### 1.2.1. Self-Crossing

When a new offer would directly cross a resting offer owned by the same account, `xrpld` deletes the old offer instead of executing a trade against it. The old offer is not partially filled, and no assets are transferred between the account and itself.[^self-cross-removal]

The self-cross removal rule applies when all of the following are true:

1. Crossing is evaluating the default, direct path, not an auto-bridged path.
2. The resting offer's quality is equal to or better than the new offer's quality threshold.
3. The resting offer is owned by the account that submitted the new offer.

The relative amounts of the offers are not considered. A smaller new offer can therefore delete one or more larger resting offers in full. Deleting self-owned offers does not reduce the new offer's amount and the new offer may then cross other accounts' liquidity. Any remainder may be placed on the order book normally.

For an offer with the `tfPassive` flag, equal-quality resting offers do not meet the crossing threshold, so an equal-quality self-offer is not deleted by this rule. 

See [`isSelfCross` in the Flow step documentation](../flow/steps.md#531-foreachoffer-pseudo-code) for the implementation logic.

[^self-cross-removal]: Self-cross removal: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L399-L454). Amount-independent behavior with multiple larger resting offers: [`Offer_test.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/test/app/Offer_test.cpp#L3243-L3310)

### 1.2.2. Sell vs Buy Offers

An offer with the `tfSell` flag set is a **sell** offer. An offer without the `tfSell` flag is a **buy** offer.

When selling, the offer will accept more than the specified `takerPays` amount to maximize the sale of `takerGets`. In the `xrpld` implementation, `takerPays` is capped at `STAmount::kMaxNative` for XRP[^cMaxNative], half the maximum representable IOU value (`STAmount::kMaxValue / 2`) for IOUs[^cMaxValue-iou], and half the maximum MPT amount (`kMaxMpTokenAmount / 2`) for MPTs[^maxMPTokenAmount-mpt]. The IOU and MPT amounts are halved to leave room for the transfer fee charged during crossing: an issuer's transfer rate can be as high as 200% (a 2.0 multiplier), so capping at half the maximum keeps the crossed amount representable.[^transfer-rate-max] XRP has no transfer rate, so its cap is not halved.

The following examples demonstrate offer crossing behavior when a new offer is created and crosses with existing offers in the order book.

**Example 1:**

Alice is willing to sell her 20 USD for 100 XRP. She wants to sell all of her 20 USD.
Bob is willing to buy 10 USD for his 100 XRP. He does not want to buy more than 10 USD.

**Offers:**

- Alice's **sell** offer rests on the book first:
    - Taker pays 100 XRP
    - Taker gets 20 USD

- Bob then submits a **buy** offer, which crosses Alice's resting offer:
    - Taker pays 10 USD
    - Taker gets 100 XRP

**Result:**

- Alice will get 50 XRP and pay 10 USD. This is the exchange rate that she offered. She did not manage to sell all of her 20 USD, but the offer was partially filled.
- Bob will get 10 USD and pay 50 XRP. This is a better exchange rate than he offered. He did not buy more than 10 USD that he originally wanted.

**Example 2:**

Alice is willing to buy 100 XRP for her 20 USD. Bob is willing to sell his 100 XRP for 10 USD, and he wants to sell all 100 XRP.

**Offers:**

- Alice's **buy** offer rests on the book first:
    - Taker pays 100 XRP
    - Taker gets 20 USD

- Bob then submits a **sell** offer, which crosses Alice's resting offer:
    - Taker pays 10 USD
    - Taker gets 100 XRP

**Result:**

- Alice will get 100 XRP and pay 20 USD. This is the exchange rate that she offered.
- Bob will get 20 USD and pay 100 XRP. Because he was selling all of his 100 XRP, he got 20 USD for it. He sold his XRP for a better exchange rate than he hoped for.

[^cMaxNative]: XRP maximum native value: [`STAmount.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STAmount.h#L55)
[^cMaxValue-iou]: IOU maximum value halved for transfer rate: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L436-L437)
[^maxMPTokenAmount-mpt]: MPT maximum amount halved for transfer rate: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L440), maximum defined in [`Protocol.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/Protocol.h#L234)
[^transfer-rate-max]: IOU transfer rate capped at 2.0 (200%): [`AccountSet.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/account/AccountSet.cpp#L128-L131)

### 1.2.3. Auto-bridging

Auto-bridging allows offers between two non-XRP currencies to execute through XRP as an intermediate currency.

Auto-bridging is used only when both `takerPays` and `takerGets` are non-XRP currencies. When it applies, the [Flow engine](../flow/README.md) is invoked with two paths:
1. The [default](../path_finding/README.md#24-default-paths) direct path
2. An auto-bridging path with XRP as intermediate (e.g., USD -> XRP -> EUR)[^auto-bridging-path]

The Flow engine evaluates both paths and selects the one(s) providing the best quality, allowing offers to execute through whichever route offers better pricing.

[^auto-bridging-path]: Auto-bridging path construction: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L410-L412)
[^passive-threshold]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L394-L397)

### 1.2.4. Creating the Residual Offer

Before the offer is sent to the Flow engine for crossing, a **quality threshold** is calculated from `takerPays` and `takerGets`. This represents the minimum exchange rate at which the offer can be crossed. If `takerGets` is a non-XRP asset (IOU or MPT) and the offer creator is not the issuer, `takerGets` is adjusted by multiplying it by the issuer's transfer rate to account for transfer fees. The quality threshold is then calculated as `takerPays / adjusted takerGets`. For a passive offer (`tfPassive`), the threshold is then incremented so the offer crosses only strictly-better-quality offers.[^passive-threshold]

The Flow engine returns the amount that was filled. During offer crossing, the offer owner pays transfer fees (see [transfer rates in flow steps](../flow/steps.md#213-quality-functions)). The transfer fee is deducted from the owner's balance but does not reduce the offer's stated amounts. For example, if the original `takerGets` was 100 USD and 50 USD was transferred with a 2% transfer fee, the owner pays 51 USD from their balance, but the remaining offer balance is 50 USD.

The transfer rate is read from the issuer's settings (AccountRoot `TransferRate` field for IOUs, or the MPTokenIssuance `TransferFee` field for MPTs) and applied identically during offer crossing, maintaining consistent offer book semantics across all asset types.

**Calculating the residual offer after partial filling:**

Both calculations preserve the original offer's quality, the `takerGets : takerPays` ratio, so the unfilled remainder rests at the rate the creator specified. The rate used below is `takerGets / takerPays`, the reciprocal of the `takerPays / takerGets` rate defined in [Rate Calculation](#13-rate-calculation).

**Buy offers** (no `tfSell` flag):[^buy-offer-residual]
1. Subtract consumed `takerPays` from original `takerPays`
2. Calculate remaining `takerGets` by multiplying remaining `takerPays` by this rate
3. Round `takerGets` up

**Sell offers** (`tfSell` flag set):[^sell-offer-residual]
1. Subtract consumed `takerGets` from original `takerGets` (accounting for transfer rates)
2. Calculate remaining `takerPays` by dividing remaining `takerGets` by this rate
3. Round `takerPays` down

**Special cases:**
- If the offer is not filled at all, the original offer is recorded on the ledger, unless it is an ImmediateOrCancel or FillOrKill offer (which are never placed) or the account lacks the reserve for a new offer[^offer-reserve]
- If, after partial filling, the signing account no longer has a positive balance in the `takerGets` currency, the remaining offer is not created[^no-balance-no-offer]. This check is skipped when `takerGets` is an MPT and the creator is its issuer (an issuer can supply the MPT without holding a balance), so the residual offer is still created in that case

[^buy-offer-residual]: Buy offer residual calculation: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L527-L533)
[^sell-offer-residual]: Sell offer residual calculation: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L501-L520)
[^no-balance-no-offer]: No balance check after crossing: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L480-L486)
[^offer-reserve]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L835-L847)

## 1.3. Rate Calculation

The exchange rate for an offer is calculated before any crossing, so that potential partial filling does not affect the
intended rate. The rate is calculated as `takerPays / takerGets` (smaller is better for the taker).

Different asset types have different internal representations:
- **XRP**: an integer number of drops, up to `kMaxNative` (9 * 10^18 drops).[^repr-xrp]
- **IOU**: a sign, an exponent (`-96` to `80`), and a mantissa. When non-zero, the mantissa is normalized to the range 10^15 to 10^16-1, always 16 significant decimal digits (a 54-bit value). This fixed-precision mantissa combined with a wide exponent lets an IOU represent both very large and very small amounts at 16 digits of precision.[^repr-iou]
- **MPT**: an unsigned integer, up to `kMaxMpTokenAmount` (2^63 - 1).[^repr-mpt]

`xrpld` implementation normalizes the rate by packing the result into a 64-bit integer[^rate-packing]:
- Upper 8 bits: exponent + 100
- Lower 56 bits: mantissa

`getRate` returns the rate `0` (and the offer is not stored) in three cases: when `takerGets` is zero, when the computed rate rounds to zero (the offer is 'too good' to represent), or when the computation overflows.

[^repr-xrp]: [`STAmount.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STAmount.h#L55)
[^repr-iou]: [`STAmount.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STAmount.h#L47-L53)
[^repr-mpt]: [`Protocol.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/Protocol.h#L234)
[^rate-packing]: [`STAmount.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/protocol/STAmount.cpp#L459-L481)

### 1.3.1. TickSize Rounding

In order to ensure that ranking of offers in order books requires a significant difference between exchange rates,
issuers can set `TickSize` field to their account.

`TickSize` sets the number of significant decimal digits an offer's rate is rounded to. An issuer can set it to 0 (disabled) or to a value from 3 to 16.[^ticksize-range] If `TickSize` is present, the offer's rate is rounded up to that many significant digits.[^ticksize-round] If the two assets' issuers have different `TickSize` values, the smaller is used; if only one is set, that one is used. `TickSize` applies only to IOU sides: XRP and MPTs are integral types and never carry a `TickSize`, so they do not contribute one to the rounded rate.[^ticksize-integral]

After the rate is rounded, one side of the offer is recomputed from the rounded rate, but only when that side is XRP or an IOU (an MPT side is left unchanged):
- For **sell offers** (`tfSell` flag): `takerPays` is recalculated based on the rounded rate, unless `takerPays` is an MPT
- For **buy offers** (no `tfSell` flag): `takerGets` is recalculated based on the rounded rate, unless `takerGets` is an MPT

[^ticksize-range]: [`Quality.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/Quality.h#L97-L98)
[^ticksize-round]: [`Quality.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/protocol/Quality.cpp#L134-L162)
[^ticksize-integral]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L668-L700)

## 1.4. Offer deletion

When creating an offer, the `OfferSequence` field can be optionally specified to cancel an existing offer before creating the new one. This allows atomic replacement of an offer in a single transaction.

If `OfferSequence` is provided:
1. The system looks up the offer with the specified sequence number belonging to the signing account
2. If the offer is found, it is deleted via `offerDelete`
3. If the offer is not found (already consumed or removed), this is **not an error** - the transaction continues
4. Only after the cancellation (if any) does the system proceed with creating the new offer

This mechanism is useful for updating an existing offer without the risk of having both the old and new offers active simultaneously.

A domain or hybrid offer (one that sets `DomainID`) can use `OfferSequence` to cancel the signing account's own regular (non-domain) offer; see [State Changes](#3112-state-changes) for the amendment-gated details.

Direct [self-cross removal](#121-self-crossing) is separate from `OfferSequence`. It happens automatically during crossing and does not require the transaction to identify an offer to cancel.

## 1.5. Permissioned DEX

PermissionedDomains enable credential-based access control for offers. When the `DomainID` field is specified in an OfferCreate transaction, the offer is placed in a domain-specific order book that only domain members can access. See [PermissionedDomains documentation](../permissioned_domains/README.md) for details on domain creation and access control.

**Open Offers** are not a part of any domain.

### 1.5.1. Domain Offers

A **domain offer** is an offer created with the `DomainID` field set. Domain offers are placed exclusively in the domain's order book (separate from the open order book) and can only be created by accounts with domain access (domain owner or credential holders). Domain offers only match with other domain offers and hybrid offers within the same domain, and cannot be consumed by regular (non-domain) payments or offers.[^domain-book-segregation]

### 1.5.2. Hybrid Offers

A **hybrid offer** is an offer created with both the `DomainID` field set AND the `tfHybrid` flag enabled. Hybrid offers exist simultaneously in both the domain order book and the open order book, with a primary entry in the domain book and a secondary entry (via the `AdditionalBooks` field) in the open book. When a hybrid offer is created, it only crosses with offers in the domain book, since the `DomainID` is passed to the flow engine which uses that domain's order book. Once the hybrid offer is resting on the books, it can be consumed by both domain payments/offers (via the domain book entry) and open payments/offers (via the open book entry).[^hybrid-books]

Under the `fixCleanup3_3_0` amendment, a resting hybrid offer's domain membership is re-validated only while the domain book is being walked. Losing domain access, for example through credential expiry, removes the offer during domain-book processing but leaves the open-book entry consumable. Without the amendment, the membership check ran during any book walk, so losing domain access also removed the hybrid offer during open-book processing.[^hybrid-eviction]

[^domain-book-segregation]: [`Indexes.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/protocol/Indexes.cpp#L102-L110)
[^hybrid-books]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L561-L603)
[^hybrid-eviction]: [`OfferStream.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/paths/OfferStream.cpp#L253-L267)

# 2. Ledger Entries

Offers are stored on the ledger using `Offer` ledger entries, which are organized in the order book via `DirectoryNode` entries.

Offers in the order book are indexed using `DirectoryNode` ledger entries, which organize offers by trading pair and quality level. Each offer is referenced in two directories:

1. **Book Directory**: Groups all offers for a specific trading pair (e.g., USD/XRP) at a particular quality level
2. **Owner Directory**: Tracks all ledger objects owned by an account

The book directory key is calculated by hashing the trading pair (asset identifiers: currency+issuer for IOUs, MPT ID for MPTs, plus optional domain ID), then the last 8 bytes are replaced with the 64-bit quality value (exchange rate). This means each quality level gets its own directory node, allowing efficient traversal from best to worst quality.

When an offer is created, it stores references to both its book directory (`sfBookDirectory` field) and owner directory, enabling efficient order book traversal and account object enumeration.

**Order Book Segregation:**

The XRP Ledger maintains separate order book directories based on domain participation[^domain-book-segregation]:

- **Open Order Books**: Standard directories without domain restrictions, computed as `hash(LedgerNameSpace::BookDir, asset_in, asset_out)`. Contains open offers and hybrid offers (via AdditionalBooks references). All accounts can create open offers.

- **Domain Order Books**: Separate directories for permissioned domains, computed as `hash(LedgerNameSpace::BookDir, asset_in, asset_out, domainID)`. Contains domain offers (primary entries) and hybrid offers (primary entries). Only domain members can create offers in domain order books.

- **Hybrid Offers**: Bridge both order books by maintaining a primary entry in the domain book and a secondary entry (via `AdditionalBooks`) in the open book. See [Hybrid Offers](#152-hybrid-offers) for crossing and consumption semantics.

## 2.1. Offer Ledger Entry

### 2.1.1. Object Identifier

The key of the `Offer` object is the result
of [SHA512-Half](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#hashes) of the following values
concatenated in order:

- The `Offer` space key `0x006F` (lowercase `o`)
- The `AccountID` of the signing account.
- OfferCreate transaction sequence number, or `TicketSequence`.

### 2.1.2. Fields

Fields are described
in [Offer Fields](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/offer#offer-fields)

#### 2.1.2.1. Asset-Specific Fields

The `Offer` entry identifies its assets entirely through the `TakerPays` and `TakerGets` Amount fields, each of which embeds the asset:
- **XRP**: an integer amount of drops
- **IOU**: an amount carrying a currency code and issuer
- **MPT**: an amount carrying a 192-bit MPTID

The Offer entry itself does not store separate `TakerPaysCurrency`/`TakerPaysIssuer`/`TakerPaysMPT` (or the `TakerGets` equivalents) fields.[^offer-asset-amounts] Those broken-out asset identifiers live on the book directory root page, not the offer (see [DirectoryNode Fields](#223-fields)). All combinations of distinct XRP, IOU, and MPT assets are supported; an offer cannot pay and receive the same asset.

[^offer-asset-amounts]: [`ledger_entries.macro`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/detail/ledger_entries.macro#L227-L240)

#### 2.1.2.2. Domain-Specific Fields

**DomainID** (UInt256, optional): The domain identifier for domain offers and hybrid offers. When present, the offer is placed in the domain's order book.

**AdditionalBooks** (Array, optional): For hybrid offers only. Contains references to additional order book directories where the offer is also listed (specifically, the open order book). Each element includes:
- `BookDirectory` (UInt256): Order book directory hash for the additional book
- `BookNode` (UInt64): Page index within that directory

This field allows hybrid offers to be discovered and consumed by both domain and open order book traversals.

Under the `fixCleanup3_1_3` amendment, a valid hybrid offer must carry exactly one `AdditionalBooks` entry (the open order book). The `ValidPermissionedDEX` invariant rejects a hybrid offer whose `AdditionalBooks` array is missing, empty, or holds more than one entry. Before the amendment a present-but-empty array slipped through, since only a missing array or more than one entry failed the invariant.[^hybrid-additionalbooks-count]

[^hybrid-additionalbooks-count]: [`PermissionedDEXInvariant.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/invariants/PermissionedDEXInvariant.cpp#L46-L75)

Under the `fixCleanup3_2_0` amendment, when a hybrid offer partially crosses on placement, the open-book `BookDirectory` referenced here is keyed by the offer's original placement rate, so it shares the same quality (`sfExchangeRate`) as the primary domain `BookDirectory`. Before the amendment the open-book directory was keyed from the post-crossing amounts and could differ slightly due to rounding.[^hybrid-open-book-rate]

[^hybrid-open-book-rate]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L944-L953)

#### 2.1.2.3. Flags

Flags are described
in [Offer Flags](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/offer#offer-flags)

### 2.1.3. Pseudo-accounts

Offer transactions (creating or canceling offers) do not create pseudo-accounts. However, offer crossing may modify pseudo-accounts such as AMMs when the offer crosses with AMM liquidity.

### 2.1.4. Ownership

The offer is stored in the ledger and tracked in an Owner Directory owned by the account submitting the OfferCreate transaction. Furthermore, the offer is also tracked in a Book Directory for the specific trading pair and quality level. The Owner Directory page is captured by the `sfOwnerNode` field. The Book Directory is identified by the `sfBookDirectory` field, and the page within that directory is captured by the `sfBookNode` field

### 2.1.5. Reserves

The `Offer` object costs one owner reserve for the account creating it.

If the account has insufficient reserve before placing the offer:
- If the offer crosses with existing offers (meaning some liquidity was consumed), the transaction succeeds even with insufficient reserve, but the unfilled remainder is not placed on the order book[^offer-reserve]
- If the offer does not cross (nothing was consumed), the transaction fails with `tecINSUF_RESERVE_OFFER`

This special behavior allows offers to succeed if they provide immediate value through crossing, even when the account cannot afford to place a standing offer on the order book.

## 2.2. DirectoryNode Ledger Entry

### 2.2.1. Object Identifier

**Book Directory Root Page (Page 0)**:

The first 192 bits are the first 192 bits of [SHA512-Half](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#hashes) of the following values, concatenated in order.[^book-dir-hash] The exact concatenation order depends on the asset types involved:

- **IOU + IOU** (includes XRP): `BOOK_DIR`, `takerPays` currency, `takerGets` currency, `takerPays` issuer, `takerGets` issuer, [`domainID`]
- **IOU + MPT**: `BOOK_DIR`, `takerPays` currency, `takerGets` MPT ID, `takerPays` issuer, [`domainID`]
- **MPT + IOU** (includes XRP): `BOOK_DIR`, `takerPays` MPT ID, `takerGets` currency, `takerGets` issuer, [`domainID`]
- **MPT + MPT**: `BOOK_DIR`, `takerPays` MPT ID, `takerGets` MPT ID, [`domainID`]

The Book directory space key (`BOOK_DIR`) is `0x0042`. For XRP, the currency code is 160 bits of zeros and the issuer is 160 bits of zeros. The `domainID` is included only when specified (for permissioned domains).

[^book-dir-hash]: Book directory hash computation: [`Indexes.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/protocol/Indexes.cpp#L102-L141)

The last 64 bits encode the exchange rate (`takerPays / takerGets`) as a 64-bit value in big-endian format.

**Owner Directory Root Page (Page 0)**:

The key is the result of [SHA512-Half](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#hashes) of the following values, concatenated in order:

- The Owner directory space key (`0x004F`)
- The `AccountID`

**Subsequent Pages (Pages 1+)**:

The key is the result of [SHA512-Half](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#hashes) of the following values, concatenated in order:

- The DirectoryNode space key (`0x0064`)
- The ID of the root DirectoryNode
- The page number (integer 1 or higher)

### 2.2.2. Directory Pages

Each directory can span multiple pages to accommodate large numbers of entries:

- **Maximum entries per page**: 32 entries
- **Maximum pages per directory**: 262,144 pages, unless the `fixDirectoryLimit` amendment is enabled (which removes this cap)[^dir-page-limit]

Pages form a doubly-linked list structure:
- Root page (page 0) serves as the entry point
- `sfIndexNext`: Points to the next page in the chain
- `sfIndexPrevious`: Points to the previous page
- Last page's `sfIndexNext` points to root page (value 0)
- Root's `sfIndexPrevious` points to the last page number
- Subsequent pages have non-sequential keys (each page key is a hash, see [2.2.1](#221-object-identifier)), but they are addressed by a sequential page number (1, 2, 3, ...). `sfIndexNext`/`sfIndexPrevious` store page numbers, and `sfRootIndex` links each page back to the root.[^page-keylet]

**Page Creation**:
- New offers are appended to the last page of the book directory (book directories preserve insertion order). Owner directories instead insert entries in sorted order, not appended.[^dir-append-insert]
- When a page reaches 32 entries, a new page is created and linked to the chain
- If creating a new page would exceed the page limit, the transaction fails with `tecDIR_FULL`. Before `fixDirectoryLimit` this limit is 262,144 pages; with `fixDirectoryLimit` enabled the cap is removed (pages are bounded only by the 64-bit page counter)

**Page Deletion**:
- When the last entry is removed from a non-root page, the page is deleted
- Empty intermediate pages cause the chain to be repaired by updating adjacent pages' links
- The root page is only deleted when the entire directory becomes empty and `keepRoot` is false

[^dir-page-limit]: [`ApplyView.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/ApplyView.cpp#L124-L129)
[^page-keylet]: [`Indexes.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/protocol/Indexes.cpp#L362-L369)
[^dir-append-insert]: [`ApplyView.h`](https://github.com/XRPLF/rippled/blob/3.3.0/include/xrpl/ledger/ApplyView.h#L326-L380)

### 2.2.3. Fields

**Common Fields** (all directories):
- `sfIndexes`: Vector of up to 32 object IDs (uint256 values)
- `sfRootIndex`: Points to the root directory node's key
- `sfIndexNext`: Optional, points to next page number (omitted if no next page)
- `sfIndexPrevious`: Optional, points to previous page number (omitted on root if it's the only page)

**Book Directory Fields** (order books):
- `sfTakerPaysCurrency`: Currency code for takerPays (when asset is an IOU)
- `sfTakerPaysIssuer`: Issuer account ID for takerPays (when asset is an IOU)
- `sfTakerPaysMPT`: MPTID for takerPays (UInt192, when asset is an MPT)
- `sfTakerGetsCurrency`: Currency code for takerGets (when asset is an IOU)
- `sfTakerGetsIssuer`: Issuer account ID for takerGets (when asset is an IOU)
- `sfTakerGetsMPT`: MPTID for takerGets (UInt192, when asset is an MPT)
- `sfExchangeRate`: The quality level encoded as 64-bit integer
- `sfDomainID`: Optional, for permissioned domain offers

Book directories include either currency/issuer fields or MPT fields depending on the asset types involved in the order book. For XRP, the currency code is used without an issuer field.

**Owner Directory Fields**:
- `sfOwner`: The account that owns the objects

# 3. Transactions

## 3.1. Offer Transactions

### 3.1.1. OfferCreate Transaction

Fields are described
in [OfferCreate Fields](https://xrpl.org/docs/references/protocol/transactions/types/offercreate#offercreate-fields)

Flags are described
in [OfferCreate flags](https://xrpl.org/docs/references/protocol/transactions/types/offercreate#offercreate-flags)

Please note that when an `OfferCreate` transaction is placed, it may not necessarily create an [Offer ledger entry](#21-offer-ledger-entry).
Please refer to [offer crossing](#12-offer-crossing) for more information.

#### 3.1.1.1. Failure Conditions

Some errors are handled within [state changes](#3112-state-changes).

For example, an `ImmediateOrCancel` offer that does not immediately get filled will return error code `tecKILLED`.  
From a business logic perspective, this outcome is expected and acceptable. The `tec` prefix in the error code indicates that the transaction did not succeed, but it was still applied to a ledger and may have side effects.

For this reason, certain `tec` outcomes are covered in the [state changes](#3112-state-changes) section of this document.

**Static validation:**

- `temDISABLED`:
    - transaction contains field `DomainID` but [PermissionedDex](https://xrpl.org/resources/known-amendments#permissioneddex) amendment is not enabled
    - either `takerPays` or `takerGets` is an MPT but [MPTokensV2](https://xrpl.org/resources/known-amendments#mptokensv2) amendment is not enabled
- `temINVALID_FLAG`:
    - one of the specified flags is not one of [flags](#2123-flags).
    - flag `tfHybrid` is specified,
      but [PermissionedDex](https://xrpl.org/resources/known-amendments#permissioneddex) amendment is not enabled or
      field `DomainID` is not present in the transaction.
    - both `tfImmediateOrCancel` and `tfFillOrKill` flags are specified.
- `temMALFORMED`: field `DomainID` is present but is all zeros. User should omit the field if they do not want to specify a domain. Enforced under the `fixCleanup3_2_0` amendment.[^domainid-zero]
- `temBAD_EXPIRATION`: `Expiration` field is set to `0`. User should omit the field if they do not want to specify it.
- `temBAD_SEQUENCE`: `OfferSequence` is set to `0`. User should omit the field if they do not want to specify an offer to delete first.
- `temBAD_AMOUNT`: either `takerPays` or `takerGets` specifies XRP, but with mantissa bigger than `100000000000000000ull`.
- `temBAD_OFFER`:
    - both `takerPays` and `takerGets` are XRP amounts.
    - either `takerPays` or `takerGets` have value `0`.
- `temREDUNDANT`: `takerPays` and `takerGets` are the same asset
- `temBAD_CURRENCY`: either `takerPays` or `takerGets` is a non-native asset that uses the XRP currency code.
- `temBAD_ISSUER`: either `takerPays` or `takerGets` contain either an XRP with issuer ID, or an IOU without an issuer ID.

**Validation against the ledger view:**

- `terNO_ACCOUNT`: signing account does not exist
- `tecFROZEN`: either `takerPays` or `takerGets` is an IOU whose issuer has the `lsfGlobalFreeze` flag set. An offer cannot be created for a frozen issuer. For an MPT whose issuance is globally locked, the same check returns `tecLOCKED` instead.[^global-frozen]
- `tecUNFUNDED_OFFER`: signing account does not have a positive balance in `takerGets` currency and it is not the issuer of `takerGets` currency. Partially funding an offer is acceptable. For MPTs, unauthorized accounts (without `lsfMPTAuthorized` flag or without valid domain credentials when MPTokenIssuance has DomainID) are treated as having zero balance.
- `temBAD_SEQUENCE`: `OfferSequence` is equal to or greater than the signing account's next sequence number (you can only cancel an offer with a lower sequence).[^offer-bad-seq]
- `tecEXPIRED`: the `Expiration` field is before the close time of the previously closed ledger. This is unconditional (no longer gated on the DepositAuth amendment).[^offer-expired]
- **IOU-specific validations for `takerPays`** (only applies when `takerPays` is an IOU):
  - `takerPays` issuer account does not exist:
      - `terNO_ACCOUNT`: `tapRETRY` enabled
      - `tecNO_ISSUER`: `tapRETRY` is not enabled.
  - `takerPays` issuer account has a flag `lsfRequireAuth` and there is no trust line between signing account and the issuer account:
      - `terNO_LINE`: `tapRETRY` enabled
      - `tecNO_LINE`: `tapRETRY` is not enabled.
  - `takerPays` issuer account has a flag `lsfRequireAuth` and there is a trust line between signing account and the issuer account, but it is not authorized:[^checkAcceptAsset-noauth]
      - `terNO_AUTH`: `tapRETRY` enabled
      - `tecNO_AUTH`: `tapRETRY` is not enabled.
  - `tecFROZEN`: trust line between signing account and the `takerPays` issuer account is deeply frozen, either on low or high
    side.
- `tecNO_AUTH`: `takerPays` is an MPT and the signing account is not authorized to hold the MPT. Authorization is checked via `lsfMPTAuthorized` flag on the holder's MPToken, or through valid domain credentials if the MPTokenIssuance has a DomainID set. See [DomainID and Authorization](../mpts/README.md#11-domainid-and-authorization) for details.[^checkAcceptAsset-mpt-auth]
- `tecNO_PERMISSION`:
    - `DomainID` is specified but one of the following domain access requirements is not met:
        - The specified domain must exist
        - The offer creator must be in the domain (either the domain owner or hold a valid accepted credential that is not expired)
    - MPT validation failure (see below)
- `tecOBJECT_NOT_FOUND`: MPT validation failure (see below)
- `tecNO_ISSUER`: MPT validation failure (see below)
- `tecLOCKED`: MPT validation failure (see below)

**MPT-specific validations**: When either `takerPays` or `takerGets` is an MPT, the transaction is validated using [`canTrade`](../mpts/README.md#361-cantrade). See [MPT Validation Functions](../mpts/README.md#36-mpt-validation-functions) for complete details on validation logic and error conditions.

[^checkAcceptAsset-noauth]: Unauthorized trust line returns auth errors: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L307)
[^checkAcceptAsset-mpt-auth]: MPT authorization via requireAuth with WeakAuth: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L330-L340), [`View.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/MPTokenHelpers.cpp#L360-L384)
[^domainid-zero]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L99-L101)
[^domain-cancel-regular]: [`PermissionedDEXInvariant.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/invariants/PermissionedDEXInvariant.cpp#L41-L43), [finalize](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/invariants/PermissionedDEXInvariant.cpp#L105-L111)
[^offer-expired]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L222-L228), [doApply](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L651-L657)
[^offer-bad-seq]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L215-L221)
[^offercancel-bad-seq]: [`OfferCancel.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCancel.cpp#L34-L46)
[^fok-killed]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L807-L812), [`features.macro`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/detail/features.macro#L100)
[^ioc-killed]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L815-L825), [`features.macro`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/detail/features.macro#L132)
[^global-frozen]: [`TokenHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/TokenHelpers.cpp#L59-L65)

**Validation during doApply:**

- `tecEXPIRED`: the `Expiration` field is before the close time of the previously closed ledger (re-checked during doApply in case the offer expired after preclaim).

#### 3.1.1.2. State Changes

- `Offer` object is **deleted**:
    - If `OfferSequence` field is specified and the offer with that sequence exists, it is deleted. See [OfferCancel State Changes](#3122-state-changes) for details on the deletion process
    - When the new offer sets `DomainID` (a domain or hybrid offer), the offer cancelled via `OfferSequence` may be the signing account's own regular (non-domain) offer. Under the `fixCleanup3_2_0` amendment this is permitted; before the amendment the `ValidPermissionedDEX` invariant treated the deleted regular offer as a violation and failed the transaction with `tecINVARIANT_FAILED`.[^domain-cancel-regular]
    - During default-path crossing, any existing offer owned by the signing account that would directly cross the new offer is deleted in full, regardless of the relative amounts. See [Self-Crossing](#121-self-crossing)


- `Offer` object is **not created**:
    - If offer is not fully crossed and it was submitted with `tfFillOrKill` flag, fail with `tecKILLED`. (`fix1578` is a retired amendment, so this is unconditional.)[^fok-killed]
    - If offer is not at all crossed and it was submitted with `tfImmediateOrCancel` flag, fail with `tecKILLED`. (`ImmediateOfferKilled` is a retired amendment, so this is unconditional.)[^ioc-killed]
    - If offer is not fully crossed and the signing account cannot cover the reserve of creating an Offer, fail with
      `tecINSUF_RESERVE_OFFER`.
    - If offer cannot be added to the OfferDirectory because it is full, fail with `tecDIR_FULL`.
    - If the offer is partially filled at crossing and the signing account's `takerGets` balance is reduced to 0, the remaining offer is not created (the transaction still succeeds).[^no-balance-no-offer]


- `Offer` object is **created**:
    - When the offer is not fully crossed and none of the not-created conditions above apply, the `Offer` is created with the remaining `takerGets` and `takerPays`.


- `DirectoryNode` object is **created or modified**:
    - When an offer is created, it is added to two directories:
      - **Owner Directory**: Added via `dirInsert` to `keylet::ownerDir(account)`. The page number is stored in the offer's `sfOwnerNode` field.
      - **Book Directory**: Added via `dirAppend` to the book directory for the trading pair and quality level. The directory key is stored in `sfBookDirectory`, and the page number is stored in `sfBookNode`.
    - Each newly created book-directory page (the root page or a subsequent page) is given these fields:
      - **For IOU assets**: `sfTakerPaysCurrency` and `sfTakerPaysIssuer` (when `takerPays` is an IOU), `sfTakerGetsCurrency` and `sfTakerGetsIssuer` (when `takerGets` is an IOU)
      - **For MPT assets**: `sfTakerPaysMPT` (when `takerPays` is an MPT), `sfTakerGetsMPT` (when `takerGets` is an MPT)
      - **Always**: `sfExchangeRate` (the rate value before any crossing), and optionally `sfDomainID`
    - If a directory page is full (32 entries), a new page is created and linked to the directory chain
    - If creating a new page would exceed the page limit, the transaction fails with `tecDIR_FULL` (the 262,144-page cap applies only before the `fixDirectoryLimit` amendment)[^dir-page-limit]


- Order books are **registered** in `OrderBookDB` (if not already present):
    - Trading pair registered for the offer's assets and domain. See [OrderBookDB documentation](../path_finding/README.md#26-orderbookdb) for details.


- `AccountRoot` object is **modified**:
    - If `Offer` is deleted, decrement `sfOwnerCount` by 1, without going below `0`.
    - If `Offer` is created, increment `sfOwnerCount` by 1, without overflowing ```std::uint32_t```.

### 3.1.2. OfferCancel Transaction

Fields are described
in [OfferCancel Fields](https://xrpl.org/docs/references/protocol/transactions/types/offercancel#offercancel-fields)

#### 3.1.2.1. Failure Conditions

**Static validation:**

- `temINVALID_FLAG`: one of the specified flags is not one of common transaction flags
- `temBAD_SEQUENCE`: `OfferSequence` is set to `0`

**Validation against the ledger view:**

- `terNO_ACCOUNT`: signing account does not exist
- `temBAD_SEQUENCE`: `OfferSequence` is equal to or greater than the signing account's next sequence number[^offercancel-bad-seq]

#### 3.1.2.2. State Changes

- `Offer` object is **deleted**:
    - If the offer with `OfferSequence` sequence number exists. If the offer does not exist, the transaction succeeds without deleting anything.
    - When an offer is deleted via `offerDelete`:
      - The offer is removed from its owner directory using `dirRemove(keylet::ownerDir(owner), sfOwnerNode, offer_index, false)`
      - The offer is removed from its book directory using `dirRemove(keylet::page(book_directory), sfBookNode, offer_index, false)`
      - If the offer has `sfAdditionalBooks` (hybrid offers), it is removed from those directories as well
      - If removing the offer empties a non-root directory page, that page is deleted and the directory chain is repaired

- `AccountRoot` object is **modified**:
    - If `Offer` is deleted, decrement `sfOwnerCount` by 1, without going below `0`.
