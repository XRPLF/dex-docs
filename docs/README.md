> [!NOTE]
> This documentation reflects the XRP Ledger release [3.3.0](https://github.com/XRPLF/rippled/tree/3.3.0).

> [!WARNING]
> 🚧 This documentation is **work in progress**

# 1. XRPL Payment System

> [!IMPORTANT]
> This is a technical specification document intended for developers implementing or verifying XRPL payment system behavior. For user-facing documentation and a high-level overview of XRPL features, please visit [https://xrpl.org/docs](https://xrpl.org/docs).

The XRP Ledger is a multi-currency network with a built-in decentralized exchange, and its payment system lets value move across all of those asset types. At its heart is the Payment Engine that figures out how value should travel and then carries out those moves so payments can seamlessly draw on trust lines, MPTs, order books, AMMs, and direct XRP. This document, as outlined in the [scope document](overview/scope.md), guides you through that landscape. It explains the ledger objects and transactions and the coordination between discovering viable routes and executing the actual transfer.

As described in the [motivation section](overview/motivation.md), this specification lays the foundation of the future work on formal verification aspects of XRP Ledger. It should also give new contributors a single place to understand the Payment Engine and the payment system in its entirety. The payment system has lived primarily inside the `xrpld` codebase, so these pages exist to explain the reasoning behind the system as a whole and offer context for its every subsystem.

## 1.1. XRP Ledger Overview

The XRP Ledger is a distributed ledger that uses a consensus protocol to validate and record transactions across a decentralized network of validator nodes.

[Transactions](transactions/README.md) are the mechanism for modifying the XRP Ledger. They are sent by end users to a `xrpld` server using RPC and are propagated to other nodes by a peer-to-peer network. Each transaction passes through three stages to be added to the open ledger by a validator:

- [Preflight](transactions/README.md#31-preflight): initial check on the transaction's basic format and signatures and the first line of defense. Transactions that fail preflight validation are never added to the ledger
- [Preclaim](transactions/README.md#32-preclaim): a more resource-intensive check that looks at the current ledger to verify things like account balances and sequence numbers. Transactions that fail preclaim with `tec` errors are added to the ledger and claim the fee; other preclaim failures are not added
- [Apply](transactions/README.md#33-doapply): the final stage where the transaction's logic is actually executed, and the ledger state is modified

When the ledger closes, these transactions form the proposal for the next consensus round. If validators reach consensus on which transactions to accept, agreed upon transactions become part of the validated ledger.

All interactions with the payment system happen through transactions. For example, a user can send an `OfferCreate` transaction to place an offer. This will, if successful, create an `Offer` *ledger entry* that will store information about this offer on the ledger. Ledger entries can be modified by other transactions. For example, sending a `Payment` transaction may consume an `Offer` ledger entry or modify the `AccountRoot` ledger entry of the sender and the receiver by changing their `Balance` fields.

# 2. Asset Types

XRP Ledger supports the following currencies that can be held and traded between accounts:

- **[XRP](glossary.md#xrp)** 
- **[IOU](glossary.md#iou)**
- **[MPT](glossary.md#mpt)**

Each asset type stores amounts differently on the ledger, which determines what values they can represent:

| Asset Type  | Ledger Entry | Ledger Field | Precision | Can store fractions? | Range |
|-------------|--------------|-----------|-----------|---------------------|-------|
| **XRP** | `AccountRoot` | `Balance` | Exact (1 XRP = 10^6 drops) | No | 0 to 10^17 drops |
| **IOU** | `RippleState` | `Balance` | 15 decimal digits | Yes | ~-10^96 to ~10^96 |
| **MPT** | `MPToken` | `MPTAmount` | Exact | No | 0 to 2^63-1 (0x7FFFFFFFFFFFFFFF) |

## 2.1. XRP

XRP is the native currency on the XRP Ledger network. The balance is tracked in `AccountRoot` ledger entry, which is a representation of a user account.

Aside from being a tradable asset, XRP serves two essential functions in the network. First, all low-level transaction fees (not to be confused with transfer fees) are paid in XRP.[^xrp-fees] When a transaction is processed, the fee is deducted from the sender's XRP balance and destroyed (burned), permanently removing it from circulation. Second, every account must maintain a minimum XRP balance called the reserve.[^xrp-reserve] The reserve requirement increases with each object the account owns on the ledger, such as trust lines, offers, or other entries.

[^xrp-fees]: Transaction fee deduction: [`Transactor.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/Transactor.cpp#L443)
[^xrp-reserve]: Account reserve calculation: [`Fees.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/Fees.h#L37-L46)

## 2.2. IOU

IOU is issued by an account, and the balance is tracked in `RippleState` ledger entry. This is a representation of a bidirectional **[trust line](glossary.md#trust-line)** which keeps the debt balance between two accounts for an IOU. An IOU is identified by the currency code and the account ID of the issuer.

Trust lines establish that two accounts can trade an IOU between them. For example, Alice can issue currency USD. Bob can establish a trust line with Alice for USD and Alice can send Bob 100 USD. Their trust line will reveal that Alice has a balance of -100 and Bob has a balance of 100 USD.

Each side of a trust line can configure QualityIn and QualityOut settings[^quality-fields] to specify a custom exchange rate that a gateway or user wants applied when receiving or sending issued tokens across that trust line. These per-trust-line quality settings are distinct from the issuer's account-level TransferRate,[^transfer-rate] which imposes a network-wide percentage fee whenever the tokens are transferred between third-party accounts (not involving the issuer directly as the sender or recipient).

Issuer accounts can set the RequireAuth flag[^require-auth] to control which trust lines are authorized to receive their IOUs. The DefaultRipple flag[^default-ripple] on an account determines the default rippling behavior for new trust lines: when set the account can serve as an intermediary in cross-currency payments. When not set, trust lines must explicitly enable rippling through the NoRipple flag.

For an explanation about different transactions used to create and modify trust lines see [Trust Lines](trust_lines/README.md). Their usage in payments will be covered in later reading.

[^quality-fields]: Quality fields on RippleState: [`ledger_entries.macro`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/detail/ledger_entries.macro#L284-L288)
[^transfer-rate]: TransferRate field on AccountRoot: [`ledger_entries.macro`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/detail/ledger_entries.macro#L142)
[^require-auth]: RequireAuth flag on AccountRoot: [`LedgerFormats.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/LedgerFormats.h#L130)
[^default-ripple]: DefaultRipple flag on AccountRoot: [`LedgerFormats.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/LedgerFormats.h#L135)

## 2.3. MPT

Multi-purpose token (MPT) is issued by an account and is identified by its `MPTokenIssuanceID`. The balance is tracked in `MPToken` ledger entry which keeps track of the balance for an account.

MPTs differ from trust line tokens in several key ways:
- **Per-token configuration**: Each MPT issuance has its own transfer fee, authorization requirements, and capability flags. Trust line tokens share the issuer's account-level settings (e.g., single `TransferRate` for all tokens issued by that account).
- **OwnerCount behavior**: MPTokens always count toward an account's `OwnerCount` once created. Trust lines only count when in a non-default state (non-zero balance, custom quality, flag set, etc.).
- **Issuer-centric control**: The issuer can lock/unlock individual holders and set per-holder authorization.
- **Balance storage format**: MPTokens store balances as unsigned 64-bit integers (`MPTAmount` field, type `UInt64`) with each holder having a separate positive-only balance in their own `MPToken` entry. Trust lines store a single signed balance (`Balance` field, type `STAmount`) in the shared `RippleState` between two accounts
- **Burning vs balance adjustment**: MPTs are burned (destroyed from circulation) when sent to the issuer or clawed back. The holder's `MPTAmount` decreases and the issuance's `OutstandingAmount` decreases. The issuer never holds a balance of their own MPTs. Trust line tokens are never burned. When sent to the issuer or clawed back, the signed balance on the shared `RippleState` entry adjusts (the amount is transferred from holder to issuer), shifting between positive and negative to reflect the debt relationship.

For details on how MPTs are created and modified please read [MPTs](mpts/README.md). This document also explains the basic mechanics of MPT transfers, but its full integration is explained in later reading. 

# 3. DEX

The decentralized exchange (DEX) is integrated directly into the XRP Ledger protocol to enable multi-currency payments. When a user wants to send one currency but the recipient wants to receive a different currency, the payment system needs a way to convert between them.
Assets can be converted through offers (limit orders in order books) and AMM pools, which can be consumed during payment execution or offer crossing.

## 3.1. Liquidity Sources

Liquidity on the DEX comes from assets that accounts hold and are willing to trade. Accounts can trade from their XRP balance, their IOU balances, and their MPT balances. These assets can be exchanged with each other in any combination - XRP for IOUs, IOUs for MPTs, MPTs for XRP, and so on.

To make these assets available for trading, accounts create offers or deposit assets into AMM pools. 

Offers represent a limit order. For example, an offer created by Alice with `takerPays` 100 USD and `takerGets` 200 XRP means that Alice is willing to sell 200 XRP for 100 USD, or a better exchange rate. *Order book* refers to valid, unused [resting offers](glossary.md#resting-offer) for a pair of assets.

To understand how offers are placed and how they can be consumed as limit orders (through crossing), please read [Offers](offers/README.md) documentation.  

AMM pools hold reserves of two assets and provide liquidity based on a conservation function, automatically adjusting the exchange rate as the pool reserves change.

To understand how AMMs are created, how money is deposited and withdrawn from them, please read [AMMs](amms/README.md). 

Whenever assets are deposited to an AMMs, a mathematical formula is used to determine how many Liquidity Provider Tokens will be awarded to the depositor. AMMs are trying to preserve their ratio of two assets, so a single-asset deposit will be penalized with fewer LP Tokens than a deposit that maintains the ratio. Similarly, withdrawals require depositors to redeem their LP tokens, and the exact amount needed for the withdrawal is calculated.  

[Deposit](amms/deposit.md) and [Withdrawal](amms/withdraw.md) are referenced from the main AMM document, and they provide detailed pseudocode and logic for multi and single-asset deposits and withdrawals. Since these operations often mirror each other, we suggest cross-referencing opposite transactions in two documents to get the full understanding and build intuition behind the inner workings of AMMs.

When traders are using AMMs they are swapping one asset for another using the AMMs. The more they swap, the worse the exchange rate they get. This discrepancy between the AMM's nominal ratio and the quality that the trader receives is called **slippage** in XRPL terminology. Note that this differs from the standard financial definition of slippage, which refers to the difference between the expected price of a trade and the actual execution price due to market movement or insufficient liquidity - an unintended outcome. In XRPL, the price degradation is intentional and deterministic, resulting from the conservation function that governs AMM behavior as the pool's asset ratio shifts. 
Implementations of mathematical functions that calculate the cost of each swap are described in the [Helper Functions](amms/helpers.md) document and this separation mirrors `xrpld` implementation. However, this document still contains information beyond implementation details. It showcases how AMMs retain their ratio during swaps, and contains an important section that showcases AMM's [slippage and quality degradation](amms/helpers.md#313-slippage-and-quality-degradation).   

Helpers document covers another aspect of AMMs: precision and rounding functions. Rounding could cause AMMs to lose value due to losing precision. This document shows how this is circumvented. Helper functions, just like in the code, are referenced from other places in this specification.

[Bidding](amms/bidding.md) is a standalone document covering the auction slot bidding process, price calculation, refund mechanism, and LP token burning.

The integration of offers and AMMs into the Payment Engine for automatic liquidity consumption during cross-currency payments is described in the path finding and flow sections.

## 3.2. Authorizations

XRPL implements several authorization mechanisms that control access to different features and protect accounts from unwanted interactions. These authorization systems serve different purposes and can work together to provide flexible access control.

**IOU RequireAuth**: Accounts issuing IOUs can set the RequireAuth flag to control which trust lines are authorized to receive their tokens. When enabled, the issuer must explicitly authorize each trust line before it can receive IOUs.

**MPT Authorization**: MPT issuances can require authorization through the lsfMPTRequireAuth flag. When enabled, holders must be individually authorized by the issuer (via MPTokenAuthorize transaction) before they can receive MPTs.

**DepositAuth**: Accounts can enable the DepositAuth flag to require authorization for incoming payments. Authorization can be granted through DepositPreauth. This protects accounts from unwanted payments and enables compliance scenarios where only verified senders should be able to send funds.

These authorization mechanisms are covered in their respective documentation sections: [Credentials](credentials/README.md) for credential-based authorization and DepositAuth integration, [Trust Lines](trust_lines/README.md) for IOU RequireAuth, and [MPTs](mpts/README.md) for MPT authorization.

## 3.3. Permissioned DEX

The Permissioned DEX builds on the credential system to enable access-controlled trading on the XRP Ledger. Using credentials, domain owners can create permissioned trading environments through PermissionedDomains. A domain owner specifies which credentials are required for access, creating segregated order books where only accounts holding those credentials can participate.

The system supports three types of offers:
- **Open offers**: Regular offers accessible to all accounts, placed in the standard order book
- **Domain offers**: Offers restricted to a specific domain, placed only in that domain's order book, matching only with other domain offers or hybrid offers
- **Hybrid offers**: Offers that exist in both the domain order book and the open order book, providing liquidity bridging between permissioned and open markets

All asset types supported by XRPL (XRP, IOUs, and MPTs) can be traded in permissioned domains. The domain owner always has access to their own domain regardless of credentials.

[Permissioned Domains](permissioned_domains/README.md) covers domain creation and management, the PermissionedDomain ledger entry, credential verification logic, and the PermissionedDomainSet and PermissionedDomainDelete transactions. It also explains how domain offers and hybrid offers work within the order book system.

# 4. Payments

With three asset types (XRP, IOUs, MPTs), offers in order books, and AMM pools providing liquidity, the challenge is completing payments between accounts - especially when the sender holds one currency and the recipient wants a different one. The payment system solves this through a two-stage process: first discovering viable routes through the network, then executing the payment along those routes.

The first stage is **path finding**. Accounts are connected through trust lines, offers, AMMs, and MPT holdings, creating a network where value can flow through multiple intermediaries and currency conversions. [Path Finding](path_finding/README.md) explains the pathfinding algorithm, how it discovers and ranks potential routes, and the structure of paths that describe where value can flow.

Once paths are discovered, they are passed to the **Payment Engine** for execution. The implementation of the Payment Engine is called **Flow** (sometimes called the "Flow Engine" to distinguish it from the payment system in broader terms). Flow converts paths into **strands** - sequences of executable **steps** that move value from source to destination.

A strand is composed of different step types, each handling a specific operation in the payment route. There are four main step types:
- **DirectStepI**: Transfers IOUs between accounts through trust lines
- **XRPEndpointStep**: Handles XRP transfers at the payment's source or destination
- **MPTEndpointStep**: Handles MPT transfers at the payment's source or destination
- **BookStep**: Converts currencies by consuming liquidity from order books and AMM pools

For example, a payment from Alice (sending USD) to Bob (receiving EUR) might use a strand composed of: DirectStepI (Alice -> USD Issuer via trust line), BookStep (USD -> EUR conversion via order book), DirectStepI (EUR Issuer -> Bob via trust line).

Flow ranks strands by quality and iteratively consumes liquidity from the best available strands until the payment is satisfied or no more liquidity is available. It supports exact amount delivery and partial payments, handling all combinations of asset types while respecting quality limits and spending constraints.

**Quality** represents the effective exchange rate between two assets, expressed as a ratio of output to input (output/input). This includes not just the base exchange rate but also any transfer fees, trust line quality settings, and other costs incurred during the exchange. From the taker's perspective, a lower quality value is better because it means less input is required to obtain a given output. For example, a quality of 0.5 means the taker pays 0.5 units of input for 1 unit of output, while a quality of 2.0 means the taker pays 2 units of input for 1 unit of output. Throughout this documentation, unless otherwise specified, quality comparisons are presented from the taker's perspective where lower quality values indicate better exchange rates.

[Flow documentation](flow/README.md) provides a high-level overview of how Flow operates. It explains the main algorithm, strand evaluation, and how AMMs and domain payments integrate into the system. The document maintains a high-level perspective to explain the overall flow logic. For detailed implementation of each step type - including the specific mechanics of trust line transfers, endpoint transfers, order book and AMM conversions, quality calculations, and liquidity constraints - see [Steps](flow/steps.md).

The path finding and Flow processes described above can be triggered in two different ways: through a Payment transaction or through offer crossing. Both use the same underlying Payment Engine (Flow) to execute value transfers, but they serve different purposes and have different transaction semantics.

## 4.1. Payment Transaction

A Payment transaction is an explicit instruction to transfer value from a source account to a destination account. The sender specifies the amount to send or the amount the destination should receive, and optionally provides paths to guide the payment. Before the Payment Engine executes the payment, path finding is required to discover viable routes through the network (unless the user explicitly provides paths in the transaction). The Payment Engine then uses these paths to find the best route and executes the transfer.

[Payments](payments/README.md) covers the Payment transaction, including direct XRP payments and cross-currency payment execution, and all the validation rules and failure conditions for payment processing.

## 4.2. Offer Crossing

Offer crossing occurs when an OfferCreate transaction is submitted, and the new offer can be immediately matched with existing offers in the order book. Instead of placing the offer on the ledger as a resting offer, the Payment Engine first attempts to "cross" the new offer with compatible existing offers, effectively executing a trade. For offer crossing, two paths are used: a default direct path and an XRP bridge path (auto-bridging, which uses XRP as an intermediary currency to potentially find better rates). If the new offer is fully satisfied through crossing, no resting offer is created. If only partially satisfied, the remainder becomes a resting offer.

[Offers](offers/README.md) covers the principle of offer crossing and how it triggers the Flow engine to execute the payment. 

