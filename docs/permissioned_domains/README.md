# Index

- [1. Introduction](#1-introduction)
    - [1.1. Key Concepts](#11-key-concepts)
- [2. Ledger Entries](#2-ledger-entries)
    - [2.1. PermissionedDomain Ledger Entry](#21-permissioneddomain-ledger-entry)
        - [2.1.1. Object Identifier](#211-object-identifier)
        - [2.1.2. Fields](#212-fields)
        - [2.1.3. Pseudo-accounts](#213-pseudo-accounts)
        - [2.1.4. Ownership](#214-ownership)
        - [2.1.5. Reserves](#215-reserves)
    - [2.2. Offer Ledger Entry](#22-offer-ledger-entry)
        - [2.2.1. Domain Field](#221-domain-field)
        - [2.2.2. Hybrid Offer Fields](#222-hybrid-offer-fields)
            - [2.2.2.1. Flags](#2221-flags)
- [3. Transactions](#3-transactions)
    - [3.1. PermissionedDomainSet Transaction](#31-permissioneddomainset-transaction)
        - [3.1.1. Failure Conditions](#311-failure-conditions)
        - [3.1.2. State Changes](#312-state-changes)
    - [3.2. PermissionedDomainDelete Transaction](#32-permissioneddomaindelete-transaction)
        - [3.2.1. Failure Conditions](#321-failure-conditions)
        - [3.2.2. State Changes](#322-state-changes)
- [4. Access Control](#4-access-control)
    - [4.1. Domain Membership](#41-domain-membership)
    - [4.2. Credential Verification](#42-credential-verification)

# 1. Introduction

PermissionedDomains enable credential-based access control for decentralized exchange activity on the XRP Ledger. A domain owner creates a PermissionedDomain specifying which credentials are required, and only accounts holding those credentials can place offers within that domain. This creates segregated order books where trading activity is restricted to authorized participants. Domain restrictions also apply to cross-currency payments that carry a `DomainID`, both the sender and receiver must be in the domain (see [§4.1 Domain Membership](#41-domain-membership)).

Domain offers support all asset types available on the XRP Ledger: XRP, tokens (issued currencies), and MPTs (Multi-Purpose Tokens, which require the `MPTokensV2` amendment). Any trading pair can be restricted to a permissioned domain. Note that domain offers cross only against the permissioned limit order book; automated market maker (AMM) pools are not consulted for domain crossing.[^amm-no-domain] Under the `fixCleanup3_3_0` amendment, AMM liquidity is also excluded from a domain book's quality estimate, so path ranking matches what domain crossing can deliver.[^amm-no-domain-estimate]

For example, a securities exchange creates a PermissionedDomain requiring "accredited_investor" credentials from a regulatory authority. When Alice wants to trade:
1. Domain Setup: ExchangeAccountID submits PermissionedDomainSet with: `AcceptedCredentials=[{Issuer: RegulatorAccountID, CredentialType: "accredited_investor"}]`
2. Alice obtains credential: RegulatorAccountID creates and Alice accepts the credential (see [Credentials documentation](../credentials/README.md))
3. Alice places offer: AliceAccountID submits OfferCreate with `DomainID=ExchangeDomainID`
4. Ledger verification: Checks Alice holds accepted credential from RegulatorAccountID of type "accredited_investor" and not expired
5. Offer placement: Alice's offer is placed in the domain's order book, matching with other domain offers and hybrid offers

The domain owner always has access to their own domain. All other participants must hold valid credentials. Credentials can be revoked (via expiration or deletion), automatically removing access without the domain owner's involvement.

[^amm-no-domain]: AMM pools are not consulted when a book has a domain: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L820-L822)
[^amm-no-domain-estimate]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/paths/BookStep.cpp#L904-L917)

## 1.1. Terminology and Concepts

**Domain Owner**: The account that creates and controls the PermissionedDomain. The owner can update the accepted credentials list or delete the domain. The owner always has access to place offers in their own domain regardless of credentials.

**Domain ID**: A unique 256-bit identifier for the domain, computed as `hash(PERMISSIONED_DOMAIN_NAMESPACE, owner_account, creation_sequence)`. This ID is immutable and used to reference the domain in OfferCreate transactions.

**AcceptedCredentials**: An array (maximum 10 entries) specifying which credentials grant access to the domain. Each entry contains an Issuer and CredentialType. An account holding any credential matching any entry in this array gains access.

**Domain Offer**: An offer created with the Domain field set, placed exclusively in the domain's order book. Only accounts with domain access can create domain offers, and domain offers only match with other domain offers or hybrid offers. Domain offers support all asset types: XRP, tokens, and MPTs.

**Hybrid Offer**: An offer with both the Domain field set and tfHybrid flag enabled. Hybrid offers exist simultaneously in both the domain order book and the open (regular) order book, providing liquidity bridging between permissioned and open markets.

**Open Offer**: A regular offer without the Domain field, placed in the standard open order book. Open offers are accessible to all accounts and match only with other open offers or hybrid offers.

# 2. Ledger Entries

## 2.1. PermissionedDomain Ledger Entry

### 2.1.1. Object Identifier

**Type Code**: `ltPERMISSIONED_DOMAIN` = `0x0082`

**Domain ID Calculation**: `hash(PERMISSIONED_DOMAIN_NAMESPACE, owner_account, creation_sequence)`

The domain ID is computed at creation using the owner's account and the sequence number consumed by the creating transaction. This can be `Sequence`, or its `TicketSequence` when submitted via a Ticket[^pd-seq].

[^pd-seq]: Domain ID and the stored `Sequence` use the transaction's effective sequence (ticket-aware) under the `fixCleanup3_1_3` amendment: [`PermissionedDomainSet.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/permissioned_domain/PermissionedDomainSet.cpp#L113-L115)

### 2.1.2. Fields

| Field Name            | Type      | Required           | Description                                   |
|-----------------------|-----------|--------------------|-----------------------------------------------|
| `Owner`               | AccountID | :heavy_check_mark: | The account that owns this domain             |
| `Sequence`            | UInt32    | :heavy_check_mark: | Sequence consumed by the creating transaction (`Sequence`, or `TicketSequence` if ticketed) |
| `AcceptedCredentials` | Array     | :heavy_check_mark: | Credentials that grant domain access (max 10) |
| `OwnerNode`           | UInt64    | :heavy_check_mark: | Owner directory page index                    |
| `PreviousTxnID`       | Hash256   | :heavy_check_mark: | Previous transaction hash                     |
| `PreviousTxnLgrSeq`   | UInt32    | :heavy_check_mark: | Previous transaction ledger sequence          |

**AcceptedCredentials Array Structure**: Each element is an object containing:
- `Issuer` (AccountID): The credential issuer account
- `CredentialType` (Blob): The credential type identifier (max 64 bytes)

Credentials are sorted by (Issuer, CredentialType) to ensure deterministic storage order.

### 2.1.3. Pseudo-accounts

PermissionedDomain transactions (creating, updating, or deleting domains) do not create pseudo-accounts.

### 2.1.4. Ownership

PermissionedDomain objects are owned by the account specified in the Owner field. The domain appears in the owner's directory via the OwnerNode field. Only the owner can update or delete the domain.

### 2.1.5. Reserves

Creating a PermissionedDomain increases the owner's object count by 1, requiring one owner reserve increment. Deleting the domain decreases the owner count and releases the reserve.

## 2.2. Offer Ledger Entry

### 2.2.1. Domain Field

**Field Name**: `DomainID` (optional, Hash256)

When present on an Offer ledger entry, this field indicates the offer exists in a permissioned domain's order book. The DomainID must reference an existing PermissionedDomain ledger entry.

### 2.2.2. Hybrid Offer Fields

#### 2.2.2.1. Flags

| Flag Name   | Hex Value    | Description                                      |
|-------------|--------------|--------------------------------------------------|
| `lsfHybrid` | `0x00040000` | Offer exists in both domain and open order books |

**AdditionalBooks Field** (Array, optional): Present on hybrid offers, contains references to additional order book directories where the offer appears. Each array element is an object with:
- `BookDirectory` (Hash256): Order book directory hash
- `BookNode` (UInt64): Page index within the directory

Under the `fixCleanup3_2_0` amendment, when a hybrid offer partially crosses on placement, the open-book `BookDirectory` listed here is keyed by the offer's original placement rate, so it shares the same quality (`ExchangeRate`) as the primary domain `BookDirectory`. Before the amendment the open-book directory was keyed from the post-crossing amounts and could differ slightly due to rounding.[^pd-hybrid-rate]

[^pd-hybrid-rate]: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L944-L953)

Under the `fixCleanup3_3_0` amendment, a resting hybrid offer's domain membership is re-validated only while the domain book is being walked. When the owner loses domain access, for example through credential expiry, the open-book entry stays consumable until a domain-book walk encounters the offer and removes it entirely, from both books. Without the amendment, the membership check ran during any book walk, so open-book processing also removed the offer.[^pd-hybrid-eviction]

[^pd-hybrid-eviction]: [`OfferStream.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/tx/paths/OfferStream.cpp#L253-L267), [`OfferHelpers.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/ledger/helpers/OfferHelpers.cpp#L38-L60)

# 3. Transactions

## 3.1. PermissionedDomainSet Transaction

Creates a new PermissionedDomain (when DomainID is omitted) or updates an existing domain's AcceptedCredentials (when DomainID is provided).

| Field Name            |     Required?      | JSON Type | Internal Type | Description                                 |
|-----------------------|:------------------:|:---------:|:-------------:|:--------------------------------------------|
| `TransactionType`     | :heavy_check_mark: |  String   |    UInt16     | Must be `"PermissionedDomainSet"`           |
| `Account`             | :heavy_check_mark: |  String   |   AccountID   | Transaction sender                          |
| `Fee`                 | :heavy_check_mark: |  String   |    Amount     | Transaction fee                             |
| `DomainID`            |                    |  String   |    UInt256    | Domain to update (omit for creation)        |
| `AcceptedCredentials` | :heavy_check_mark: |   Array   |     Array     | Credentials granting domain access (max 10) |

**AcceptedCredentials Array**: Each element must contain:
- `Issuer` (AccountID): Credential issuer
- `CredentialType` (Blob): Credential type (max 64 bytes)

### 3.1.1. Failure Conditions

**Static validation**:
- `temDISABLED`: featurePermissionedDomains or featureCredentials not enabled
- `temARRAY_EMPTY`: AcceptedCredentials array is empty
- `temARRAY_TOO_LARGE`: AcceptedCredentials exceeds 10 entries
- `temINVALID_ACCOUNT_ID`: AcceptedCredentials contains invalid issuer account id
- `temMALFORMED`:
  - AcceptedCredentials contains CredentialType that is empty or exceeds 64 bytes
  - AcceptedCredentials contains duplicate credentials
  - DomainID is all zeros (update case)

**Validation against the ledger view**:
- `tefINTERNAL`: Account does not exist
- `tecNO_ISSUER`: AcceptedCredentials contains issuer that does not exist
- `tecNO_ENTRY`: DomainID provided but domain does not exist (update case)
- `tecNO_PERMISSION`: DomainID provided but Account is not domain owner (update case)

**Validation during doApply**:
- `tefINTERNAL`: Failed to create domain SLE (creation case)
- `tecINSUFFICIENT_RESERVE`: Insufficient reserve for owner count increase (creation case)
- `tecDIR_FULL`: Owner directory is full (creation case)

### 3.1.2. State Changes

**If DomainID is omitted (creation)**:
- `PermissionedDomain` object is **created** with:
  - `Owner`: set to Account
  - `Sequence`: set to the transaction's effective sequence (its `Sequence`, or `TicketSequence` if ticketed)
  - `AcceptedCredentials`: sorted credentials array
  - `OwnerNode`: page index in owner directory
- `Owner`'s owner count is **incremented** by 1
- `DirectoryNode` entry is **added** to owner's directory

**If DomainID is provided (update)**:
- `PermissionedDomain` object is **updated**:
  - `AcceptedCredentials`: replaced with new sorted credentials array
  - `PreviousTxnID` and `PreviousTxnLgrSeq`: updated

## 3.2. PermissionedDomainDelete Transaction

Deletes a PermissionedDomain. Only the domain owner can delete their domain.

| Field Name | Required? | JSON Type | Internal Type | Description |
|------------|:---------:|:---------:|:-------------:|:------------|
| `TransactionType` | :heavy_check_mark: | String | UInt16 | Must be `"PermissionedDomainDelete"` |
| `Account` | :heavy_check_mark: | String | AccountID | Transaction sender (must be domain owner) |
| `Fee` | :heavy_check_mark: | String | Amount | Transaction fee |
| `DomainID` | :heavy_check_mark: | String | UInt256 | Domain to delete |

### 3.2.1. Failure Conditions

**Static validation**:
- `temDISABLED`: featurePermissionedDomains not enabled
- `temMALFORMED`: DomainID is all zeros

**Validation against the ledger view**:
- `tecNO_ENTRY`: DomainID does not exist
- `tecNO_PERMISSION`: Account is not domain owner

**Validation during doApply**:
- `tefBAD_LEDGER`: Unable to remove directory entry

### 3.2.2. State Changes

- `PermissionedDomain` object is **deleted** (removed from ledger)
- `Owner`'s owner count is **decremented** by 1
- `DirectoryNode` entry is **removed** from owner's directory

Note: Deleting a domain does not remove existing offers from the order book. Those offers remain in the ledger but become unfunded for domain payments because domain membership verification fails when the domain no longer exists.

# 4. Access Control

## 4.1. Domain Membership

An account is considered "in domain" if either condition is met:

1. **Owner Access**: Account is the domain owner (specified in PermissionedDomain.Owner field)
2. **Credential Access**: Account holds at least one accepted credential where:
   - Subject = Account
   - Issuer matches one entry in AcceptedCredentials
   - CredentialType matches the same entry in AcceptedCredentials
   - lsfAccepted flag is set (0x00010000)
   - Credential is not expired (checked against parentCloseTime)

Domain membership is checked at:
- OfferCreate preclaim: Offer creator must be in domain
- Payment preclaim: Both payer (Account) and payee (Destination) must be in domain (when DomainID field is present)
- Offer matching: Offer creator must remain in domain (otherwise offer becomes unfunded)

## 4.2. Credential Verification

The ledger verifies domain access using the `accountInDomain()` function:

```
Function: accountInDomain(view, account, domainID)
1. Lookup PermissionedDomain by domainID
2. If account == domain.Owner: return TRUE
3. For each credential in domain.AcceptedCredentials:
   a. Lookup Credential(account, credential.Issuer, credential.CredentialType)
   b. If credential exists AND lsfAccepted is set AND not expired:
      return TRUE
4. Return FALSE
```

**Expiration Check**: Credential expiration is compared against the ledger's `parentCloseTime`. Expired credentials are treated as if they don't exist for domain access purposes. During transaction apply, an expired credential encountered while verifying domain membership is also deleted to reclaim its reserve; under the `fixCleanup3_1_3` amendment, if that deletion fails the transaction halts and returns the propagated error (e.g. `tecINTERNAL`) instead of continuing the membership check.[^pd-expiry-delete]

[^pd-expiry-delete]: [`removeExpired`](https://github.com/XRPLF/rippled/blob/3.3.0/src/libxrpl/ledger/helpers/CredentialHelpers.cpp#L61-L64), [`verifyValidDomain`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/ledger/helpers/CredentialHelpers.cpp#L332-L334)

**Performance**: Verification iterates through the domain's AcceptedCredentials array (max 10 entries), performing one ledger lookup per credential until a valid match is found.

