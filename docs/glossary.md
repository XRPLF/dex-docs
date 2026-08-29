# Glossary

## Amendment

Amendments are new features or other changes to the functional behavior of XRP Ledger.

**Other terms**
- *Feature* - in `xrpld`, used to refer to amendments. E.g.:
  - `bool isFeatureEnabled(featureSingleAssetVault);`

## CLOB

Central Limit Order Book: list of offers for a pair.

**Other terms**
- *Order book* (but we refer to order book as a book containing both CLOB and synthetic AMM offers)
- *Book*
- *LOB*

## Currency

A type of asset in XRP Ledger. There are three different types of assets:

- **XRP**
- **IOU**
- **MPT**

**Other terms**
- *Issue* - in `xrpld`, used to denote any currency.
- *Asset*

**Other meanings**
- *Currency code* - three-letter code of an IOU.

## IOU

Currency issued by an account which balance is tracked in trust lines.

**Other terms**
- *Trust line token*
- *Issue*, in `xrpld`, used as a term for a currency issued by an account, but not MPT (*MPTIssue*).
- *Issued Currency*

## MPT

Multi-purpose token.

**Other terms**
- *MPTIssue* - in `xrpld` used to refer to a wrapper around MPT ID, especially when disambiguating from token (referred to as *Issue*) 

## Offer

A limit order on the XRP Ledger decentralized exchange that specifies the maximum exchange rate at which the creator is willing to trade. An offer is defined by `takerGets` (what the offer creator provides) and `takerPays` (what they want to receive), and will only execute at a rate that is as favorable as, or better than, the rate specified.

**Other terms**
- *Limit order* - traditional financial markets term for the same concept

**Related terms**
- *Resting offer* - an offer that has been placed on the ledger but not yet consumed

## Order Book

A collection of CLOB and AMM synthetic offers for an asset pair.

## Resting Offer

An offer that has been placed in the order book and is waiting to be consumed. 

**Other terms**
- *Sitting offer* - alternative term for the same concept
- *Book offer* - used in some contexts to refer to offers in the order book

## Quality

The exchange rate, calculated as the ratio of input amount to output amount, including cost of transfer fees. For example, if converting 105 USD results in 100 EUR, the quality is 1.05. Lower quality values are better (less input required for the same output).

Quality can represent the exchange rate of individual components (such as a single offer or liquidity source) or the composite exchange rate across multiple components in a path.

In `xrpld`, quality is represented as a `Quality` class that encapsulates the input/output ratio and provides comparison operations for ranking.

## Rippling

The process where IOU payments flow through an intermediary account's trust lines to connect the sender and receiver. For example, if Alice holds USD from Issuer and wants to send to Bob who also trusts Issuer, the payment "ripples" through Issuer's account: Alice -> Issuer -> Bob. An account must not have the NoRipple flag set on a trust line for rippling to occur on that line.

Rippling enables multi-hop IOU payments without requiring direct trust lines between the sender and receiver, as long as they both trust a common issuer or chain of intermediaries.

## Trust Line

Trust Lines are a bidirectional relationship between an issuer of a token and another account.

**Other terms**
- *Trust* - in `xrpld`, used as a noun to describe a trust line. `TrustSet` is used to create a trust line, represented by `RippleState` ledger entry. 
- *Ripple line* - seldomly used in `xrpld`. 
**Related terms**
- *RippleState* - in `xrpld`, name for ledger entry representing a trust line.

## XRP

Native currency in XRP Ledger.

**Other terms**
- *Native* - in `xrpld`, often used to disambiguate a currency as XRP.