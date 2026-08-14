# Index

- [1. Introduction](#1-introduction)
  - [1.1. Paths](#11-paths)
    - [1.1.1. Example: Issuing IOUs](#111-example-issuing-ious)
    - [1.1.2. Example: Same Currency IOU](#112-example-same-currency-iou)
    - [1.1.3. Example: Issuing and Redeeming MPTs](#113-example-issuing-and-redeeming-mpts)
    - [1.1.4. Example: MPT Holder to Holder](#114-example-mpt-holder-to-holder)
    - [1.1.5. Example: Different Currencies](#115-example-different-currencies)
    - [1.1.6. Example: Same Currency Code IOU, Different Issuers](#116-example-same-currency-code-iou-different-issuers)
  - [1.2. Path Types](#12-path-types)
  - [1.3. Path Finding](#13-path-finding)
  - [1.4. Algorithm](#14-algorithm)
    - [1.4.1. Setup](#141-setup)
    - [1.4.2. Path Type Expansion](#142-path-type-expansion)
    - [1.4.3. Rank Paths](#143-rank-paths)
  - [1.5. Structure](#15-structure)
- [2. Terminology and Concepts](#2-terminology-and-concepts)
  - [2.1. Terminology](#21-terminology)
  - [2.2. Payment Types](#22-payment-types)
  - [2.3. Path Types](#23-path-types)
    - [2.3.1. Node Types](#231-node-types)
  - [2.4. Default Paths](#24-default-paths)
- [3. Pathfinder](#3-pathfinder)
  - [3.1. Pathfinder Class](#31-pathfinder-class)
  - [3.2. Path Elements](#32-path-elements)
  - [3.3. Path](#33-path)
- [4. Path Discovery](#4-path-discovery)
  - [4.1. findPaths Function](#41-findpaths-function)
  - [4.2. addPathsForType](#42-addpathsfortype)
  - [4.3. addLinks](#43-addlinks)
  - [4.4. addLink](#44-addlink)
  - [4.5. OrderBookDB](#45-orderbookdb)
  - [4.6. AssetCache](#46-assetcache)
- [5. Path Ranking](#5-path-ranking)
  - [5.1. computePathRanks](#51-computepathranks)
  - [5.2. rankPaths](#52-rankpaths)
  - [5.3. getPathLiquidity](#53-getpathliquidity)
- [6. Path Selection](#6-path-selection)
- [7. RPC Requests](#7-rpc-requests)
  - [7.1. `ripple_path_find` RPC (Legacy)](#71-ripple_path_find-rpc-legacy)
  - [7.2. `path_find` RPC](#72-path_find-rpc)
  - [7.3. Source Currency Handling](#73-source-currency-handling)

# 1. Introduction

The XRP Ledger is a network where accounts are connected via [trust lines](../trust_lines/README.md), [offers](../offers/README.md), and [MPTs](../mpts/README.md). To send non-[XRP](../glossary.md#xrp) [currencies](../glossary.md#currency) or to perform currency conversions, payments often cannot go directly from source to destination. Instead, they must find routes through:

- Direct XRP payments, trust lines and MPT payments
- Currency conversions through the decentralized exchange ([CLOBs](../glossary.md#clob) and [AMMs](../amms/README.md))
- Multi-hop paths combining both

**Path finding** discovers viable routes and returns them as **paths**. A path describes a potential route for value to flow, such as "Alice -> USD/EUR order book -> Bob" or "Alice -> USD/XRP order book -> XRP/EUR order book -> Bob". Each element in the path (an account or an order book) represents a location where value can move through. 

Path finding takes into account [domain-restricted order books](../permissioned_domains/README.md) when searching for routes. The permissioned DEX allows users to create domain-specific offers that are only accessible to accounts with valid credentials for that domain. When a domain is specified in a path finding request, the algorithm searches only within that domain's offers. 

The [**Flow engine**](../flow/README.md) then takes these paths and converts them into executable operations called **strands**. A strand is a sequence of **steps**, where each step is a concrete action that moves value between path elements.

Path finding discovers **where** payments can go, while Flow figures out **how** to execute them.

## 1.1. Paths

**Paths** are sequences of intermediate steps that describe a route from source to destination. The examples below show common payment scenarios and the paths they require.

### 1.1.1. Example: Issuing IOUs

Issuer wants to send USD to Alice. Alice has a [trust line](../trust_lines/README.md) to Issuer for USD. The trust line is a direct connection so the path is:

- **Path**: Issuer -> Alice

### 1.1.2. Example: Same Currency IOU

Alice wants to send USD issued by Issuer to Bob. Both Alice and Bob have a trust line to Issuer for USD. The payment has to go through Issuer, because Alice cannot issue a currency in Issuer's name:

- **Path**: Alice -> Issuer -> Bob

### 1.1.3. Example: Issuing and Redeeming MPTs

Holder Alice holds an [MPT](../mpts/README.md) issued by Issuer. The issuer wants to send (mint) additional MPT to Alice:

- **Path**: Issuer -> Alice

If Alice wants to send the MPT back to the issuer (redeeming):

- **Path**: Alice -> Issuer

### 1.1.4. Example: MPT Holder to Holder

Alice wants to send an MPT issued by Issuer to Bob. Both Alice and Bob are holders of the MPT. The payment path goes through the issuer. See [MPT Payment Execution](../payments/README.md#4-payment-execution-paths) for details on how holder-to-holder transfers are processed.

- **Path**: Alice -> Issuer -> Bob

### 1.1.5. Example: Different Currencies

Alice wants to send USD to Bob and have Bob receive an MPT. Alice is not the issuer of USD and Bob is a holder of the MPT issued by MPT Issuer.
Since Alice is paying in a different currency than what Bob will get, the currency has to be exchanged.

There are multiple ways in which Alice can achieve this transfer. For example, if there is a USD/MPT [order book](../glossary.md#order-book) (a set of offers and AMMs that can exchange one currency for another), it could be used to complete the payment.

However, USD/MPT order book may not exist, or it may lack liquidity to perform the whole payment. In that case, two order books could be used - for example USD/XRP and then XRP/MPT - if both exist and have sufficient liquidity.

Two possible routes a payment could take would be:

- **Incomplete Path 1**: Alice -> [USD/MPT order book] -> Bob
- **Incomplete Path 2**: Alice -> [USD/XRP order book] -> [XRP/MPT order book] -> Bob

However, these are not complete paths. Each payment that Alice makes in the issuer's USD has to be reflected on the trust line between her and USD Issuer. To consume the offer, she needs to send USD to the USD issuer, who will in turn send USD to the market maker who created the offer that is consumed (or multiple market makers if multiple offers are consumed).
To make a payment, she has to go through the issuer of USD to whom she has a trust line:

- **Incomplete Path 1**: Alice -> USD Issuer -> [USD/MPT order book] -> Bob
- **Incomplete Path 2**: Alice -> USD Issuer -> [USD/XRP order book] -> [XRP/MPT order book] -> Bob

Bob is a holder (not the issuer) of the MPT, so he also has to receive the payment through the MPT Issuer:

- **Complete Path 1**: Alice -> USD Issuer -> [USD/MPT order book] -> MPT Issuer -> Bob
- **Complete Path 2**: Alice -> USD Issuer -> [USD/XRP order book] -> [XRP/MPT order book] -> MPT Issuer -> Bob

### 1.1.6. Example: Same Currency Code IOU, Different Issuers

Let's say that Alice wants to send USD to Bob. Alice has a USD trust line to Issuer A. Bob has an USD trust line to Issuer B. Issuer A and Issuer B have no trust lines between them, but there is an Exchanger who has USD trust lines to both Issuer A and Issuer B and NoRipple cleared on their trust lines, so payments can ripple through Exchanger. 

This payment can be completed via: 

- **Path**: Alice -> Issuer A -> Exchanger -> Issuer B -> Bob

The payment is [**rippling**](../glossary.md#rippling) through Exchanger. Exchanger is taking an exchange risk between Issuer A's USD and Issuer B's USD value.

## 1.2. Path Types

The XRP Ledger has countless possible payment routes, but exploring all of them would be computationally infeasible. 

**Path types** constrain the search by defining templates for the structure of a route. Each path type specifies the sequence of hop types (accounts, order books, XRP bridges) that a path should follow, and the pathfinder fills in the template with concrete accounts and order books from the ledger.

Each type is a sequence of building blocks:

- **s** (source) - Start at the source account
- **a** (accounts) - Find accounts connected via trust lines or MPT holdings
- **b** (books) - Use an order book to exchange currencies
- **x** (XRP books) - Use an order book that outputs XRP, since XRP frequently serves as a bridge currency between other assets
- **f** (destination book) - Use an order book to get the destination currency
- **d** (destination) - Arrive at the destination account

For example, the type `"sfd"` means: *"Start at source, find an order book to exchange into the destination currency, then deliver to destination."*

A more complex type like `"saxfd"` means: *"Start at source, go through an intermediate account, use an order book to exchange to XRP, then use another order book to exchange to the destination currency, and deliver to destination."*

Path types are predefined for every payment type. For example, XRP->NonXRP has one set of types, while NonXRP->NonXRP has another. See [Section 2.3](#23-path-types) for the full table, node type details, and search configuration.

## 1.3. Path Finding

Given a source and a destination, path finding uses path types to create actual paths that will be used to complete a payment. For example, when expanding `"sfd"`, the `"f"` gets translated into concrete order books - path finding will create a separate path for each feasible order book that can convert to the destination currency. 

Each path represents a different way value can flow from Alice's USD to Bob's EUR, with different exchange rates and liquidity characteristics. A single payment may need multiple paths to complete, because not every path will provide full liquidity. 

It is important to note that the path finding code in `xrpld` is not responsible for returning the full path. For example, in [different currencies example](#115-example-different-currencies), the path finding will only return:

Path 1:

- USD/EUR order book

Path 2:

- USD/XRP order book
- XRP/EUR order book

It is the responsibility of the Flow engine to do the [path normalization](../flow/README.md#51-path-normalization) that will decide how to connect the source to the first element and how to connect the last element to the destination.

Additionally, for IOU and MPT payments, path finding searches to the **effective destination** rather than the final destination. The effective destination is the issuer of the destination amount. 
For example, if Bob is receiving EUR issued by EUR Issuer, path finding only needs to find paths that reach EUR Issuer. Similarly, if Bob is receiving an MPT, path finding finds paths that reach the MPT Issuer. Flow handles the final hop to Bob. For XRP payments, the effective destination is simply the destination account itself.

## 1.4. Algorithm

Path finding is a constrained graph search algorithm that explores the ledger's network to find viable payment routes. We will illustrate the algorithm using an example similar to [section 1.1.5](#115-example-different-currencies): Alice wants to send USD and Bob should receive an MPT.

### 1.4.1. Setup

- Alice has USD trust line to USD Issuer
- Bob is a holder of an MPT issued by MPT Issuer
- Available order books: USD/MPT, USD/XRP, XRP/MPT, USD/CAD, CAD/MPT, USD/JPY, JPY/CHF
- For this example, assume we only have the following path types defined for this payment type: `"safd"`,  `"sbfd"`, `"saxfd"`, `"sabfd"` (in reality, the algorithm could explore more types based on the search depth and payment type)
- The [**default path**](#24-default-paths) is always tested separately and not included in path type expansion
- This is a non-permissioned payment (no domain specified), so all open order books are available for consideration

### 1.4.2. Path Type Expansion

Path type expansion works by incrementally building paths one node at a time. For each node in the path type (like `"s"`, `"a"`, `"f"`, `"d"`), the algorithm:
1. Queries the ledger to find all possible options for that node
2. Creates a new path branch for each option found
3. Adds all branches to a list of incomplete paths
4. Continues to the next node, expanding each path in the list

When a path reaches the destination with the correct currency, it's marked as complete and added to the complete paths list.

**Step 1: Determine Payment Type and Select Path Types**

Payment type is NonXRP->NonXRP (different currencies), and the algorithm selects path types to explore based on the payment type:
- `"safd"`: source -> account -> destination book -> destination
- `"sbfd"`: source -> book -> destination book -> destination
- `"saxfd"`: source -> account -> XRP book -> destination book -> destination
- `"sabfd"`: source -> account -> book -> destination book -> destination

**Step 2: Expand Path Type `"safd"`**

Starting from Alice with USD, the algorithm will expand each node in the path type.

- `"s"`: Start at Alice (USD), creates an empty path: `[]`
- `"sa"`: Query AssetCache for accounts connected via trust lines holding USD, which have enough liquidity and allow rippling.
  - Finds: USD Issuer
  - Incomplete paths so far: `[USD Issuer]`
- `"saf"`: Query OrderBookDB for any order books from Issuer USD to MPT destination
  - Finds: USD/MPT
  - MPT is the destination so adds `[USD Issuer, USD/MPT Book]` to complete paths
- `"safd"`: Has no incomplete paths to examine

**Step 3: Expand Path Type `"sbfd"`**

Starting from Alice with USD:
- `"s"`: Start at Alice (USD)
- `"sb"`: Query OrderBookDB for any order books that accept USD as input. Because no source issuer was specified, the source asset's issuer defaults to Alice herself[^source-issuer-default]. No order books exist for USD.Alice, so no books are found.
  - Terminates (`"sbf"` and `"sbfd"` have no incomplete paths to examine)

[^source-issuer-default]: Source issuer defaults to source account: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L274-L277)

**Step 4: Expand Path Type: `"saxfd"`**
- `"s"`: Start at Alice (USD)
- `"sa"`: Query AssetCache for accounts connected via trust lines holding USD, which have enough liquidity and allow rippling.
    - Finds: USD Issuer
    - Incomplete paths so far: `[USD Issuer]`
- `"sax"`: Query OrderBookDB for any order book that will convert Issuer USD convert to XRP
    - Finds: Issuer USD -> XRP
    - Incomplete paths so far: `[USD Issuer, USD/XRP Book]` 
- `"saxf"`: Query OrderBookDB for any order book that will convert to MPT
    - Finds: Issuer XRP -> MPT
    - MPT is the destination so adds to complete paths: `[USD Issuer, USD/XRP Book, XRP/MPT book]`
- `"saxfd"`: Has no incomplete paths to examine

**Step 5: Expand Path Type: `"sabfd"`**
- `"s"`: Start at Alice (USD)
- `"sa"`: Finds that `"sa"` has already created an incomplete path `[USD Issuer]` (in step 4)
- `"sab"`: Query OrderBookDB for any order book that will convert from Issuer USD
    - Finds: Issuer USD -> JPY, Issuer USD -> MPT, Issuer USD -> XRP, Issuer USD -> CAD
    - Adds `[USD Issuer, USD/MPT Book]` to complete paths, as MPT is the destination
    - This step will add both the books it finds, but also the issuer account for IOUs and MPTs if the issuer is not the final destination
    - Incomplete paths so far:  `[USD Issuer, USD/XRP Book]`, `[USD Issuer, USD/JPY Book, JPY Issuer]`, `[USD Issuer, USD/CAD Book, CAD Issuer]`
- `"sabf"`: Query OrderBookDB for any order book that will convert from previous incomplete path to MPT:
  - `[USD Issuer, USD/JPY book, JPY Issuer]` branch:
    - Finds JPY/CHF Book. CHF is not the destination asset
    - Rejects this path and terminates.
  - `[USD Issuer, USD/XRP Book]` branch:
    - Finds XRP/MPT Book. While we found this in `"saxfd"` already, we have not seen it in this path type
    - MPT is the destination so tries to add `[USD Issuer, USD/XRP Book, XRP/MPT Book]` to complete paths. However, since this path is already in completed paths, it is ignored
  - `[USD Issuer, USD/CAD Book, CAD Issuer]` branch:
    - Finds CAD/MPT Book. The path ends with `[... USD/CAD Book, CAD Issuer]`, so the redundant `CAD Issuer` account is replaced with `CAD/MPT Book`
    - MPT is the destination so adds `[USD Issuer, USD/CAD Book, CAD/MPT Book]` to complete paths
- `"sabfd"`: Has no incomplete paths to examine

*Complete Path 1:* `[USD Issuer, USD/MPT Book]`
*Complete Path 2:* `[USD Issuer, USD/XRP Book, XRP/MPT book]`
*Complete Path 3:* `[USD Issuer, USD/CAD Book, CAD/MPT Book]`

### 1.4.3. Rank Paths

Now that there is a set of feasible paths, the algorithm ranks them.

The algorithm begins by testing the [default path](#24-default-paths). If the default path returns some liquidity, it will be deducted from the remaining liquidity used to test discovered paths.

The algorithm then simulates each discovered path to measure its [quality](../flow/README.md#21-quality) and liquidity:
- Simulate Path 1 `[USD Issuer, USD/MPT Book]`:
  - Quality: 1.05 (costs 105 USD to get 100 MPT)
  - Liquidity: 1000 MPT capacity
- Simulate Path 2 `[USD Issuer, USD/XRP Book, XRP/MPT book]`:
  - Quality: 1.04 (costs 104 USD to get 100 MPT)
  - Liquidity: 750 MPT capacity
- Simulate Path 3 `[USD Issuer, USD/CAD Book, CAD/MPT Book]`:
  - Quality: 1.06 (costs 106 USD to get 100 MPT)
  - Liquidity: 500 MPT capacity

Unless the user is trying to convert the entire possible amount of an asset, when it checks the liquidity of each path, the algorithm checks that each path can deliver at least 1/6th of the total amount[^min-liquidity], to prevent returning paths that can return very small liquidity. The total amount is divided by 6 because maxPaths is 4[^max-paths], and two is added.

Paths are ranked by quality first (lower cost is better), then by liquidity (higher is better), then by path length (shorter is better)[^rank-sort]. In this example, quality alone determines the order: Path 2 (1.04), Path 1 (1.05), Path 3 (1.06). When the user is converting the entire possible amount of an asset, quality is ignored and paths are ranked by liquidity first.

[^min-liquidity]: Minimum liquidity calculation: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L182-L186)

[^max-paths]: Maximum paths constant: [`TransactionSign.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/TransactionSign.cpp#L317-L321)

[^rank-sort]: Path ranking comparator: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L582-L597)

## 1.5. Structure

The algorithm executes four main phases:

**1. Path Discovery ([Section 4](#4-path-discovery))**

The [`findPaths`](#41-findpaths-function) function determines the payment type (XRP->NonXRP, NonXRP->XRP, NonXRP->NonXRP, etc.) and selects appropriate [path types](#23-path-types) to explore. Each path type is a template like `"sfd"` or `"saxfd"` that describes a routing strategy. The algorithm expands these templates into concrete paths by recursively querying the ledger through [`addPathsForType`](#42-addpathsfortype), which calls [`addLinks`](#43-addlinks) and [`addLink`](#44-addlink) to build paths step by step.

As it explores, `addLink` queries [AssetCache](#46-assetcache) to find accounts connected by trust lines and MPTs, and [OrderBookDB](#45-orderbookdb) to find available order books for currency conversion (including domain-restricted order books when a domain is specified). It filters out invalid options (loops, trust lines with insufficient liquidity, `NoRipple` violations, unauthorized MPT holders) and prioritizes promising routes by ranking accounts by their "paths out" score (how many onward connections they have). Complete paths that reach the destination currency at the effective destination account are stored in `mCompletePaths`.

**2. Path Ranking ([Section 5](#5-path-ranking))**

The [`computePathRanks`](#51-computepathranks) function evaluates discovered paths by simulating their execution through the Flow engine. It first tests the default path to determine `mRemainingAmount` - the liquidity still needed beyond what the default provides. This ensures paths are evaluated for their incremental value.

The [`rankPaths`](#52-rankpaths) function then tests each path in `mCompletePaths` by calling [`getPathLiquidity`](#53-getpathliquidity), which uses the Flow engine to simulate payment execution on a sandbox ledger. Paths are scored and sorted by quality (exchange rate), liquidity (capacity), and length (hop count). In normal mode, better quality is prioritized. In convert-all mode (when discovering maximum liquidity), quality is ignored and only liquidity matters.

**3. Path Selection ([Section 6](#6-path-selection))**

The [`getBestPaths`](#6-path-selection) function selects the optimal set of paths from the ranked results. It merges rankings from discovered paths and any extra paths provided by the caller, then iteratively selects paths by comparing quality first, then liquidity. It validates issuer constraints (ensuring non-default paths route through the correct issuer for IOUs and MPTs when needed) and applies different selection rules based on remaining slots: filling slots greedily when multiple remain, requiring the last path to cover all remaining liquidity, and optionally saving a "full liquidity path" that can handle the entire payment alone.

**4. Payment Execution**

The selected paths are returned to the caller ([RPC handler](#7-rpc-requests) or [Payment transaction](../payments/README.md)) and passed to the [Flow engine](../flow/README.md) for actual payment execution. Flow performs path normalization to add source and destination accounts, then executes the payment by processing each path as a strand of steps.

```mermaid
flowchart LR
    rpc((RPC/Payment))
    pathfinder[Pathfinder]
    findPaths[1. Path Discovery<br/>findPaths]
    computePathRanks[2. Path Ranking<br/>computePathRanks]
    getBestPaths[3. Path Selection<br/>getBestPaths]
    flow[4. Payment Execution<br/>Flow]

    rpc --> pathfinder
    pathfinder --> findPaths
    findPaths --> computePathRanks
    computePathRanks --> getBestPaths
    getBestPaths --> flow
```

**Domain Parameter for Permissioned DEX:**

Path finding supports an optional `domain` parameter that enables permissioned DEX functionality. When a domain is specified, the pathfinder restricts order book queries to only include offers that belong to that domain's order book. This domain value is:
- Passed into the Pathfinder constructor and stored as `mDomain`[^pathfinder-domain-constructor]
- Forwarded to OrderBookDB queries in `addLink` when discovering available books[^pathfinder-domain-orderbook] (see [Section 4.4](#44-addlink))
- Passed to RippleCalc and Flow during path ranking and execution[^pathfinder-domain-flow] (see [Section 5.1](#51-computepathranks))

[^pathfinder-domain-constructor]: Pathfinder domain parameter storage: [`Pathfinder.cpp:220,230`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L220-L230)
[^pathfinder-domain-orderbook]: OrderBookDB domain filtering flow: `addLink` ([`Pathfinder.cpp:995`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L995)) calls `getPathsOut` ([`Pathfinder.cpp:1145`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1145)), which queries OrderBookDB with domain parameter ([`Pathfinder.cpp:786`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L786))
[^pathfinder-domain-flow]: Domain passed to Flow: [`Pathfinder.cpp:411`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L411)

# 2. Terminology and Concepts

## 2.1. Terminology

**Path Elements** are the building blocks that describe a location in a path. Each path element can contain:
- An **account ID** (for rippling through trust lines or MPT connections)
- A **currency** (for specifying what currency flows through this point)
- An **issuer ID** (for non-XRP currencies, who issues the currency)
- An **MPTID** (for MPT currencies, identifies the specific MPT)

A path element can be either:
- An **account element** - represents an account that holds balances (has account ID set)
- An **offer element** - represents an order book for currency exchange (no account ID, just currency/issuer or MPTID)

**Paths** are sequences of path elements that describe a complete route from source to destination. For example:
- `[Alice] -> [USD IssuerA] -> [Book: USD/IssuerA -> EUR/IssuerB] -> [EUR/IssuerB] -> [Bob]`

**Steps** are the executable operations created by the Flow engine when it [converts paths into strands](../flow/README.md#52-path-to-strand-conversion). Path finding creates paths and path elements, not steps or strands, but it relies on the Flow engine to rank the paths. 

The relationship is: **Node Types** (search strategy) -> **Path Elements** (route description) -> **Steps** (executable operations)

**Liquidity** refers to the capacity of a path - how much value it can move in a single payment. Paths with higher liquidity can deliver more to the destination.

**Effective Destination** is the account where path finding actually searches to, which differs from the final destination for IOU and MPT payments. For XRP payments, effective destination equals the destination account. For IOU and MPT payments, effective destination is the issuer of the destination amount, since paths only need to reach the issuer - Flow handles the final hop to the destination through path normalization.

**Complete Path** is a path that successfully reaches the effective destination with the correct destination currency. Incomplete paths are discarded during path discovery.

**Convert-all Mode** is activated when the destination amount equals the maximum possible value for that currency. In this mode, path finding discovers maximum available liquidity rather than targeting a specific amount, and prioritizes liquidity over quality when ranking paths.

## 2.2. Payment Types

The pathfinder categorizes each payment request into one of five types:

| PaymentType | Description                  | Example                                |
|-------------|------------------------------|----------------------------------------|
| `PaymentType.XrpToXrp` | XRP to XRP payment           | Alice sends XRP to Bob                 |
| `PaymentType.XrpToNonXrp` | XRP to IOU or MPT payment    | Alice sends XRP, Bob receives MPT      |
| `PaymentType.NonXrpToXrp` | IOU or MPT to XRP payment    | Alice sends USD or MPT, Bob receives XRP |
| `PaymentType.NonXrpToSame` | Same IOU or MPT payment      | Alice sends USD, Bob receives USD (same asset) |
| `PaymentType.NonXrpToNonXrp` | Different IOU or MPT payment | Alice sends EUR, Bob receives USD      |

While `PaymentType.XrpToXrp` is defined as a payment type and is initialized with an empty path type list, XRP->XRP payments **never actually invoke the path finding or flow system**. The Payment transactor detects XRP->XRP payments and processes them as direct balance transfers, bypassing both path finding and the Flow engine entirely.

## 2.3. Path Types

Each payment type has a predefined table of path types at different search levels (costs). Higher search levels explore more complex paths.

Example types for `PaymentType.XrpToNonXrp`:

| Cost | Type | Path Structure                                               |
|------|----------|--------------------------------------------------------------|
| 1 | `sfd` | Source -> Destination Book -> Destination                                |
| 3 | `sfad` | Source -> Destination Book -> Account -> Destination                     |
| 5 | `sfaad` | Source -> Destination Book -> Account -> Account -> Destination          |
| 6 | `sbfd` | Source -> Book -> Destination Book -> Destination            |
| 8 | `sbafd` | Source -> Book -> Account -> Destination Book -> Destination |

The path finding algorithm searches through these types based on the requested search depth. A search level of 0 finds no paths, while higher levels (4-7 typical, 10 maximum) explore increasingly complex routing options.

For the complete list of path types for each payment type, see `Pathfinder::initPathTable()`[^init-path-table].

[^init-path-table]: Path table initialization: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1365-L1430)

**Configuration**

The search depth can be configured in `xrpld.cfg`:

| Parameter | Description | Default | Recommended |
|-----------|-------------|---------|-------------|
| `path_search` | Default search aggressiveness | 2 | 7 (for advanced path finding) |
| `path_search_fast` | Minimum search aggressiveness | 2 | 2 (for advanced path finding) |
| `path_search_max` | Maximum search aggressiveness | 3 | 10 (for advanced path finding) |
| `path_search_old` | Search level for legacy path finding interfaces | 2 | 7 (for advanced path finding) |

Higher values can exponentially increase resource usage. Setting `path_search_max` to 0 disables path finding entirely. On a server configured as a validator (one with `[validation_seed]` or `[validator_token]`), `path_search_max` defaults to 0 (path finding disabled) unless explicitly set.

### 2.3.1. Node Types

Path types are constructed from a sequence of node types. Each node type tells the path finding algorithm what kind of connection to explore at that step in the path:

**`NodeType.Source` (code: `s`)** - The source account

This represents the starting point of the payment. The source is always the first node in any path type. When path finding expands this node, it creates a single empty path representing the starting position at the source account.

**`NodeType.Accounts` (code: `a`)** - Accounts connected via trust lines or MPTs

When path finding encounters an `a` node, it expands to neighboring accounts connected to the current position via [trust lines](../trust_lines/README.md) or [MPTs](../mpts/README.md). The actual account selection involves filtering by NoRipple flags, liquidity, and authorization, then ranking candidates by their number of viable outgoing paths. See [Section 4.4](#44-addlink) for details.

**`NodeType.Books` (code: `b`)** - Order books for currency conversion

When path finding encounters a `b` node, it queries [OrderBookDB](#45-orderbookdb) for all order books that accept the current currency as input, allowing the path to exchange into a different currency. See [Section 4.4](#44-addlink) for details on book expansion, including how output issuers are handled.

**`NodeType.XrpBook` (code: `x`)** - Order book to XRP

A specialized version of `NodeType.Books` that only considers order books that convert the current currency to XRP. XRP often serves as a bridge currency between other currencies, and limiting to XRP books reduces the search space. [OrderBookDB](#45-orderbookdb) maintains a separate `xrpBooks` cache (and `xrpDomainBooks` for permissioned DEX) for faster lookups.

**`NodeType.DestBook` (code: `f`)** - Order book to destination currency

This is another specialized version of `NodeType.Books` that only considers order books that output the destination currency. The `f` stands for "final" book. When path finding encounters an `f` node, it only looks for order books that convert the current currency into whatever currency the destination wants to receive. This ensures the path ends with the correct currency.

For example, if the destination wants EUR, an `f` node will only consider order books like USD/EUR, XRP/EUR, GBP/EUR, etc.

**`NodeType.Destination` (code: `d`)** - The destination account

The destination is always the last node in any path type. When path finding encounters a `d` node, it searches for account connections to the effective destination. For IOUs and MPTs, the effective destination is the issuer, not the final recipient. If a preceding `f` step already ended at the issuer, the path is already complete and `d` has nothing to add. When it does fire, `d` adds the issuer as a trust line or MPT hop, completing the path via rippling.


## 2.4. Default Paths

A **default path** is the direct route between source and destination that does not need to be explicitly specified. The default path is:

- **For same-currency IOU or MPT payments**: Direct transfer between source and destination through the issuer
- **For cross-currency payments**: Uses the order book between the source currency and the destination currency

Unless the Payment transaction contains `tfNoRippleDirect`, the Flow engine always attempts the default path, even when explicit paths are provided. The default path can fail (e.g., no trust line exists, no order book available), in which case the Flow engine continues with any explicit paths.

Default path is an empty path that is passed to [Path Normalization](../flow/README.md). 

# 3. Pathfinder

The Pathfinder class is responsible for orchestrating finding of paths. It creates a Path set - a vector of Paths. Paths consist of Path Elements. 
The Pathfinder uses path types to systematically explore the ledger, building STPath objects from STPathElement components and collecting them into an STPathSet.

```mermaid
classDiagram
    class Pathfinder {
        +STPathSet mCompletePaths
        +AssetCache cache
        +findPaths()
        +computePathRanks()
        +getBestPaths()
    }

    class STPathSet {
        +vector~STPath~ paths
    }

    class STPath {
        +vector~STPathElement~ mPath
        +hasSeen()
        +push_back()
    }

    class STPathElement {
        +unsigned int mType
        +AccountID mAccountID
        +PathAsset mAssetID
        +AccountID mIssuerID
    }

    Pathfinder "1" --> "1" STPathSet : produces
    STPathSet "1" *-- "0..*" STPath : contains
    STPath "1" *-- "1..*" STPathElement : contains
```
*Figure: Key components of Pathfinder*


## 3.1. Pathfinder Class

The `Pathfinder` class is the entry point for path finding. It is constructed with:

> [!IMPORTANT]
> Parameter names and definitions are simplified to provide an overview. They do not map 1:1 to the C++ implementation, but are intended to make the pseudocode in later sections easier to follow.

| Parameter | Description                                                        | Required |
|-----------|--------------------------------------------------------------------|----------|
| `cache` | AssetCache containing ledger state, trust line, and MPT information | ✅ |
| `srcAccount` | Source account ID - the account sending funds                      | ✅ |
| `dstAccount` | Destination account ID - the account receiving funds               | ✅ |
| `srcPathAsset` | Asset the source wants to spend                 | ✅ |
| `srcIssuer` | Issuer for the source currency                                     | ❌ |
| `dstAmount` | The amount to be delivered to the destination                      | ✅ |
| `srcAmount` | Maximum amount the source is willing to spend                      | ❌ |
| `domain` | Domain identifier for permissioned DEX                             | ❌ |
| `app` | Application reference for accessing ledger state                   | ✅ |

The pathfinder maintains these key variables:
- `app` - Application reference which can be used to fetch `OrderBookDB`
- `convert_all_` - Boolean flag indicating "convert all" mode (find maximum liquidity instead of exact amount). Set to true when destination amount equals the maximum possible value for that currency
- `mSrcAccount` - Source account
- `mSrcPathAsset` - Asset the source wants to spend, derived from `srcPathAsset` constructor parameter
- `mDstAccount` - Destination account (the account that will ultimately receive the payment)
- `mEffectiveDst` - The account where paths must end. For XRP destinations, this is `mDstAccount`. For IOU and MPT destinations, this is the issuer of the destination amount. Paths discovered by path finding end at this account, not at `mDstAccount`. The Flow engine later handles the final hop from `mEffectiveDst` to `mDstAccount` through path normalization.
- `mCompletePaths` - Collection of all complete paths found
- `mPathRanks` - Rankings of paths based on quality and liquidity
- `mPaths` - Cache of paths organized by PathType
- `mPathsOutCountMap` - Cache of "paths out" counts for each Issue
- `mPathTable` - Static (file-level) table of [path types](#23-path-types), shared across all Pathfinder instances
- `mSource` - STPathElement representing the starting point for path discovery. Computed in `findPaths`: if the source asset has a non-XRP issuer (`mSrcIssuer`), the element's account and issuer are set to that issuer; otherwise, they are set to `mSrcAccount`. For XRP, the issuer is the zero account. Used by `addLink` as the implicit first element when the current path is empty.
- `mRemainingAmount` - Amount remaining to deliver after accounting for default path contribution
- `mAssetCache` - [AssetCache](#46-assetcache) for querying trust line and MPT information

## 3.2. Path Elements

An `STPathElement`[^stpathelement] is the data structure representing a single step in a payment path. It is a class containing:

- `mType`[^mtype] (unsigned int) - Bitmask indicating which fields are present:
    - `0x01` (typeAccount)[^typeaccount] - Has account field
    - `0x10` (typeCurrency)[^typecurrency] - Has currency field
    - `0x20` (typeIssuer)[^typeissuer] - Has issuer field
    - `0x40` (typeMPT)[^typempt] - Has MPT field
- `mAccountID` (AccountID) - The account (if typeAccount bit is set)
- `mAssetID` (PathAsset) - Holds either a Currency or an MPTID
- `mIssuerID` (AccountID) - The issuer (if typeIssuer bit is set)

[^stpathelement]: STPathElement class definition: [`STPathSet.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STPathSet.h#L17)
[^mtype]: mType field: [`STPathSet.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STPathSet.h#L19)
[^typeaccount]: typeAccount constant: [`STPathSet.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STPathSet.h#L32)
[^typecurrency]: typeCurrency constant: [`STPathSet.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STPathSet.h#L33)
[^typeissuer]: typeIssuer constant: [`STPathSet.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STPathSet.h#L34)
[^typempt]: typeMPT constant: [`STPathSet.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/STPathSet.h#L35)

**Types of path elements:**

**Account element** (`type = 0x01`):
```
{type: 0x01, account: aliceAccountID}
```
Represents an account in a path - either an intermediate account for rippling or the destination account. Created during pathfinding[^account-element-creation] when adding accounts to incomplete paths.

**Offer/Book element for XRP** (`type = 0x10`):
```
{type: 0x10, currency: "XRP"}
```
Represents a conversion to XRP via order book. Has currency only[^xrp-book-element-creation] (0x10).

**Offer/Book element for IOUs** (`type = 0x30`):
```
{type: 0x30, currency: "USD", issuer: issuerAccountID}
```
Represents a currency conversion via order book. Has both currency and issuer[^iou-book-element-creation] (0x10 | 0x20 = 0x30).

**Offer/Book element for MPTs** (`type = 0x60`):
```
{type: 0x60, mptid: mptID, issuer: issuerAccountID}
```
Represents an MPT order book. Has both MPT identifier and issuer (0x40 | 0x20 = 0x60).

**Account+Asset element** (`type = 0x31` for IOUs, `0x61` for MPTs):
```
{type: 0x31, account: sourceAccountID, currency: "USD", issuer: issuerAccountID}
```
Represents the source account holding a specific asset. **Always added as the first element**[^source-element-normalization] during path normalization (in the Flow engine's `toStrand` function). Has account, currency, and issuer (0x01 | 0x10 | 0x20 = 0x31) or account, MPT, and issuer (0x01 | 0x40 | 0x20 = 0x61).
The issuer in the first element is set to source, and the next element connects it to a particular issuer.

[^account-element-creation]: Account element creation during pathfinding: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1202-L1203)
[^xrp-book-element-creation]: XRP book element creation during pathfinding: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1225-L1226)
[^iou-book-element-creation]: IOU book element creation during pathfinding: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1287-L1291)
[^source-element-normalization]: Source element added during path normalization: [`PaySteps.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/PaySteps.cpp#L262-L273)


## 3.3. Path

An `STPath` represents a single payment path. It is a vector of [STPathElements](#32-path-elements).

**Key methods:**

- `size()` - Returns the number of path elements in the path
- `empty()` - Returns true if path has no elements
- `push_back(element)` - Adds an element to the end of the path
- `back()` - Returns the last element in the path
- `hasSeen(account, asset, issuer)` - Checks if this combination already appears in the path (loop detection). The asset parameter is a PathAsset which can hold either Currency (for IOUs) or MPTID (for MPTs).

For example, a normalized XRP -> USD path would look like: 

```
[0] Type=0x31 (Account Currency Issuer ) Account: sourceAccountID Currency: XRP Issuer: rrrrrrrrrrrr
[1] Type=0x1 (Account ) Account: rrrrrrrrrrrr
[2] Type=0x30 (Currency Issuer ) Currency: USD Issuer: issuerAccountID
[3] Type=0x1 (Account ) Account: issuerAccountID
[4] Type=0x1 (Account ) Account: destinationAccountId
```

An `STPathSet` is a collection of multiple `STPath` objects, representing alternative paths that can be used simultaneously for a payment.


# 4. Path Discovery

The path discovery process works by expanding **path types** into a **tree of concrete paths**.
The core algorithm takes a type like `"sfad"` and expands it step-by-step: first it builds all paths matching `"s"`, then extends those to match `"sf"`, then `"sfa"`, and finally `"sfad"`. At each step, it queries the ledger (AssetCache for trust lines and MPTs, OrderBookDB for order books) to find what's actually available, creating branches for each viable option while filtering out loops and insufficient liquidity.

Consider a payment where the source holds USD and wants to deliver EUR.Issuer to the destination. The pathfinder uses type `"sfad"` (source -> destination book -> account -> destination):

The type `"sfad"` expands as follows:
1. **s** (source) - Start with source account
2. **f** (destination book) - Query OrderBookDB for order books from USD that output the destination currency (EUR). Finds USD->EUR.Issuer book
3. **a** (accounts) - Query AssetCache for accounts holding EUR.Issuer. Finds Alice (has liquidity), Bob (zero balance, filtered out), and Carol (has liquidity)
4. **d** (destination) - Try to complete each path by reaching the effective destination (Issuer):
   - Alice has a trust line to Issuer -> path complete (blue)
   - Carol has no trust line to Issuer -> filtered out (red)

```mermaid
graph TD
    Start["source"] --> Book1["source<br/>USD->EUR.Issuer book"]

    Book1 --> Acct1["source<br/>USD->EUR.Issuer book<br/>Alice"]
    Book1 --> Filtered1["source<br/>USD->EUR.Issuer book<br/>Bob (no liquidity)"]
    Book1 --> Acct3["source<br/>USD->EUR.Issuer book<br/>Carol"]

    Acct1 --> Complete1["source<br/>USD->EUR.Issuer book<br/>Alice<br/>Issuer"]

    Acct3 --> Filtered2["source<br/>USD->EUR.Issuer book<br/>Carol (no path to Issuer)"]

    style Complete1 fill:blue
    style Filtered1 fill:red
    style Filtered2 fill:red
```

The algorithm uses:
- **Filtering** - Rejects loops, insufficient liquidity, NoRipple violations before creating branches
- **Ranking** - Prioritizes destination connections and high "paths out" scores
- **Limits** - Maximum 1000 complete paths; when expanding accounts, up to 50 candidates from the source account, up to 10 candidates from any other account


**Implementation:**

1. **`findPaths`** - Entry point that:
   - Determines payment type (XRP->NonXRP, NonXRP->XRP, NonXRP->NonXRP, etc.) and iterates through all path types for that payment type
   - For each type, calls `addPathsForType` to expand it

2. **`addPathsForType`** - Recursive function that:
   - Takes a type like `"sfad"` (source -> book -> account -> destination) and builds it incrementally by first building `"s"`, then `"sf"`, then `"sfa"`, then `"sfad"`
   - For each step, calls `addLinks` to expand the last node type
   - Returns a list of paths (complete and incomplete) for that type, which is cached in `mPaths` for reuse

3. **`addLinks`** - Simple wrapper that:
   - Takes a set of incomplete paths and calls `addLink` once for each path

4. **`addLink`** - Core expansion logic that:
   - Takes one incomplete path and flags indicating what type of expansion to perform (add accounts via `a` node type, or add books via `b`/`f` node types)
   - Based on the flags:
     - If expanding accounts (`a`): queries **`AssetCache`** to find connected accounts via trust lines or MPTs
     - If expanding books (`b`/`f`): queries **`OrderBookDB`** to find available order books (from offers and AMMs)
   - For each viable option found, creates a new branch by appending path elements to `incompletePaths`
   - **When a path reaches the destination with the correct currency, adds it directly to `mCompletePaths`**
   - Filters out loops, NoRipple violations, and insufficient liquidity

The entire flow can be interrupted by using `continueCallback`. This callback allows the caller to interrupt path finding by returning `false`. It is checked at multiple points during path discovery.

- **[`path_find` subscriptions](#72-path_find-rpc)** provides a callback that checks if the WebSocket client is still connected
- **[`ripple_path_find`](#71-ripple_path_find-rpc-legacy)** and **[transaction signing with `build_path`](../payments/README.md#421-path-finding)** do NOT provide a callback


## 4.1. findPaths Function

The `findPaths` function[^find-paths] is the main path discovery engine. It searches for paths from `mSrcAccount` to `mEffectiveDst`.

[^find-paths]: Main path discovery function: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L241-L380)

- **For XRP payments**: `mEffectiveDst` equals `mDstAccount`, so paths go directly to the destination
- **For IOU and MPT payments**: `mEffectiveDst` is the issuer of the destination amount. Path finding only needs to reach the issuer - the Flow engine will handle the final hop from issuer to destination through path normalization.

**Parameters:**

| Parameter | Description | Required |
|-----------|-------------|----------|
| `searchLevel` | Maximum cost/depth to search | ✅ |
| `continueCallback` | Optional callback to check if search should continue | ❌ |

The function validates the payment request, determines [payment type](#22-payment-types) from source and destination currencies and iterates through path types for that payment type and calls addPathsForType, making sure not to exceed the `searchLevel`.

All found paths are stored in PathFinder object state, so this function only loops over types without much responsibility itself. 

### 4.1.1. findPaths Pseudo-Code

```python
def findPaths(searchLevel, continueCallback) -> bool:
    # Validate payment request
    if mDstAmount == 0:
        # Destination amount is 0
        return False
    
    if mSrcAccount == mDstAccount and mDstAccount == mEffectiveDst and mSrcPathAsset == mDstAmount.asset():
        # No need to send to same account with same currency
        return False

    if mSrcAccount == mEffectiveDst and mSrcPathAsset == mDstAmount.asset():
        # Default path might work, but any additional path would loop back to source
        # (since paths must end at mEffectiveDst which is the source)
        return True

    if not mLedger:
        # No ledger to search
        return False

    if not accountExists(mSrcAccount):
        # Source account has to exist. Destination does not if we are sending XRP to it.
        return False
    
    if mEffectiveDst != mDstAccount and not accountExists(mEffectiveDst):
        # Issuer account has to exist
        return False

    if not accountExists(mDstAccount) and (not isXRP(mDstAmount) or mDstAmount < getAccountReserve()):
        # New account must be funded with XRP meeting minimum reserve
        return False

    # Build the source element used by addLink when the path is empty.
    # If the source asset has a non-XRP issuer, start from the issuer's account;
    # otherwise start from the source account.
    if mSrcIssuer and not isXRP(mSrcPathAsset) and not isXRP(mSrcIssuer):
        account = mSrcIssuer
    else:
        account = mSrcAccount
    issuer = xrpAccount() if isXRP(mSrcPathAsset) else account
    mSource = STPathElement(account, mSrcPathAsset, issuer)

    # Determine payment type (one of: PaymentType.XrpToXrp, PaymentType.XrpToNonXrp, etc.)
    paymentType = determinePaymentType(mSrcPathAsset, mDstAmount.asset())

    # Search for paths using types for this payment type
    for costedPath in mPathTable[paymentType]:
        if continueCallback.shouldBreak():
            return False
        if costedPath.searchLevel <= searchLevel:
            # costedPath.type is a PathType (sequence of node types like "sfd", "sfad", etc.)
            addPathsForType(costedPath.type, continueCallback)
            if len(mCompletePaths) > PATHFINDER_MAX_COMPLETE_PATHS: # 1000
                break

    return True
```

## 4.2. addPathsForType

`addPathsForType`[^add-paths-for-type] takes a `PathType` like `"sfad"` and converts it into concrete paths by building incrementally, one node type at a time.

[^add-paths-for-type]: Incremental path building function: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L872-L943)

**Parameters:**

| Parameter | Description | Required |
|-----------|-------------|----------|
| `pathType` | Sequence of node types (e.g., `"sfd"`, `"saxfd"`) | ✅ |
| `continueCallback` | Optional callback to check if search should continue | ❌ |

The function works recursively. When asked to build `"sfad"`, it first checks if `"sfad"` was already built and cached in `mPaths` - if so, it returns immediately. Otherwise, it recursively strips off the last character until it hits the empty string:

- `"sfad"` -> `"sfa"` -> `"sf"` -> `"s"` -> `""` (base case returns empty list)

Now the recursion unwinds and each level builds paths by extending what its parent returned. Each node type maps to a specific expansion strategy:

- **`"s"`**: Returns a single empty path (the starting point at source)
- **`"sf"`**: Takes that empty path and calls `addLinks` with flags to query OrderBookDB for order books from the source currency that output the destination currency. If 5 such books exist, returns 5 paths.
- **`"sfa"`**: Takes those 5 paths and calls `addLinks` with flags to query AssetCache for accounts holding the book's output currency. If each book leads to 3 accounts, returns 15 paths (5 * 3).
- **`"sfad"`**: Takes those 15 paths and calls `addLinks` with flags to find the destination account. Only paths reaching the destination with the correct currency are not discarded.

The flags passed to `addLinks` tell `addLink` what to query (accounts via AssetCache or books via OrderBookDB) and what filters to apply (only XRP books, only destination currency, only destination account, etc.). Each node type (`s`, `a`, `b`, `x`, `f`, `d`) uses different flags to control this behavior.

Finally, the result is stored in `mPaths[pathType]` and returned.

### 4.2.1. addPathsForType Pseudo-Code

```python
def addPathsForType(pathType, continueCallback) -> list[STPath]:
    # pathType is a sequence of node types like "sfd" or "saxfd"

    # Check cache
    if pathType in mPaths:
        return mPaths[pathType]

    # Base case - empty path type
    if len(pathType) == 0:
        mPaths[pathType] = []
        return mPaths[pathType]

    if continueCallback.shouldBreak():
        return []

    # Recursive case - build parent paths first
    parentPathType = pathType[:-1]  # Remove last node type (e.g., "sfd" -> "sf")
    parentPaths = addPathsForType(parentPathType, continueCallback)

    # Add final node type to parent paths
    nodeType = pathType[-1]  # Get last node type (e.g., 'd' from "sfd")
    pathsOut = []

    if nodeType == NodeType.Source:
        pathsOut = [empty_path]
    elif nodeType == NodeType.Accounts:
        addLinks(currentPaths=parentPaths, incompletePaths=pathsOut, addFlags=afADD_ACCOUNTS, continueCallback=continueCallback)
    elif nodeType == NodeType.Books:
        addLinks(currentPaths=parentPaths, incompletePaths=pathsOut, addFlags=afADD_BOOKS, continueCallback=continueCallback)
    elif nodeType == NodeType.XrpBook:
        addLinks(currentPaths=parentPaths, incompletePaths=pathsOut, addFlags=afADD_BOOKS | afOB_XRP, continueCallback=continueCallback)
    elif nodeType == NodeType.DestBook:
        addLinks(currentPaths=parentPaths, incompletePaths=pathsOut, addFlags=afADD_BOOKS | afOB_LAST, continueCallback=continueCallback)
    elif nodeType == NodeType.Destination:
        addLinks(currentPaths=parentPaths, incompletePaths=pathsOut, addFlags=afADD_ACCOUNTS | afAC_LAST, continueCallback=continueCallback)

    mPaths[pathType] = pathsOut
    return pathsOut
```

## 4.3. addLinks

`addLinks`[^add-links] is a simple wrapper that calls `addLink` for each path in a set.

[^add-links]: Wrapper function for batch path extension: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L857-L870)

**Parameters:**

| Parameter | Description | Required |
|-----------|-------------|----------|
| `currentPaths` | Set of incomplete paths to extend | ✅ |
| `incompletePaths` | Output list where new paths are added | ✅ |
| `addFlags` | Flags controlling expansion behavior (e.g., `afADD_ACCOUNTS`, `afADD_BOOKS`) | ✅ |
| `continueCallback` | Optional callback to check if search should continue | ❌ |

### 4.3.1. addLinks Pseudo-Code

```python
def addLinks(currentPaths, incompletePaths, addFlags, continueCallback):
    for path in currentPaths:
        if continueCallback.shouldBreak():
            return
        addLink(currentPath=path, incompletePaths=incompletePaths, addFlags=addFlags, continueCallback=continueCallback)
```

## 4.4. addLink

`addLink`[^add-link] is where the actual path expansion happens - it's the function that queries the ledger and creates new path branches.

[^add-link]: Core path expansion function: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L972-L1301)

**Parameters:**

| Parameter | Description | Required |
|-----------|-------------|----------|
| `currentPath` | Single incomplete path to extend | ✅ |
| `incompletePaths` | Output list where new paths are added | ✅ |
| `addFlags` | Flags controlling expansion behavior (e.g., `afADD_ACCOUNTS`, `afADD_BOOKS`, `afOB_XRP`, `afOB_LAST`, `afAC_LAST`) | ✅ |
| `continueCallback` | Optional callback to check if search should continue | ❌ |

Every partial path has an **endpoint**, derived from its last [path element](#32-path-elements) (or from the source if the path is empty)[^addlink-endpoint]. The endpoint provides an account, an asset (currency or MPTID), and an issuer, which `addLink` uses to determine where to search for the next hop.

[^addlink-endpoint]: Endpoint extraction from partial path: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1001-L1005)

The function examines the path's current endpoint (which account and currency/asset) and uses `addFlags` to determine which ledger data source to query and what filters to apply. It produces two kinds of output: paths that reach the effective destination with the correct asset are added to `mCompletePaths`, while paths that still need further extension are added to `incompletePaths`. `addPathsForType` feeds incomplete paths back into `addLink` for the next expansion round, and complete paths proceed to [path ranking](#5-path-ranking), where they are simulated through the Flow engine to measure quality and liquidity.

**Flag meanings:**

- **`afADD_ACCOUNTS`** - Queries AssetCache for trust lines and MPT holdings to find connected accounts
- **`afADD_BOOKS`** - Queries OrderBookDB to find order books where the current asset is the input (TakerPays)
- **`afOB_XRP`** (modifier for `afADD_BOOKS`) - Restricts books to only those outputting XRP
- **`afOB_LAST`** (modifier for `afADD_BOOKS`) - Restricts books to only those outputting the destination asset
- **`afAC_LAST`** (modifier for `afADD_ACCOUNTS`) - Restricts accounts to only the effective destination

The function's behavior varies significantly between account expansion and book expansion:

**Account expansion** (`afADD_ACCOUNTS`):

When the path's current endpoint is on XRP and the destination amount is XRP and the current path is non-empty, it adds the path to complete paths.[^xrp-endpoint-check] Empty paths are not added because they would represent XRP->XRP payments and those do not require pathfinding.

[^xrp-endpoint-check]: XRP endpoint completion check: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1019-L1027)

For non-XRP endpoints, the function queries for trust lines or MPTs connected to the account at the path's current endpoint.

For IOU currencies, `addLink` calls `AssetCache::getRippleLines`[^get-ripple-lines-call] to fetch trust lines for the endpoint account. The returned trust lines cover all currencies; `addLink` later filters them to match the current currency via `correctAsset`[^currency-match]. These trust lines are candidates for the next hop in the payment path, subject to the checks described below.

The query is pre-filtered using `LineDirection`, which hints to `getRippleLines` which trust lines are needed. Rippling through an account is blocked when that account has NoRipple set on **both** its incoming and outgoing trust lines. `addLink` uses this rule at two levels:

- `addLink` calls `isNoRippleOut(currentPath)`[^is-no-ripple-out] to check whether the trust line between the previous account in the path and the endpoint account has NoRipple set on the endpoint account's side.
- If `isNoRippleOut` returns true, `getRippleLines` is called with `LineDirection::incoming`[^noripple-direction], which requests only trust lines where the endpoint account does **not** have NoRipple set[^get-ripple-lines-direction]. However, this is a best effort optimization. The same account may be reached via different partial paths during pathfinding, and if its full set of trust lines was already cached from an earlier call, `AssetCache` returns those instead of fetching the filtered subset, to avoid duplicate storage[^asset-cache-superset].
- If `isNoRippleOut` returns false, `getRippleLines` is called with `LineDirection::outgoing`, which returns all trust lines.

Regardless of which set `getRippleLines` returned, `addLink` performs a per-candidate check that provides the actual gating: if `isNoRippleOut` was true and the candidate trust line also has NoRipple set on the endpoint account's side (`asset.getNoRipple()`), the candidate is skipped[^noripple-candidate-check].

For MPTs, the function queries for MPTs associated with the current account. The peer account is always the issuer, extracted from the MPTID[^mpt-peer-issuer], so MPT account expansion only navigates from holder to issuer. The reverse direction (issuer to holder) is never needed because pathfinding only needs to reach `mEffectiveDst` (the issuer for non-XRP destinations); the flow engine handles the final hop from issuer to destination holder through path normalization[^mpt-no-reverse].

Each asset connection undergoes these checks in order:

1. **Currency/asset match**[^currency-match] - The asset must match the path's current currency code or MPTID
2. **Destination account bypass**[^dest-bypass] - When the destination issuer differs from the destination account, skips the destination account (pathfinding only needs to reach the issuer, not the final recipient)
3. **Destination-only filtering**[^dest-only] - When `afAC_LAST` is set, rejects all accounts except the effective destination (the issuer)
4. **Loop detection**[^loop-detection] - Rejects accounts already visited using `hasSeen(account, asset, issuer)`
5. **Liquidity and NoRipple check**[^liquidity-check]:
   - For trust lines: Rejects if (balance <= 0 AND (no peer limit OR peer limit exhausted OR unauthorized when `lsfRequireAuth` is set)) OR (both the previous link and current link have NoRipple set)
   - For MPTs: Rejects if zero balance OR maxed out OR not authorized
6. **Source loop prevention**[^source-loop] - Rejects accounts that would loop back to the source

[^currency-match]: Currency/asset match check: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1082-L1092)
[^dest-bypass]: Destination account bypass check: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1069-L1073)
[^dest-only]: Destination-only filtering check: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1077-L1080)
[^loop-detection]: Loop detection using hasSeen: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1110)
[^liquidity-check]: Liquidity and NoRipple check: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1093-L1108)
[^source-loop]: Source loop prevention check: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1138-L1141)
[^get-ripple-lines-call]: getRippleLines call in addLink: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1161-L1163)
[^is-no-ripple-out]: isNoRippleOut checks whether the last account in the path has NoRipple set on its outgoing link: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L958-L979)
[^noripple-direction]: Trust line fetch direction based on NoRipple: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1159-L1166)
[^get-ripple-lines-direction]: LineDirection::incoming excludes trust lines where the account has NoRipple set: [`TrustLine.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/TrustLine.cpp#L61)
[^noripple-candidate-check]: Per-candidate NoRipple check in addLink: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1101)
[^asset-cache-superset]: AssetCache returns the outgoing superset when incoming is requested but outgoing is already cached: [`AssetCache.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/AssetCache.cpp#L78-L87)
[^getpathsout]: getPathsOut computes the paths out score for an account: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L748-L835)
[^getpathsout-auth]: getPathsOut checks lsfRequireAuth on the candidate account: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L771-L775)
[^getpathsout-booksize]: Score starts with order book size: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L786)
[^getpathsout-destination-bonus]: Destination bonus of +10000: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L800-L804)
[^getpathsout-frozen]: Global freeze check in getPathsOut: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L776-L784)
[^getpathsout-iou-loop]: IOU trust line scoring loop: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L788-L830)
[^getpathsout-noripple]: getPathsOut skips trust lines where the peer has NoRipple set: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L805-L806)
[^getpathsout-freeze]: getPathsOut skips trust lines where the peer has frozen the line: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L807-L808)
[^getpathsout-mpt-loop]: MPT scoring loop: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L813-L831)
[^getpathsout-mpt-match]: MPT ID match check: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L818-L820)
[^getpathsout-mpt-balance]: MPT zero balance or maxed out check: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L818-L820)
[^getpathsout-mpt-auth]: MPT authorization check: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L818-L820)
[^getpathsout-mpt-destination]: MPT destination bonus of +10000: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L821-L825)
[^getpathsout-mpt-frozen]: MPT frozen check (redundant with outer freeze check): [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L826-L827)
[^getpathsout-mpt-count]: MPT count increment: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.3.0/src/xrpld/rpc/detail/Pathfinder.cpp#L827-L828)
[^compare-account-candidate]: compareAccountCandidate sorts by priority descending, then account ID descending: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L108-L124)
[^dest-complete-path]: Destination account with matching asset completes the path: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1121-L1130)
[^dest-high-priority]: Destination account with non-matching asset receives high priority directly: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1132-L1136)
[^getpathsout-zero-filter]: Candidates with score 0 are not added: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1152)
[^candidate-extend]: Selected candidates are extended into incomplete paths: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1196-L1206)
[^mpt-peer-issuer]: MPT peer is always the issuer, extracted from the MPTID: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L1057-L1059), [`MPTIssue.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/MPTIssue.h#L84-L93)
[^mpt-no-reverse]: Pathfinding targets `mEffectiveDst` (the issuer for non-XRP destinations), so it never needs to navigate from issuer to holder. The flow engine handles the final hop to the destination holder via path normalization: [`PaySteps.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/PaySteps.cpp#L261-L320)

Accounts that pass all filters become candidates. When the candidate is the destination account and the current asset matches the destination asset, the path is complete[^dest-complete-path]. When the candidate is the destination account but the asset does not match, it receives a score of 10000 directly, bypassing `getPathsOut()`[^dest-high-priority]. All other candidates are scored by `getPathsOut()`[^getpathsout], which counts the number of viable onward connections from that account in the current asset.

Candidates that score 0 are excluded entirely[^getpathsout-zero-filter], because an account with no viable onward connections (e.g., globally frozen, or all trust lines are frozen, unauthorized, or have NoRipple set) would be a dead end in the path. Non-zero scores determine priority during sorting. In the rules below, **Skipped** means the connection does not count toward the score (+0):

1. If the account is globally frozen for the asset, no scoring happens and the count stays 0[^getpathsout-frozen].
2. Otherwise, the function checks whether the account has `lsfRequireAuth` set[^getpathsout-auth].
3. The score starts with the number of order books available for the asset[^getpathsout-booksize].
4. For IOU currencies, the function iterates over trust lines from `getRippleLines(account, direction)` and for each trust line[^getpathsout-iou-loop]:
   - **Skipped** if the trust line currency does not match the current asset
   - **Skipped** if the balance is zero or negative and the peer has no available credit, or if `lsfRequireAuth` is set (step 2) and the trust line is not authorized
   - **+10000** if the current asset matches the destination asset and the peer is the destination account[^getpathsout-destination-bonus]
   - **Skipped** if the peer has NoRipple set on its side of the trust line[^getpathsout-noripple]
   - **Skipped** if the peer has frozen the trust line[^getpathsout-freeze]
   - **+1** otherwise
5. For MPTs, the function iterates over MPTs from `getMPTs(account)` and for each MPT[^getpathsout-mpt-loop]:
   - **Skipped** if the MPT ID does not match the current asset[^getpathsout-mpt-match]
   - **Skipped** if zero balance or maxed out[^getpathsout-mpt-balance]
   - **Skipped** if authorization is required (step 2)[^getpathsout-mpt-auth]
   - **+10000** if the current asset matches the destination asset and the peer is the destination account[^getpathsout-mpt-destination]
   - **Skipped** if frozen (redundant with the outer freeze check in step 1, but present in the code)[^getpathsout-mpt-frozen]
   - **+1** otherwise[^getpathsout-mpt-count]

`addLink` then sorts all candidates by score descending, with account ID as a tiebreaker[^compare-account-candidate], and selects the top 50 if expanding from the source account, or top 10 otherwise. For each selected candidate, `addLink` extends the current path by appending the candidate as an account element and adds the extended path to `incompletePaths`[^candidate-extend]. These incomplete paths are fed back into `addLink` by `addPathsForType` for further expansion in subsequent rounds.

**Book expansion** (`afADD_BOOKS`):

When `afOB_XRP` is set, the function checks whether an order book exists from the current asset to XRP (using domain filtering if configured). If found, it adds an XRP book element to the path.

Without `afOB_XRP`, the function queries all order books where the current asset is the input currency (TakerPays). Each book undergoes these filters:

1. **Output loop detection** - Rejects books whose output asset/issuer was already seen using `hasSeen(xrpAccount(), book.out, book.out.getIssuer())`
2. **Origin issuer check** - Rejects books that would create a loop back to the source issuer
3. **Destination asset filter** - When `afOB_LAST` is set, rejects books not outputting the destination asset

For books outputting XRP, the function adds an XRP book element. If the destination amount is XRP, this completes the path; otherwise the path is added to `incompletePaths` for further expansion.

For books outputting non-XRP assets, the function performs an additional check using `hasSeen(book.out.getIssuer(), book.out, book.out.getIssuer())` to prevent issuer loops. It then adds the book element, with an optimization: if the path already has a book -> account -> book pattern, it replaces the redundant intermediate account with the new book element.

After adding the book element, the function determines whether the path is complete. If the destination requires reaching a specific issuer (non-XRP destination), the function checks whether the book's output issuer matches. When the issuer matches the destination account but differs from the effective destination, the path is rejected (this indicates an issuer bypass violation). When the issuer matches the effective destination and the asset matches, the path is complete. Otherwise, the function appends the issuer's account element.

### 4.4.1. addLink pseudoCode

```python
def addLink(currentPath, incompletePaths, addFlags, continueCallback):
    pathEnd = currentPath.back() if currentPath else mSource
    endPathAsset = pathEnd.getPathAsset()
    endAccount = pathEnd.getAccountID()
    isOnXRP = isXRP(endPathAsset)
    hasEffectiveDst = mEffectiveDst != mDstAccount
    destOnly = afAC_LAST in addFlags

    if afADD_ACCOUNTS in addFlags:
        if isOnXRP:
            if mDstAmount.isXRP() and not currentPath.empty():
                mCompletePaths.add(currentPath)  # Complete XRP->XRP path
        elif endAccount.exists():
            # Check if the trust line between the previous account and the endpoint account
            # has NoRipple set on the endpoint account's side
            noRippleOut = isNoRippleOut(currentPath)
            direction = LineDirection.incoming if noRippleOut else LineDirection.outgoing

            # Get trust lines or MPTs from current account, based on asset type.
            # For IOUs, getRippleLines returns trust lines filtered by direction:
            #   incoming = only lines where the account does not have NoRipple set (best effort, see AssetCache)
            #   outgoing = all trust lines
            # For MPTs, getMPTs returns all MPT holdings for the account.
            if isIOU(endPathAsset):
                assets = mAssetCache.getRippleLines(endAccount, direction)
            else:
                assets = mAssetCache.getMPTs(endAccount)
            candidates = []

            for asset in assets:
                if continueCallback.shouldBreak():
                    return

                # Get peer account (for trust lines) or issuer account (for MPTs)
                peerAccount = asset.getAccountIDPeer() if isTrustLine(asset) else asset.getIssuer()

                # Skip if issuer bypass
                if hasEffectiveDst and peerAccount == mDstAccount:
                    continue

                # Check if this is the destination
                isDestination = peerAccount == mEffectiveDst

                # Destination-only filter
                if destOnly and not isDestination:
                    continue

                # Skip if asset does not match the path's current currency/MPTID
                if not correctAsset(asset, endPathAsset):
                    continue

                # Skip if creates loop
                if currentPath.hasSeen(peerAccount, endPathAsset, peerAccount):
                    continue

                # Skip if insufficient liquidity or NoRipple violation
                # For trust lines: (balance > 0 OR (available credit AND authorized))
                #                   AND NOT (noRippleOut AND asset.getNoRipple())
                # For MPTs: balance > 0 AND not maxed out AND authorized
                if not hasLiquidity(asset):
                    continue

                # Handle destination account
                if isDestination:
                    if endPathAsset == mDstAmount.asset():
                        if not currentPath.empty():
                            mCompletePaths.add(currentPath)  # Complete path
                    elif not destOnly:
                        candidates.add({priority: HIGH_PRIORITY, account: peerAccount})
                # Skip if going back to source
                elif peerAccount == mSrcAccount:
                    continue
                else:
                    # Rank by paths out
                    pathsOut = getPathsOut(endPathAsset, peerAccount)
                    if pathsOut > 0:
                        candidates.add({priority: pathsOut, account: peerAccount})

            # Sort and select top candidates
            candidates.sort(by: priority descending, then account ID descending)
            maxCandidates = 10 if endAccount != mSrcAccount else 50

            for candidate in candidates[:maxCandidates]:
                if continueCallback.shouldBreak():
                    return
                newPath = currentPath + [accountElement(candidate.account)]
                incompletePaths.add(newPath)

    if afADD_BOOKS in addFlags:
        if afOB_XRP in addFlags:
            # Only add book to XRP (includes domain filtering if mDomain is set)
            if not isOnXRP and app.orderBookDB.isBookToXRP(endAsset, mDomain):
                incompletePaths.add(currentPath.append(xrpBookElement))
        else:
            # Add all viable order books (includes domain filtering if mDomain is set)
            books = app.orderBookDB.getBooksByTakerPays(endAsset, mDomain)

            for book in books:
                if currentPath.hasSeen(xrpAccount(), book.out, book.out.getIssuer()):
                    continue
                if issueMatchesOrigin(book.out):
                    continue
                # afOB_LAST keeps only books whose output token matches the destination.
                # equalTokens compares currency/MPTID and ignores the issuer.
                if afOB_LAST in addFlags and not equalTokens(book.out, mDstAmount.asset()):
                    continue

                newPath = currentPath.append(bookElement(book))

                if isXRP(book.out):
                    if mDstAmount.isXRP():
                        mCompletePaths.add(newPath)  # Complete path
                    else:
                        incompletePaths.add(newPath)
                elif not currentPath.hasSeen(book.out.getIssuer(), book.out, book.out.getIssuer()):
                    if hasEffectiveDst and book.out.getIssuer() == mDstAccount and equalTokens(book.out, mDstAmount.asset()):
                        continue  # Skipped required issuer
                    elif book.out.getIssuer() == mEffectiveDst and book.out.asset == mDstAmount.asset():
                        mCompletePaths.add(newPath)  # Complete path
                    else:
                        # If the path already ends in an account, the issuer-bearing element
                        # replaces that trailing account rather than being appended after it.
                        incompletePaths.add(newPath.append(issuerAccount(book.out)))
```

## 4.5. OrderBookDB

`OrderBookDB` serves as an **in-memory index** that catalogs all available trading pairs (order books) on the XRP Ledger. This index enables path finding to rapidly discover which currency conversions are available without scanning the entire ledger for each query.

OrderBookDB maintains four separate indexes for both offer-based order books and AMM pools:

- **`allBooks`**: Maps each asset to all assets it can be traded for (includes both offers and AMMs in open order books)
- **`xrpBooks`**: Set of all assets that have a direct trading pair with XRP (subset of `allBooks` for fast XRP bridge lookups)
- **`domainBooks`**: Maps (asset, domainID) pairs to tradeable assets (permissioned order books with domain restrictions, offers only)
- **`xrpDomainBooks`**: Set of (asset, domainID) pairs that have direct XRP trading pairs in permissioned books (offers only)

### 4.5.1. OrderBookDB Construction

On startup, `OrderBookDB.update()` ([OrderBookDBImpl.cpp:91-219](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/app/ledger/OrderBookDBImpl.cpp#L91-L219)) scans the entire ledger looking for two types of entries:

**1. Order book directories (`ltDIR_NODE` with `sfExchangeRate`):**

Order book directories are created when offers are placed via `OfferCreate` transaction. Each directory represents a specific quality level (exchange rate) for a trading pair.

When OrderBookDB finds an order book directory (identified by `ltDIR_NODE` type with `sfExchangeRate` field at root), it extracts the trading pair:

- **TakerPays asset**: Retrieved from `sfTakerPaysCurrency` + `sfTakerPaysIssuer` fields (for IOUs and XRPs) or `sfTakerPaysMPT` field (for MPTs)
- **TakerGets asset**: Retrieved from `sfTakerGetsCurrency` + `sfTakerGetsIssuer` fields (for IOUs and XRPs) or `sfTakerGetsMPT` field (for MPTs)

The book is registered based on whether it has domain restrictions:

- **Without domain** (`sfDomainID` not present):
  - Registered in `allBooks`
  - If TakerGets is XRP: also registered in `xrpBooks`
- **With domain** (`sfDomainID` present):
  - Registered in `domainBooks` (indexed by asset + domainID)
  - If TakerGets is XRP: also registered in `xrpDomainBooks`

**2. AMM ledger entries (`ltAMM`):**

AMM instances are created by `AMMCreate` transaction and provide liquidity without traditional order book directories. Each AMM holds two assets and can facilitate trades in both directions.

When OrderBookDB finds an AMM (identified by `ltAMM` type), it extracts the two pool assets from `sfAsset` and `sfAsset2` fields. Both trading directions are registered:

- `asset1 -> asset2` is registered in `allBooks`
- `asset2 -> asset1` is registered in `allBooks`
- If either asset is XRP, the other asset is also registered in `xrpBooks`

Unlike offer-based order books, AMMs are discovered directly from their `ltAMM` ledger entries without needing order book directory entries (`ltDIR_NODE`). Both offer-based and AMM-based liquidity are indexed together, allowing path finding to treat them uniformly when searching for currency conversion options.

## 4.6. AssetCache

`AssetCache` is an **in-memory cache** that provides fast access to trust line and MPT information during path finding. Each AssetCache is tied to a specific ledger view. For `path_find` subscriptions, a single AssetCache is shared across all path finding operations during batch processing (when a ledger changes and there are multiple `path_find` connections open), then deallocated when the batch completes. For one-shot requests like `ripple_path_find`, a new AssetCache is created for each request.

Assets (trust lines and MPTs) are fetched and cached on-demand for specific accounts as path finding explores the network.

The AssetCache provides two query methods:
- `getRippleLines(accountID, direction)`[^get-ripple-lines-impl]: Returns trust lines for an account. The `direction` parameter controls filtering:
  - `LineDirection::outgoing`: returns all trust lines for the account.
  - `LineDirection::incoming`: returns only trust lines where the account does **not** have NoRipple set on its side[^get-trust-line-items-filter].

  To avoid storing two copies per account, the cache keeps at most one set[^asset-cache-superset]:
  - If the full (`outgoing`) set is already cached when `incoming` is requested, the full set is returned. `addLink` relies on its per-candidate NoRipple check to filter out the extra trust lines.
  - If the `incoming` subset is cached when `outgoing` is requested, the subset is discarded and the full set is rebuilt[^asset-cache-rebuild].
- `getMPTs(accountID)`: Returns MPTs held by an account

[^get-ripple-lines-impl]: AssetCache::getRippleLines: [`AssetCache.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/AssetCache.cpp#L38-L106)
[^get-trust-line-items-filter]: getTrustLineItems filters by direction, excluding trust lines with NoRipple when incoming: [`TrustLine.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/TrustLine.cpp#L61)
[^asset-cache-rebuild]: AssetCache discards incoming subset when outgoing is requested: [`AssetCache.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/AssetCache.cpp#L66-L76)

# 5. Path Ranking

After path discovery completes, the pathfinder must evaluate which paths are worth using. Not all discovered paths have sufficient liquidity or good exchange rates - some may have already been consumed by the default path, while others may be too inefficient to be useful.

## 5.1. computePathRanks

The `computePathRanks` function evaluates path quality and liquidity by simulating payment execution through the Flow engine.

**Parameters:**

| Parameter          | Description                                           | Required |
|--------------------|-------------------------------------------------------|----------|
| `maxPaths`         | Maximum number of paths to rank and return            | ✅        |
| `continueCallback` | Optional callback to check if ranking should continue | ❌        |

The function performs two key steps:

**1. Account for the default path**

The default path is the direct route between source and destination that Flow always attempts first (unless the `tfNoRippleDirect` flag is set). Before ranking discovered paths, path finding must determine how much liquidity the default path provides.

To measure this, `computePathRanks` calls `RippleCalc.rippleCalculate()` with an empty path set and partial payment enabled[^default-path-partial], which in turn calls Flow and tests only the default path. Partial payment is enabled so the default path can deliver whatever liquidity it has, even if it cannot cover the full amount. RippleCalc simulates payment execution and returns:
- `actualAmountIn` - How much was consumed from the source
- `actualAmountOut` - How much was delivered to the destination
- Result code indicating success or failure

If the default path succeeds, its delivery is subtracted from `mRemainingAmount`[^remaining-amount-init] to calculate the additional liquidity still needed beyond what the default path provides.

[^default-path-partial]: Default path tested with partial payment enabled: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L462)
[^remaining-amount-init]: `mRemainingAmount` initialized via `convertAmount`, which returns the largest possible amount in convert-all mode or the destination amount otherwise: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L454) (`convertAmount` defined in [`PathfinderUtils.h`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/PathfinderUtils.h#L19-L26))

By accounting for the default path first, path finding ensures that discovered paths are evaluated for their **incremental value** - what they contribute beyond the baseline liquidity that Flow will attempt anyway.

**2. Rank all discovered paths**

With `mRemainingAmount` calculated, `computePathRanks` calls `rankPaths` to evaluate all paths in `mCompletePaths`. Each path is tested by simulating its execution through Flow, and successful paths are scored based on quality (exchange rate), liquidity (capacity), and length (number of hops). These rankings determine which paths `getBestPaths` will ultimately select for the payment.

### 5.1.1. computePathRanks Pseudo-Code

```python
def computePathRanks(maxPaths: int, continueCallback):
    # convertAmount returns the largest possible amount if convert_all_ is true (to find max liquidity)
    # otherwise returns mDstAmount unchanged
    mRemainingAmount = convertAmount(mDstAmount, convert_all_)

    # The default path is the direct path that always exists (source -> destination)
    # We test it first to see how much it can deliver, then rank additional paths
    # based on what they add beyond the default
    sandbox = PaymentSandbox(mLedger)
    inputs = Input(partialPaymentAllowed=True)
    rc = RippleCalc.rippleCalculate(
        view=sandbox,
        maxAmountIn=mSrcAmount,
        deliver=mRemainingAmount,
        account=mDstAccount,
        issuer=mSrcAccount,
        paths=[],  # Empty path set = test default path only
        domain=mDomain,
        inputs=inputs
    )

    if rc.success():
        mRemainingAmount -= rc.actualAmountOut

    # Rank all found paths
    rankPaths(maxPaths, mCompletePaths, mPathRanks, continueCallback)
```

### 5.1.2. rippleCalculate Pseudo-Code

`RippleCalc.rippleCalculate()` is a wrapper function that calls the [Flow engine](../flow/README.md) to simulate payment execution.

**Parameters:**

| Parameter     | Description                                                                                                                                                                                                       | Required |
|---------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------|
| `view`        | PaymentSandbox view of the ledger for simulation                                                                                                                                                                  | ✅        |
| `maxAmountIn` | Maximum amount willing to spend from source (SendMax)                                                                                                                                                             | ✅        |
| `deliver`     | Amount to deliver to destination                                                                                                                                                                                  | ✅        |
| `account`     | Destination account ID                                                                                                                                                                                            | ✅        |
| `issuer`      | Source account ID                                                                                                                                                                                                 | ✅        |
| `paths`       | Set of paths to test                                                                                                                                                                                              | ✅        |
| `domain`      | Optional domain ID                                                                                                                                                                                                | ❌        |
| `inputs`      | Optional Input struct containing flags: `defaultPathsAllowed` (whether to test default path), `partialPaymentAllowed` (whether partial delivery is acceptable), `limitQuality` (whether to enforce quality limit) | ❌        |

```python
def rippleCalculate(view, maxAmountIn, deliver, account, issuer, paths, domain, inputs=None):
    # Create sandbox for simulation
    sandbox = PaymentSandbox(view)

    # Extract parameters from inputs or use defaults
    defaultPaths = inputs.defaultPathsAllowed if inputs else True
    partialPayment = inputs.partialPaymentAllowed if inputs else False

    # Calculate quality limit if requested
    qualityLimit = None
    if inputs and inputs.limitQuality and maxAmountIn > 0:
        qualityLimit = Quality(maxAmountIn / deliver)

    # Determine sendMax (if different from source account's native currency)
    sendMax = maxAmountIn if (maxAmountIn >= 0 or
                               maxAmountIn.currency != deliver.currency or
                               maxAmountIn.issuer != issuer) else None

    # Call Flow engine to execute payment simulation
    flowResult = flow(
        sandbox,
        deliver,
        issuer,
        account,
        paths,
        defaultPaths=defaultPaths,
        partialPayment=partialPayment,
        limitQuality=qualityLimit,
        sendMax=sendMax,
        domain=domain
    )

    # Commit the simulated consumption back to the caller's view, so that sequential
    # rippleCalculate calls (e.g. the two calls in getPathLiquidity) measure liquidity
    # incrementally on the depleted state rather than double-counting it.
    sandbox.apply(view)

    return {
        result: flowResult.result,
        actualAmountIn: flowResult.actualAmountIn,
        actualAmountOut: flowResult.actualAmountOut
    }
```

## 5.2. rankPaths

The `rankPaths` function evaluates each discovered path by testing its liquidity and quality, then sorts them to identify the best paths for the payment.

**Parameters:**

| Parameter          | Description                                                    | Required |
|--------------------|----------------------------------------------------------------|----------|
| `maxPaths`         | Maximum number of paths to rank                                | ✅        |
| `paths`            | Set of complete paths to evaluate (typically `mCompletePaths`) | ✅        |
| `rankedPaths`      | Output vector where ranked paths are stored                    | ✅        |
| `continueCallback` | Optional callback to check if ranking should continue          | ❌        |

The function first calculates a minimum liquidity threshold that each path must meet to be worth including. This threshold serves as a quality filter, preventing the pathfinder from wasting computational resources ranking paths that contribute only negligible amounts to the payment.

When not in convert-all mode, this threshold is `dstAmount / (maxPaths + 2)`, where `maxPaths` is hardcoded to **4** in the `xrpld` implementation. This ensures each path can deliver at least a meaningful fraction of the destination amount. In convert-all mode, the threshold is set to the largest possible amount to find maximum available liquidity.

Convert-all mode is triggered when the destination amount equals the maximum possible value for that currency (checked via `convertAllCheck(mDstAmount)`). This mode is used when discovering maximum available liquidity rather than targeting a specific amount - for example, when a user wants to convert their entire balance of one currency to another. In convert-all mode, path finding prioritizes liquidity over quality, finding paths that can move the most value regardless of exchange rates.

For each path in the input set, `rankPaths` calls `getPathLiquidity` to simulate its execution and measure how much it can deliver. Paths that fail or cannot meet the minimum threshold are discarded. Successful paths are recorded as `PathRank` entries containing their quality (exchange rate), liquidity (capacity), length (hop count), and original index.

Finally, the function sorts all ranked paths using multiple criteria in order of importance: quality (better exchange rates first, unless in convert-all mode), liquidity (higher capacity first), length (shorter paths first), and index (as a tie breaker). This sorted ranking determines which paths `getBestPaths` will ultimately select for the payment.

### 5.2.1. rankPaths Pseudo-Code

```python
def rankPaths(maxPaths, paths, rankedPaths, continueCallback):
    rankedPaths.clear()

    if convert_all_:
        minDstAmount = largestAmount(dstAmount)
    else:
        minDstAmount = dstAmount / (maxPaths + 2)

    for i, path in enumerate(paths):
        if continueCallback.shouldBreak():
            return
        
        ter, liquidity, quality = getPathLiquidity(path, minDstAmount)

        if ter == tesSUCCESS:
            rankedPaths.add({
                quality: quality,
                length: len(path),
                liquidity: liquidity,
                index: i
            })

    # Sort by quality, liquidity, length
    rankedPaths.sort(key=lambda rank: (
        rank.quality if not convert_all_ else 0,  # Quality first (unless convert_all_)
        -rank.liquidity,  # Higher liquidity better (negative for desc sort)
        rank.length,      # Shorter better
        -rank.index       # Tie breaker
    ))
```

## 5.3. getPathLiquidity

The `getPathLiquidity` function determines how much liquidity a single path can provide by simulating its execution through the Flow engine.

The [**Flow engine**](../flow/README.md) takes paths and converts them into executable operations called **strands**. A strand is a sequence of **steps**, where each step is a concrete action that moves value between path elements. Path finding uses Flow to simulate execution and measure how much liquidity each path can actually deliver.

**Parameters:**

| Parameter      | Description                                                  | Required |
|----------------|--------------------------------------------------------------|----------|
| `path`         | The path to test for liquidity                               | ✅        |
| `minDstAmount` | Minimum amount the path must deliver to be considered viable | ✅        |

The function calls `RippleCalc.rippleCalculate()` to simulate payment execution along the path, with default paths explicitly disabled so only the specific path's liquidity is measured[^getpathliq-no-default]. RippleCalc uses the Flow engine to execute a payment simulation on a sandbox ledger (a copy of the ledger that can be modified without affecting the real ledger state), returning how much was consumed from the source (`actualAmountIn`), how much was delivered to the destination (`actualAmountOut`), and whether the payment succeeded. The first call tests whether the path can deliver at least `minDstAmount`. In convert-all mode, partial payment is allowed to find the maximum available liquidity. In normal mode, the path must deliver exactly the minimum amount or it's rejected.

[^getpathliq-no-default]: Default paths disabled in getPathLiquidity: [`Pathfinder.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/xrpld/rpc/detail/Pathfinder.cpp#L394)

If the first call succeeds and we are not in convert-all mode, the function makes a second call to probe for additional liquidity beyond the minimum. This second call attempts to deliver `(dstAmount - amountOut)` with partial payment allowed, discovering how much more the path can provide beyond the minimum threshold.

The function returns the total liquidity the path can deliver and its initial quality (exchange rate), calculated as `actualAmountIn / actualAmountOut` from the first call. Paths that fail to meet the minimum threshold return an error code instead.

### 5.3.1. getPathLiquidity Pseudo-Code

```python
def getPathLiquidity(path, minDstAmount):
    pathSet = [path]
    sandbox = PaymentSandbox(ledger)

    # Test minimum liquidity (default paths disabled, test only this path)
    inputs = Input(defaultPathsAllowed=False, partialPaymentAllowed=convert_all_)
    rc = RippleCalc.rippleCalculate(
        view=sandbox,
        maxAmountIn=srcAmount,
        deliver=minDstAmount,
        account=dstAccount,
        issuer=srcAccount,
        paths=pathSet,
        domain=domain,
        inputs=inputs
    )

    if rc.result != tesSUCCESS:
        return rc.result

    quality = rc.actualAmountIn / rc.actualAmountOut
    amountOut = rc.actualAmountOut

    if not convert_all_:
        # Test remaining liquidity
        inputs2 = Input(defaultPathsAllowed=False, partialPaymentAllowed=True)
        rc = RippleCalc.rippleCalculate(
            view=sandbox,
            maxAmountIn=srcAmount,
            deliver=dstAmount - amountOut,
            account=dstAccount,
            issuer=srcAccount,
            paths=pathSet,
            domain=domain,
            inputs=inputs2
        )

        if rc.result == tesSUCCESS:
            amountOut += rc.actualAmountOut

    return (tesSUCCESS, amountOut, quality)
```

# 6. Path Selection

After discovering and ranking paths, `getBestPaths` selects the optimal set of paths to use for the payment.

In the [RPC layer](#7-rpc-requests), path finding may be called multiple times for the same payment as the ledger state changes (e.g., in `path_find` subscriptions that continuously update). When path finding runs again, previously discovered paths are passed in as `extraPaths` so they can be merged with newly discovered paths, ensuring the best overall set is selected.

`getBestPaths` takes:

| Parameter           | Description                                                   | Required |
|---------------------|---------------------------------------------------------------|----------|
| `maxPaths`          | Maximum number of paths to return                             | ✅        |
| `fullLiquidityPath` | Output parameter for a path that can handle full amount alone | ✅        |
| `extraPaths`        | Paths from previous path finding runs (empty for first run)   | ✅        |
| `srcIssuer`         | Source issuer for validation                                  | ✅        |
| `continueCallback`  | Optional callback to check if search should continue          | ❌        |

The function works by merging two sets of ranked paths:
1. **`mPathRanks`** - Paths discovered in the current path finding run (already ranked by `computePathRanks`)
2. **`extraPathRanks`** - Paths from `extraPaths`, which are ranked by calling `rankPaths` at the start of this function

At each iteration, the function selects the best path from either set (prioritizing better quality, then better liquidity). If both quality and liquidity are identical, both paths are advanced to handle potential duplicates. Selected paths go through issuer constraint validation and slot management rules before being added to the result.

**Issuer Constraint Validation:**

During path discovery, the Pathfinder searches for paths of a specific currency (like USD) without constraining which issuer. This allows it to discover all possible USD paths efficiently. However, in the [RPC layer](#7-rpc-requests), path finding is called separately for each source asset the sender holds (USD from IssuerA, USD from IssuerB, etc.). When `getBestPaths` is called for a specific issuer's currency, it must filter the discovered paths to only include those that route through that specific issuer.

For example, when searching for paths using USD from IssuerA, the Pathfinder discovers all USD paths. But only paths that explicitly route through IssuerA should be returned - otherwise Alice might spend USD from a different issuer she doesn't hold, or the payment might fail.

To enforce this, `getBestPaths` validates paths from the discovered set (not extra paths, which are assumed already validated): if the source currency issuer is not the source account itself, discovered paths must start with the issuer account element. Paths that don't start with the issuer are skipped - this includes default paths and paths from simpler path types like `"sfd"` that go directly to books without routing through an account first. When a path does start with the issuer, that initial issuer element is removed before being added to the result, since the Flow engine will add it back during path normalization.

**Path Slot Management:**

A "path slot" refers to one position in the result set - `getBestPaths` returns up to `maxPaths` paths, so there are `maxPaths` slots available to fill.

The function applies different selection rules depending on how many path slots remain. If more than one slot is available (`pathsLeft > 1`), it adds the path to the result, subtracts its liquidity from the remaining amount needed, and continues. If only one slot remains (`pathsLeft == 1`), it only adds the path if its liquidity can cover the entire remaining amount - this ensures the last path is useful. If no slots remain (`pathsLeft == 0`) but a path can handle the full `mDstAmount` by itself, it's saved as `fullLiquidityPath` for potential use as a single-path alternative.

## 6.1. getBestPaths Pseudo-Code

```python
def getBestPaths(maxPaths, fullLiquidityPath, extraPaths, srcIssuer, continueCallback):
    # Rank extra paths
    extraPathRanks = []
    rankPaths(maxPaths, extraPaths, extraPathRanks, continueCallback)

    bestPaths = []
    remaining = mRemainingAmount
    issuerIsSender = isXRP(srcAsset) or (srcIssuer == srcAccount)

    i = 0  # index into mPathRanks
    j = 0  # index into extraPathRanks

    # Merge and select best paths
    while i < len(mPathRanks) or j < len(extraPathRanks):
        # Reset per-iteration flags (the C++ declares these inside the loop body)
        usePath = False
        useExtra = False
        startsWithIssuer = False

        # Determine which path to use next
        if i >= len(mPathRanks):
            useExtra = True
        elif j >= len(extraPathRanks):
            usePath = True
        elif extraPathRanks[j].quality < mPathRanks[i].quality:
            useExtra = True
        elif extraPathRanks[j].quality > mPathRanks[i].quality:
            usePath = True
        elif extraPathRanks[j].liquidity > mPathRanks[i].liquidity:
            useExtra = True
        elif extraPathRanks[j].liquidity < mPathRanks[i].liquidity:
            usePath = True
        else:
            usePath = True
            useExtra = True  # Both might be same path

        rank = mPathRanks[i] if usePath else extraPathRanks[j]
        path = mCompletePaths[rank.index] if usePath else extraPaths[rank.index]

        if useExtra:
            j += 1
        if usePath:
            i += 1

        pathsLeft = maxPaths - len(bestPaths)

        if pathsLeft == 0 and not fullLiquidityPath.empty():
            break

        # Validate issuer constraint
        if not issuerIsSender and usePath:
            if isDefaultPath(path) or path[0].getAccountID() != srcIssuer:
                continue  # Skip paths that don't start with issuer
            startsWithIssuer = True

        # Apply selection rules
        if pathsLeft > 1 or (pathsLeft > 0 and rank.liquidity >= remaining):
            # Add to best paths
            pathsLeft -= 1
            remaining -= rank.liquidity
            bestPaths.add(removeIssuer(path) if startsWithIssuer else path)

        elif pathsLeft == 0 and rank.liquidity >= dstAmount and fullLiquidityPath.empty():
            # Found extra path that can handle full amount
            fullLiquidityPath = (removeIssuer(path) if startsWithIssuer else path)

    return bestPaths
```

# 7. RPC Requests

Both `path_find` and `ripple_path_find` RPCs support an optional `domain` parameter (256-bit hex string) for permissioned DEX functionality. When specified, path finding restricts order book queries to only include offers within the specified domain.

## 7.1. `ripple_path_find` RPC (Legacy)

The `ripple_path_find` RPC command is the legacy interface for one-shot path finding requests. It is **deprecated** but still supported.

**The request flow works as follows:**

1. Client sends a `ripple_path_find` request with source, destination, and amount
2. Server creates a path finding job and enqueues it
3. The RPC handler uses a **coroutine** that yields while waiting for path finding to complete
4. When path finding finishes, the coroutine resumes and returns the result

**continueCallback usage:**

`ripple_path_find` does NOT provide a `continueCallback` when calling `Pathfinder::findPaths()`. It runs the path finding synchronously without interruption checking. This is acceptable because:
- It's a one-shot request (not streaming updates)
- The coroutine mechanism already handles shutdown gracefully
- The search completes relatively quickly with typical search levels

## 7.2. `path_find` RPC

The `path_find` RPC command is the modern interface for path finding with three subcommands:

- `path_find create` - Creates a path finding subscription
- `path_find status` - Gets current status of an active subscription
- `path_find close` - Closes an active subscription

**The subscription flow works as follows:**

1. Client creates a subscription with `path_find create`
2. Server creates a `PathRequest` associated with the WebSocket connection
3. The `PathRequest` continuously updates as ledgers close, sending updates to the client
4. Client receives streaming path updates until they close the subscription or disconnect

**continueCallback usage:**

`path_find` subscriptions do provide a `continueCallback` to `Pathfinder::findPaths()`:

```cpp
auto continueCallback = [&getSubscriber, &request]() {
    return (bool)getSubscriber(request);
};
```

This callback:
- Returns `true` if the subscriber (WebSocket client) is still connected
- Returns `false` if the client has disconnected
- Allows path finding to abort immediately if the client is no longer listening

This is critical for subscriptions because:
- path finding can take significant time at high search levels
- Multiple subscriptions may be active simultaneously
- No point computing paths if the client disconnected

## 7.3. Source Currency Handling

Both `path_find` and `ripple_path_find` RPCs support the `source_currencies` parameter, which controls which currencies the pathfinder considers as potential sources for funding the payment.

The `source_currencies` handling happens in the `PathRequest` layer (RPC handler), **not** in the core Pathfinder algorithm. The PathRequest creates one Pathfinder instance per source currency, reusing it across different issuers of that currency:

```mermaid
flowchart TD
    RPC["path_find / ripple_path_find RPC"]
    PR["PathRequest<br/>(processes source_currencies parameter)"]
    PF1["Pathfinder #1<br/>(source currency: USD)"]
    PF2["Pathfinder #2<br/>(source currency: EUR)"]
    PF3["Pathfinder #3<br/>(source currency: XRP)"]
    EXEC1["findPaths() -> computePathRanks() -> getBestPaths()"]
    EXEC2["findPaths() -> computePathRanks() -> getBestPaths()"]
    EXEC3["findPaths() -> computePathRanks() -> getBestPaths()"]

    RPC --> PR
    PR -->|creates| PF1
    PR -->|creates| PF2
    PR -->|creates| PF3
    PF1 --> EXEC1
    PF2 --> EXEC2
    PF3 --> EXEC3
```

**When `source_currencies` is specified:**

The client provides an array of currency/issuer pairs (up to 18 currencies):

```json
{
  "source_currencies": [
    {"currency": "USD", "issuer": "rIssuer1..."},
    {"currency": "EUR", "issuer": "rIssuer2..."},
    {"currency": "XRP"},
    {"mpt_issuance_id": "00000001B2..."}
  ]
}
```

For each source asset:
1. PathRequest looks up or creates a Pathfinder for that source currency (the cache is keyed by currency, so the Pathfinder is reused across issuers of the same currency)
2. Path discovery (`findPaths`/`computePathRanks`) runs once per currency; path selection (`getBestPaths`) then runs per source asset, filtering the discovered paths to that asset's issuer
3. Results are collected in a hash map keyed by the issuer-qualified asset: `mContext[issue] = pathSet`

PathRequest does not create a separate Pathfinder for each issuer of an IOU; it reuses one Pathfinder per currency and runs a separate `getBestPaths` call per issuer to filter the shared discovered paths to that issuer.

**When `source_currencies` is NOT specified:**

PathRequest auto-discovers source currencies in this order:

1. **If `send_max` is provided**: Use only the `send_max` currency/issuer
2. **Otherwise, scan the source account** by calling `accountSourceAssets()`:
    - Always includes XRP
    - Scans all outgoing trust lines and MPTs from the source account
    - Includes a currency if either:
        - Account has positive balance (has asset to send), OR
        - Peer extends credit AND there's available credit remaining (can issue more)
    - Limited to 88 currencies maximum (`max_auto_src_cur`, limited at RPC layer)
    - Excludes currencies matching the destination currency (when source == destination account)
