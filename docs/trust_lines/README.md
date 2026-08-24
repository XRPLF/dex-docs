# Index

- [1. Introduction](#1-introduction)
    - [1.1. Default State](#11-default-state)
- [2. Ledger Entries](#2-ledger-entries)
    - [2.1. RippleState Ledger Entry](#21-ripplestate-ledger-entry)
        - [2.1.1. Object Identifier](#211-object-identifier)
        - [2.1.2. Fields](#212-fields)
            - [2.1.2.1. Flags](#2121-flags)
        - [2.1.3. Pseudo-accounts](#213-pseudo-accounts)
        - [2.1.4. Ownership](#214-ownership)
        - [2.1.5. Reserves](#215-reserves)
- [3. Transactions](#3-transactions)
    - [3.1. RippleState Transactions](#31-ripplestate-transactions)
        - [3.1.1. TrustSet Transaction](#311-trustset-transaction)
            - [3.1.1.1. Failure Conditions](#3111-failure-conditions)
            - [3.1.1.2. State Changes](#3112-state-changes)
        - [3.1.2. Clawback Transaction](#312-clawback-transaction)
            - [3.1.2.1. Failure Conditions](#3121-failure-conditions)
            - [3.1.2.2. State Changes](#3122-state-changes)

# 1. Introduction

Trust lines are a mechanism that enables the XRP Ledger to support user-issued [IOUs](../glossary.md#iou). They represent bilateral relationships between accounts that establish trust limits and govern the flow of value for specific IOUs.

Think of a trust line as a credit agreement: it defines how much of a particular IOU one account holds from an issuer, along with the terms of that relationship.

For example, suppose Alice is an issuer and Bob wants to hold USD issued by her. Bob would create a trust line, specifying USD as the currency code and Alice's address as the issuer. 
Alice can now send USD to Bob. 
The IOU is identified by currency code `USD` and `Alice's address`. The balance between Alice and Bob is stored on the trust line. 
If Alice sent 10 USD to Bob, her balance would be -10, while Bob's balance would be 10. Please note that `xrpld` implementation may not store the balance as such, but the user will see it that way. 
Alice can keep issuing the same IOU to other parties, and her total balance would be the sum of her balance on each of the trust lines.

If Alice's account contains `RequireAuth` flag, then her trust lines have to be authorized. This means that, after Bob creates a trust line, Alice has to authorize it before Bob can receive any IOUs on that trust line. Alice authorizes the trust line by sending a TrustSet transaction with the `tfSetfAuth` flag, specifying Bob's address in the `LimitAmount.issuer` field. Alice may choose to freeze or deep freeze the trust line. She can also clawback IOUs from a trust line.

Trust lines have a concept of QualityIn and QualityOut. This is covered in [Cross Currency Payments section](../payments/README.md#42-cross-currency-payment-execution) and [trust line quality in DirectStepI](../flow/steps.md#221-quality-implementation). For the sake of manipulating `RippleState` ledger entry, it is important to know that a quality of `1,000,000,000` is the default quality (QUALITY_ONE). This represents a 1:1 transfer rate, meaning the full amount is transferred without adjustment during cross-currency payments through this trust line.

Trust line limits define the maximum amount of an IOU an account is willing to hold. The `LimitAmount` field in the `TrustSet` transaction specifies this maximum. A limit of 0 means the account will not accept any incoming IOUs on that trust line. Trust line limits are soft limits - they can be exceeded during offer crossing, as creating an offer is considered explicit consent to receive IOUs. 
See [DirectIOfferCrossingStep](../flow/steps.md#23-directioffercrossingstep-offer-crossing-specific-implementation) for implementation details.

Payments on the XRP Ledger often need to flow through intermediate accounts to reach the destination. For example, if Alice wants to pay Bob in USD and both hold trust lines to the same issuer, the payment flows through the issuer: Alice's balance on her trust line with Issuer decreases, and Bob's balance on his trust line with Issuer increases. When an account other than the issuer sits between two trust lines for the same currency, the payment can also flow through that account, entering on one trust line and exiting on another. This is called **rippling**.

The NoRipple flag (`lsfLowNoRipple` / `lsfHighNoRipple`) is a per-account, per-trust-line flag that controls whether a trust line can be used for rippling. A payment is blocked from rippling through an account only when that account has NoRipple set on **both** the trust line the payment enters on and the trust line it exits on. If the account has NoRipple cleared on at least one of the two trust lines, the payment can flow through.

On a newly created trust line, each side's NoRipple flag is initialized independently. The account that submits the `TrustSet` controls its **own** side: NoRipple is set there only if that transaction includes `tfSetNoRipple`. The **counterparty's** side is set automatically when the counterparty's account does not have `lsfDefaultRipple` (the account-level flag set via AccountSet's `asfDefaultRipple`). Because issuers set `DefaultRipple`, a holder opening a trust line to an issuer leaves the issuer's side clear, so the both-sides condition is never met and payments can always ripple through the issuer. A regular holder, by contrast, is not protected automatically: to stop payments from rippling through their own account (for example across USD.IssuerA and USD.IssuerB), the holder must set `tfSetNoRipple` on each line so that both sides carry NoRipple.

NoRipple is checked during both [path finding](../path_finding/README.md) and [payment execution](../flow/steps.md#215-check-implementation). Path finding uses NoRipple as a heuristic filter to avoid exploring paths that would be rejected. The flow engine enforces it as a hard constraint, failing the strand with `terNO_RIPPLE` when violated. See [trust line creation](#3112-state-changes) for how NoRipple flags are initialized.

## 1.1. Default State

A trust line is in **default state** when both accounts have all their parameters set to default values. The default state is important because:
- Trust lines in default state are automatically deleted to reduce ledger bloat
- Default state determines whether an account must pay a reserve for its side of the trust line
- Attempting to create a trust line in default state fails with `tecNO_LINE_REDUNDANT`

**Default values for an account's side of a trust line:**

- **QualityIn**: 0 or absent (equivalent to QUALITY_ONE = 1,000,000,000)
- **QualityOut**: 0 or absent (equivalent to QUALITY_ONE = 1,000,000,000)
- **NoRipple flag**: set if the account does **not** have `lsfDefaultRipple`; cleared if it does
- **Freeze flag**: not set
- **Limit**: 0
- **Balance**: 0 or negative from the account's perspective (meaning the account owes IOUs rather than holds them) 

Both the low and high accounts must have their parameters in the default state for the trust line to be considered in the default state and eligible for deletion.

**Issuer vs Holder perspective:**

- **Issuer** (negative balance): Has limit = 0, balance <= 0. The issuer's side is typically in default state when no IOUs have been redeemed yet.
- **Holder** (positive balance): Has limit > 0, balance >= 0. The holder's side requires a non-zero limit to receive IOUs, so it's not in default state while the trust line is usable.

When a TrustSet transaction is processed and the trust line would transition to default state, the `RippleState` object is deleted, and both accounts' owner directories are updated to remove the trust line entries.

# 2. Ledger Entries

```mermaid
graph LR
    A[Account A<br/>Low Account]
    B[Account B<br/>High Account]
    R[RippleState<br/>Currency: USD<br/>Balance: -10]

    A -->|lowLimit<br/>lowQualityIn/Out<br/>LowNode| R
    B -->|highLimit<br/>highQualityIn/Out<br/>HighNode| R

    R -.->|Balance: -10<br/>A owes B<br/>A is issuer| A
    R -.->|Balance: +10<br/>B holds IOUs<br/>B is holder| B
```

## 2.1. RippleState Ledger Entry

A single `RippleState` ledger entry represents the trust line relationship between two accounts for a specific **currency code**. `RippleState` stores two account IDs in canonical order: the first is always the account ID sorted lower (the **low account**), and the second is the account ID sorted higher (the **high account**). Please see [xrpl.org](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/ripplestate#high-vs.-low-account).

**Storage vs User Perspective:**

- **What's stored**: One `RippleState` entry with a single `sfBalance` field from the low account's perspective
- **What users see**: Each account sees their own balance on the trust line, which is the inverse of their peer's balance
- **Example**: If `sfBalance = -10` (stored), low account sees `-10` and high account sees `+10`

**Issuer** is not defined by `RippleState`. The issuer is conceptually the entity that created (issued) the IOU, but nothing in `RippleState` stores that information explicitly. The issuer is determined by the balance direction: the account with a negative balance (owing IOUs) is issuing, while the account with a positive balance is holding (has
redeemed IOUs). When the balance is `0`, either party can potentially issue IOUs, limited by the other party's trust line limit. The flow engine determines the issuer contextually based on the debt direction during payment execution. See [DirectStepI debt direction](../flow/steps.md#211-revimp-implementation) for details.

The `sfBalance` field in `RippleState` is stored from the low account's perspective:

- Positive balance = Low account holds currency (owes nothing, has credit)
- Negative balance = High account holds currency (low account owes money)

### 2.1.1. Object Identifier

The key of the `RippleState` object is the result
of [SHA512-Half](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#hashes) of the following values
concatenated in order:

- The `RippleState` space key `0x0072` (lowercase `r`)
- The `AccountID` of the *low* account.
- The `AccountID` of the *high* account.
- The 160-bit `Currency` code.

*Low* account is the account with a lower `ID` than the *high* account, ensuring that a trust line between any two
accounts is always represented by the same `RippleState`.

### 2.1.2. Fields

Please
see [RippleState Fields](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/ripplestate#ripplestate-fields)

#### 2.1.2.1. Flags

Please
see [RippleState Flags](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/ripplestate#ripplestate-flags)

### 2.1.3. Pseudo-accounts

Trust lines can be created or modified with certain pseudo-accounts as the destination:

- **AMM accounts** (has `sfAMMID`): Can create new trust lines for the AMM's LP token, or modify existing trust lines
- **Vault accounts** (has `sfVaultID`) and **LoanBroker accounts** (has `sfLoanBrokerID`): Can only modify existing trust lines; attempting to create a new one fails with `tecNO_PERMISSION`
- **Other pseudo-accounts**: Cannot create or modify trust lines (fails with `tecPSEUDO_ACCOUNT`)

The TrustSet transaction never creates, deletes, or modifies the pseudo-account itself - it only creates or modifies
the `RippleState` entry that represents the trust line relationship.

### 2.1.4. Ownership

A `RippleState` ledger entry is jointly owned by both participating accounts. When a trust line is created, it is added to the owner directory of both the low account and the high account.

The `RootIndex` of each account's owner `DirectoryNode` is the result of [SHA512-Half](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#hashes) of the following values concatenated in order:

- The `OwnerDirectory` space key `0x004F` (uppercase `O`)
- The `AccountID` of the account

The `sfLowNode` and `sfHighNode` fields in the `RippleState` entry store the directory page indices where the trust line appears in each account's owner directory. These serve as deletion hints for efficient removal when the trust line is deleted.

### 2.1.5. Reserves

Every non-default trust line increments the account's `OwnerCount`, which raises its reserve requirement. The `TrustSet` transaction enforces that incremental reserve only when the account already owns two or more objects; while it owns fewer than two, a new trust line is allowed even if the account's balance would not cover the extra reserve (this lets a gateway fund new users cheaply).

An account's side of a trust line requires a reserve when any of the following are in a non-default state:

- **QualityIn** is set (non-zero)
- **QualityOut** is set (non-zero)
- **NoRipple flag** differs from the account's default ripple setting
- **Freeze flag** is set
- **Limit** is non-zero
- **Balance** is positive (account holds IOUs)

When a trust line side transitions from default to non-default state, the `lsfLowReserve` or `lsfHighReserve` flag is
set and the account's `OwnerCount` is incremented. When all parameters return to default state, the reserve flag is
cleared and `OwnerCount` is decremented.

Under the `Sponsor` amendment, the reserve for the source account's side of a trust line can be covered by a reserve sponsor. Each side records its own sponsor in the `LowSponsor` or `HighSponsor` field of the `RippleState` entry. Only the transaction's own account's side can be sponsored, never the counterparty's. The reserve waiver for accounts owning fewer than two objects does not apply to a sponsored transaction. When a sponsored side returns to default state, the reserve release is accounted against the recorded sponsor and the field is removed. Trust lines created implicitly during payment execution and offer crossing are never sponsored. The `SponsorshipTransfer` transaction can start, reassign, or end sponsorship of a side that currently holds a reserve. The sponsorship mechanism is described in the [transactions documentation](../transactions/README.md).[^tl-sponsor]

[^tl-sponsor]: [`ledger_entries.macro`](https://github.com/XRPLF/rippled/blob/3.3.0/include/xrpl/protocol/detail/ledger_entries.macro#L280-L294), [`TrustSet.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/token/TrustSet.cpp#L319-L331), [`TokenHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/ledger/helpers/TokenHelpers.cpp#L1480-L1497), [`SponsorshipTransfer.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/sponsor/SponsorshipTransfer.cpp#L238-L295)

# 3. Transactions

## 3.1. RippleState Transactions

### 3.1.1. TrustSet Transaction

The `TrustSet` transaction creates, modifies or deletes a `RippleState` ledger entry and manages the corresponding `DirectoryNode` entries.

When a trust line is created or deleted, it affects the owner directories of both participating accounts. Each account maintains an owner directory that tracks all ledger objects it owns, including its trust lines. The `sfLowNode` and `sfHighNode` fields in the `RippleState` entry store the page numbers where the trust line appears in each account's owner directory

| Field Name        |     Required?     | Modifiable? | JSON Type | Internal Type | Default Value | Description                                                                                                                         |
|-------------------|:-----------------:|:-----------:|:---------:|:-------------:|:-------------:|:------------------------------------------------------------------------------------------------------------------------------------|
| `TransactionType` | :heavy_checkmark: |    `No`     | `String`  |   `UINT16`    |               | The transaction type. Must be `TrustSet`                                                                                            |
| `LimitAmount`     | :heavy_checkmark: |    `Yes`    | `Object`  |   `Amount`    |               | Object defining the currency and the peer account when manipulating a trust line                                                    |
| `QualityIn`       |                   |    `Yes`    | `Number`  |   `UINT32`    |    `None`     | Quality ratio applied when receiving IOUs on this trust line. When absent or 0, defaults to QUALITY_ONE (1,000,000,000 = 1:1 ratio) |
| `QualityOut`      |                   |    `Yes`    | `Number`  |   `UINT32`    |    `None`     | Quality ratio applied when sending IOUs on this trust line. When absent or 0, defaults to QUALITY_ONE (1,000,000,000 = 1:1 ratio)   |
| `Flags`           |                   |    `Yes`    | `String`  |   `UINT32`    |      `0`      |                                                                                                                                     |

The `LimitAmount` field specifies the maximum amount of the IOU the account is willing to hold. It is a composite field containing the currency code, the peer account, and the limit value. Setting the value to 0 means the account will not accept any incoming IOUs on that trust line.

Counterintuitively, `LimitAmount.issuer` is not always the issuer account, even in the business logic sense. For
example, if the issuer wants to authorize a trust line, they will send their account id in `account` and **holder's**
account id as the `issuer`.

For flags, please refer
to [TrustSet Flags](https://xrpl.org/docs/references/protocol/transactions/types/trustset#trustset-flags).

**Terminology in this document**

- *Source account* is the account specified as `account` in the transaction.
- *Destination account* is the account specified as `issuer` field in `LimitAmount`.

#### 3.1.1.1. Failure Conditions

**Static validation:**

- `temINVALID_FLAG`: one of the specified flags is not one of [flags](#2121-flags).
- `temINVALID_FLAG`: flags contain `tfSetDeepFreeze` or `tfClearDeepFreeze` and [DeepFreeze amendment](https://xrpl.org/resources/known-amendments#deepfreeze) is not enabled.
- `temBAD_AMOUNT`: `LimitAmount` is XRP and mantissa is bigger than `100000000000000000ull`. This is a defensive `isLegalNet` check; in practice an XRP `LimitAmount` fails with `temBAD_LIMIT` (below).
- `temBAD_LIMIT`: `LimitAmount` is XRP.
- `temBAD_CURRENCY`: `currency` field in `LimitAmount` is `XRP`.
- `temBAD_LIMIT`: `value` field in `LimitAmount` is less than `0`.
- `temDST_NEEDED`: `issuer` field in `LimitAmount` is not specified or it represents a [noAccount](transactions/README.md#noAccount).

**Validation against the ledger view:**

- `terNO_ACCOUNT`: source account does not exist.
- `tefNO_AUTH_REQUIRED`: source account does not have a `lsfRequireAuth` flag set, but the transaction contains `tfSetfAuth` flag.
- `temDST_IS_SRC`: the source account and the destination account (`LimitAmount.issuer`) are the same.
- `tecNO_DST`: the [AMM](https://xrpl.org/resources/known-amendments#amm) or [SingleAssetVault](https://xrpl.org/resources/known-amendments#singleassetvault) amendment is enabled and the destination (issuer) account does not exist.
- `tecNO_PERMISSION`: the destination account has the `lsfDisallowIncomingTrustline` flag set:
    - If the trust line already exists, do not fail (the [fixDisallowIncomingV1](https://xrpl.org/resources/known-amendments#fixdisallowincomingv1) amendment that introduced this exemption was retired, so it applies unconditionally).
- If the destination account is a pseudo-account:
    - `sfAMMID`: destination is an AMM account (has `sfAMMID` field), but the trust line does not already exist between source and AMM.
            - `tecAMM_EMPTY`: AMM has zero LP IOUs - cannot create trust lines to empty AMMs.
            - `tecNO_PERMISSION`: currency in the trust line request does not match the AMM's LP token currency.
            - `tecINTERNAL`: AMM ledger entry cannot be found.
    - `tecNO_PERMISSION`: destination is a Vault account (has `sfVaultID` field) or LoanBroker account (has `sfLoanBrokerID` field) and the trust line does not already exist.
    - `tecPSEUDO_ACCOUNT`: destination is any other type of pseudo-account.
- If [DeepFreeze](https://xrpl.org/resources/known-amendments#deepfreeze) amendment is enabled, validate freeze flag combinations:
    - `tecNO_PERMISSION`: source account has `lsfNoFreeze` flag set and the transaction contains `tfSetFreeze` or `tfSetDeepFreeze`
      flags.
    - `tecNO_PERMISSION`: transaction contains both freeze flags (`tfSetFreeze` or `tfSetDeepFreeze`) and unfreeze flags (
      `tfClearFreeze` or `tfClearDeepFreeze`) - user should not be able to send
      conflicting `set` and `clear` flags in the same instructions.
    - `tecNO_PERMISSION`: transaction results in a trust line having a deep freeze set but a normal freeze cleared - user should not be able to deep freeze a trust line that is not in frozen state.

**Validation during doApply:**

- `tefINTERNAL`: source account does not exist.
- `tecNO_DST`: destination account does not exist.
- `tecNO_PERMISSION`: the user is trying to set `tfSetNoRipple` and the source account's balance on the trust line is negative.
- `tecINSUF_RESERVE_LINE`: user does not have enough balance to cover the reserve and wants to modify an existing trust line, regardless
  of whether they or the counterparty have created the original trust line. For a sponsored reserve, the sponsor has insufficient XRP or the pre-funded sponsorship has no remaining owner-count allowance.
- `tecNO_LINE_INSUF_RESERVE`: user does not have enough balance to cover the reserve and wants to create a new trust line. For a sponsored reserve, the sponsor has insufficient XRP or the pre-funded sponsorship has no remaining owner-count allowance.
- `tecNO_LINE_REDUNDANT`: trust line does not already exist, amount is `0`, and `QualityIn` and `QualityOut` are either not set, or set to their default value (`1,000,000,000`), and if `tfSetfAuth` flag is not set.

#### 3.1.1.2. State Changes

- `RippleState` object is **deleted** if an existing trust line exists when sending `TrustSet` transaction and:
    - If the trust line is in its [default state](#11-default-state) after updating it.
    - If the currency code of the IOU is `XRP`. This is a defensive check; a `TrustSet` with an `XRP` currency is already rejected at preflight (`temBAD_CURRENCY`/`temBAD_LIMIT`), so this deletion branch is not reached in normal flow.
    - When deleted, the trust line is removed from both accounts' owner directories:
      - Removed from low account's owner directory via `dirRemove` using the `sfLowNode` page number.
      - Removed from high account's owner directory via `dirRemove` using the `sfHighNode` page number.
      - If removing the trust line empties a non-root directory page, that page is deleted and the directory chain is repaired.
      - If either removal fails, the transaction returns `tefBAD_LEDGER`.
 

- `RippleState` object is **modified**:
    - If `QualityIn` field was specified in the transaction:
        - If `QualityIn` is non-zero, set the appropriate quality field (`sfLowQualityIn` for low source account, `sfHighQualityIn` for high source account) to the `QualityIn` value. Unlike `QualityOut`, a `QualityIn` equal to `1,000,000,000` (QUALITY_ONE) is stored as-is, not folded to the default.
        - If `QualityIn` is `0` (or absent), clear the appropriate quality field (`sfLowQualityIn` for low source account, `sfHighQualityIn` for high source account).
    - If `QualityOut` field was specified in the transaction:
        - If `QualityOut` != `1,000,000,000`, set the appropriate quality field (`sfLowQualityOut` for low source
          account, `sfHighQualityOut` for high source account) to the `QualityOut` value
        - If `QualityOut` = `1,000,000,000`, clear the appropriate quality field (`sfLowQualityOut` for low source
          account, `sfHighQualityOut` for high source account)
    - If the transaction contains `tfSetNoRipple` flag and not `tfClearNoRipple` flag:
        - If the source account's balance on the trust line is non-negative, set the appropriate NoRipple flag (`lsfLowNoRipple` for low account, `lsfHighNoRipple` for high account)
        - If the source account's balance is negative, the transaction fails with `tecNO_PERMISSION`
    - If the transaction contains `tfClearNoRipple` flag and not `tfSetNoRipple` flag, clear the appropriate NoRipple flag (`lsfLowNoRipple` for low account, `lsfHighNoRipple` for high account)
    - If the transaction contains `tfSetfAuth` flag, set the appropriate authorization flag for the source account's side only (`lsfLowAuth` if the source is the low account, `lsfHighAuth` if it is the high account).
    - Note: if, after applying the above modifications, the trust line is in its [default state](#11-default-state), it is deleted rather than kept as modified (see deletion conditions above)[^modify-then-delete].
    - If account's parameters in a trust line change to non-default values such that it requires reserve but did not
      before:
        - Set appropriate `lsfLowReserve` or `lsfHighReserve` flag
        - If the transaction's reserve is sponsored, record the sponsor in the side's `LowSponsor` or `HighSponsor` field
    - If account no longer requires reserve because its values in a trust line are now default values:
        - Clear appropriate `lsfLowReserve` or `lsfHighReserve` flag
        - The reserve release is accounted against the sponsor recorded on that side, if any, and the sponsor field is removed
    - Only the NoRipple, freeze, authorization, and reserve flag bits are individually set or cleared (as described above); all other stored flag bits are preserved, and `sfFlags` is rewritten only if it changed.


- `RippleState` object is **created**:
    - If an existing trust line for the same IOU does not exist between the two accounts.
    - When created, the trust line is added to both accounts' owner directories:
      - Added to low account's owner directory via `dirInsert`. The page number is stored in `sfLowNode`.
      - Added to high account's owner directory via `dirInsert`. The page number is stored in `sfHighNode`.
      - If either directory is full, the transaction fails with `tecDIR_FULL`.
    - NoRipple flags are initialized on both sides of the trust line[^trustcreate-noripple]:
      - The source account's NoRipple flag (`lsfLowNoRipple` or `lsfHighNoRipple`) is set if the TrustSet transaction contains `tfSetNoRipple` and not `tfClearNoRipple`[^trustcreate-noripple-src].
      - The destination account's NoRipple flag is set if the destination account does **not** have `lsfDefaultRipple` on their account[^trustcreate-noripple-dst]. `lsfDefaultRipple` is an account-level flag set via AccountSet (`asfDefaultRipple`). When an issuer sets `lsfDefaultRipple`, new trust lines are created without NoRipple on the issuer's side, allowing rippling by default.

[^modify-then-delete]: Default state check and deletion after modification: [`TrustSet.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/transactors/token/TrustSet.cpp#L617-L622)
[^trustcreate-noripple]: NoRipple initialization in trustCreate: [`RippleStateHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/RippleStateHelpers.cpp#L264-L281)
[^trustcreate-noripple-src]: Source account NoRipple from transaction flags: [`RippleStateHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/RippleStateHelpers.cpp#L264-L267)
[^trustcreate-noripple-dst]: Destination account NoRipple from lsfDefaultRipple: [`RippleStateHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/RippleStateHelpers.cpp#L277-L281)


- `DirectoryNode` object is **created or modified**:
    - When a trust line is created, it is added to both participating accounts' owner directories.
    - If an owner directory page is full (32 entries), a new page is created and linked to the directory chain.
    - Before the `fixDirectoryLimit` amendment, if creating a new page would exceed 262,144 pages, the transaction fails with `tecDIR_FULL`. With `fixDirectoryLimit` enabled, that cap is removed and a new page can only fail to be created on 64-bit page-number overflow.


- `AccountRoot` object is **modified**:
    - If account now requires reserve but did not before:
        - Increment `sfOwnerCount` by 1, without overflowing ```std::uint32_t```.
    - If account no longer requires reserve:
        - Decrement `sfOwnerCount` by 1, without going below `0`.

### 3.1.2. Clawback Transaction

**Design note** - in `xrpld`, the same transactor implementation of `Clawback` is used to clawback
both [IOUs](../glossary.md#iou) and [MPTs](../glossary.md#mpt). `xrpld`'s `Clawback` implementation of
`Transactor` uses an adaptation of a visitor pattern. Any transaction containing an MPT in `Amount` field will visit
`MPTIssue` implementation, while transactions referring to IOUs in `Amount` field will visit `Issue` implementation.
This allows for the code to be sufficiently separated that even when describing `xrpld` implementation of Clawback we
can discuss two implementations separately.

Transaction fields are described in [Clawback Fields](https://xrpl.org/docs/references/protocol/transactions/types/clawback#clawback-fields).

**Terminology in this document**

- *Issuer* is the account specified as `account` in the transaction.
- *Holder* is the account specified, counterintuitively, as `issuer` field in `Amount`. This is a commonly used
  pattern to use `issuer` to denote a peer's account.

#### 3.1.2.1. Failure Conditions

Static validation

- `temINVALID_FLAG`: any flags, other than universal transaction flags, are specified.
- `temMALFORMED`:
  - For IOU clawback: user provided `Holder` field in the transaction (IOUs use `Amount.issuer` to specify the holder).
  - For MPT clawback: user did NOT provide `Holder` field in the transaction (MPTs require the `Holder` field).
- `temBAD_AMOUNT`: 
  - issuer and holder are the same account (for IOU clawback; an MPT clawback with the same issuer and holder returns `temMALFORMED` instead).
  - `Amount` provided is XRP.
  - `Amount` is not bigger than `0`.

Validation against the ledger view

- `terNO_ACCOUNT`: issuer's or holder's account does not exist.
- `tecAMM_ACCOUNT`: holder's account is an AMM account (has `sfAMMID`) and the [SingleAssetVault](https://xrpl.org/resources/known-amendments#singleassetvault) amendment is not enabled.
- `tecPSEUDO_ACCOUNT`: If `SingleAssetVault` is enabled and holder's account is any pseudo-account.
- `tecNO_PERMISSION`: issuer does not have `lsfAllowTrustLineClawback` or it does have `lsfNoFreeze`. Only
  trust lines of issuers that can be frozen and that allow trust line clawback can be clawed back.
- `tecNO_LINE`: there is no existing trust line between issuer and holder for the specified currency.
- `tecNO_PERMISSION`: the account specified as `account` (issuer) is not actually the issuer based on the trust line
  balance. The true issuer is the account with the negative balance (owing IOUs), not the positive balance (holding
  IOUs). This error occurs when the transaction sender has the accounts reversed.
- `tecINSUFFICIENT_FUNDS`: the holder's available balance (computed via `accountHolds`) is `0` or negative. This is the spendable balance, which can differ from the raw trust-line balance.


#### 3.1.2.2. State Changes

- `RippleState` object is **modified**:
    - Amount specified in `Amount` field is transferred from holder to issuer on the trust line for specified currency
      code. This is done by subtracting the amount from holder's account and adding it to issuer's account. No limits or
      fees are enforced.
        - If the `amount` is bigger than the available balance, only the available balance is moved.
    - If the clawback returns the holder's side of the trust line to its default state (the holder's balance falls from positive to zero, and its limit, qualities, NoRipple, and freeze are already at default), the holder's `lsfLowReserve` or `lsfHighReserve` flag is cleared and the holder's `OwnerCount` is decremented. Only the holder (sender) side is adjusted; the issuer's side is not.


- `RippleState` object is **deleted**:
    - If, after clawing back, the trust line is in its default state, `RippleState` is deleted.
    - When deleted, the trust line is removed from both accounts' owner directories:
      - Removed from low account's owner directory via `dirRemove` using the `sfLowNode` page number.
      - Removed from high account's owner directory via `dirRemove` using the `sfHighNode` page number.
      - If removing the trust line empties a non-root directory page, that page is deleted and the directory chain is repaired.


- `AccountRoot` (`sfOwnerCount`): deleting the `RippleState` does not by itself change any account's `sfOwnerCount`. The only owner-count change a clawback makes is the single holder-side decrement noted above, applied when the holder's side returns to default (whether or not the line is then deleted). The issuer's `sfOwnerCount` is not adjusted by a clawback.
