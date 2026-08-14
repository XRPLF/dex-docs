# Scope

The **Payment Engine** is a feature of `xrpld` responsible for executing payment transactions and offer
crossings. Additionally, the Payment Engine serves as a component in pathfinding where it is used to evaluate paths. 

Term `Payment Engine` has been used synonymously with the **Flow** feature, an amendment enabled on the XRP Ledger since 2016. 
This specification takes a broader view than Flow alone, as fully understanding the Payment Engine requires discussing 
pathfinding, direct XRP payments, MPT payments, and AMMs.

**Pathfinding** identifies how assets can be transferred between accounts using different types of intermediary steps. 
The Payment Engine provides the logic to rate the quality of a path and carry out these cross-currency payments.

Direct XRP-to-XRP transfers bypass Flow entirely, since they require no pathfinding or order book interaction.
However, these are still relevant to cover, as the code and logic around payment execution often overlap between
XRP-only and cross-currency cases.

This specification also covers additional features of the payment system. The **AMM** amendment introduced automated market
makers as a source of liquidity transparent to the order book. The **MPT** (Multi-Purpose Transaction) amendment,
enabled as of October 2025, further expands the ways payments can be executed and will be
discussed briefly for completeness.

This specification aims to explain the broader system surrounding the Payment Engine, including cross-currency payments,
direct XRP payments, offers, AMMs, and MPT payments. This wider view is necessary because, in practice, both the code
and the requirements for these features are often intertwined.

