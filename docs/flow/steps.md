# Index

- [1. Introduction](#1-introduction)
    - [1.1. Step Catalog](#11-step-catalog)
    - [1.2. Class Relationships](#12-class-relationships)
    - [1.3. Methods](#13-methods)
- [2. DirectStepI](#2-directstepi)
    - [2.1. Common Implementation](#21-common-implementation)
        - [2.1.1. `revImp` Implementation](#211-revimp-implementation)
            - [2.1.1.1. `revImp` Pseudo-Code](#2111-revimp-pseudo-code)
        - [2.1.2. `fwdImp` (Forward Pass) Implementation](#212-fwdimp-implementation)
            - [2.1.2.1. `fwdImp` Pseudo-Code](#2121-fwdimp-pseudo-code)
        - [2.1.3. Quality Functions](#213-quality-functions)
            - [2.1.3.1. Quality Functions Pseudo-Code](#2131-quality-functions-pseudo-code)
        - [2.1.4. `qualityUpperBound` Implementation](#214-qualityupperbound-implementation)
            - [2.1.4.1. `qualityUpperBound` Pseudo-Code](#2141-qualityupperbound-pseudo-code)
        - [2.1.5. `check` Implementation (Base Class)](#215-check-implementation)
    - [2.2. DirectIPaymentStep (Payment-Specific Implementation)](#22-directipaymentstep-payment-specific-implementation)
        - [2.2.1. `quality` Implementation](#221-quality-implementation)
            - [2.2.1.1. `quality` Pseudo-Code](#2211-quality-pseudo-code)
        - [2.2.2. `maxFlow` Implementation](#222-maxflow-implementation)
            - [2.2.2.1. `maxFlow` Pseudo-Code](#2221-maxflow-pseudo-code)
        - [2.2.3. `check` Implementation](#223-check-implementation)
    - [2.3. DirectIOfferCrossingStep (Offer Crossing-Specific Implementation)](#23-directioffercrossingstep-offer-crossing-specific-implementation)
        - [2.3.1. `quality` Implementation](#231-quality-implementation)
        - [2.3.2. `maxFlow` Implementation](#232-maxflow-implementation)
        - [2.3.3. `check` Implementation](#233-check-implementation)
- [3. XRPEndpointStep](#3-xrpendpointstep)
    - [3.1. `revImp` Implementation](#31-revimp-implementation)
        - [3.1.1. `revImp` Pseudo-Code](#311-revimp-pseudo-code)
    - [3.2. `fwdImp` Implementation](#32-fwdimp-implementation)
        - [3.2.1. `fwdImp` Pseudo-Code](#321-fwdimp-pseudo-code)
    - [3.3. `qualityUpperBound` Implementation](#33-qualityupperbound-implementation)
        - [3.3.1. `qualityUpperBound` Pseudo-Code](#331-qualityupperbound-pseudo-code)
    - [3.4. `check` Implementation](#34-check-implementation)
- [4. MPTEndpointStep](#4-mptendpointstep)
    - [4.1. `revImp` Implementation](#41-revimp-implementation)
       - [4.1.1. `revImp` Pseudo-Code](#411-revimp-pseudo-code)
    - [4.2. `fwdImp` Implementation](#42-fwdimp-implementation)
       - [4.2.1. `fwdImp` Pseudo-Code](#421-fwdimp-pseudo-code)
    - [4.3. `qualityUpperBound` Implementation](#43-qualityupperbound-implementation)
      - [4.3.1. `qualityUpperBound` Pseudo-Code](#431-qualityupperbound-pseudo-code)
    - [4.4. `check` Implementation](#44-check-implementation)
- [5. BookStep](#5-bookstep)
    - [5.1. `revImp` Implementation](#51-revimp-implementation)
        - [5.1.1. `revImp` Pseudo-Code](#511-revimp-pseudo-code)
    - [5.2. `fwdImp` Implementation](#52-fwdimp-implementation)
        - [5.2.1. `fwdImp` Pseudo-Code](#521-fwdimp-pseudo-code)
    - [5.3. `forEachOffer`](#53-foreachoffer)
        - [5.3.1. `forEachOffer` Pseudo-Code](#531-foreachoffer-pseudo-code)
        - [5.3.2. Transfer Rate Helper Functions](#532-transfer-rate-helper-functions)
            - [5.3.2.1. `getOfrInRate` Pseudo-Code](#5321-getofrinrate-pseudo-code)
            - [5.3.2.2. `getOfrOutRate` Pseudo-Code](#5322-getofroutrate-pseudo-code)
    - [5.4. AMM Integration](#54-amm-integration)
        - [5.4.1. Integration with BookStep](#541-integration-with-bookstep)
        - [5.4.2. Spot Price Quality and CLOB Comparison](#542-spot-price-quality-and-clob-comparison)
        - [5.4.3. Offer Generation Strategies](#543-offer-generation-strategies)
            - [5.4.3.1. `getOffer` Pseudo-Code](#5431-getoffer-pseudo-code)
            - [5.4.3.2. Multi-Path Mode: Fibonacci Sequence Sizing](#5432-multi-path-mode-fibonacci-sequence-sizing)
            - [5.4.3.3. Single-Path Mode: CLOB-Matching Sizing](#5433-single-path-mode-clob-matching-sizing)
    - [5.5. `qualityUpperBound` Implementation](#55-qualityupperbound-implementation)
        - [5.5.1. `qualityUpperBound` Method](#551-qualityupperbound-method)
            - [5.5.1.1. `qualityUpperBound` Pseudo-Code](#5511-qualityupperbound-pseudo-code)
        - [5.5.2. `tipOfferQuality` Helper Function](#552-tipofferquality-helper-function)
            - [5.5.2.1. `tipOfferQuality` Pseudo-Code](#552-tipofferquality-helper-function)
        - [5.5.3. `adjustQualityWithFees` - BookPaymentStep Implementation](#553-adjustqualitywithfees---bookpaymentstep-implementation)
            - [5.5.3.1. `adjustQualityWithFees` Pseudo-Code](#5531-adjustqualitywithfees-pseudo-code)
        - [5.5.4. `adjustQualityWithFees` - BookOfferCrossingStep Implementation](#554-adjustqualitywithfees---bookoffercrossingstep-implementation)
            - [5.5.4.1. `adjustQualityWithFees` Pseudo-Code](#5541-adjustqualitywithfees-pseudo-code)
    - [5.6. `check` Implementation](#56-check-implementation)

# 1. Introduction

The [payment engine](README.md) in `xrpld` uses **[Strands](README.md#11-strands-and-steps)** (sequences of steps) to move value from source to destination. Each **step** in a strand transfers value between accounts and order books.

The **fundamental types** of steps in `xrpld`:

1. **DirectStepI** - [IOU](../glossary.md#iou) transfer between accounts via trust line
2. **BookStep** - Asset conversion through order books and [AMM](../amms/README.md) pools
3. **XRPEndpointStep** - Strand endpoint for [XRP](../glossary.md#xrp) (first or last step only)
4. **MPTEndpointStep** - Strand endpoint for [MPT](../glossary.md#mpt) (first or last step only)

**How Steps Fit in the Flow Process:**

During [path to strand conversion](README.md#52-path-to-strand-conversion), Flow examines each pair of adjacent path elements to determine what type of step connects them. The [toStep function](README.md#53-step-generation) examines element characteristics (accounts vs order books, currency types, position in strand) and calls the appropriate factory function.

- `makeDirectStepI`[^make-directstepi] for account-to-account IOU transfers
- `makeBookStepIi`[^make-bookstepii], `makeBookStepIx`[^make-bookstepix], `makeBookStepXi`[^make-bookstepxi], `makeBookStepMm`[^make-bookstepmm], `makeBookStepMx`[^make-bookstepmx], `makeBookStepXm`[^make-bookstepxm], `makeBookStepMi`[^make-bookstepmi], `makeBookStepIm`[^make-bookstepim] for order book conversions
- `make_XRPEndpointStep`[^make-xrpendpointstep] for XRP source/destination steps
- `make_MPTEndpointStep`[^make-mptendpointstep] for MPT source/destination steps

Steps handle different asset type combinations - XRP, IOU and MPT. In `xrpld`, these are represented by different types (`XRPAmount`, `IOUAmount`, `MPTAmount`) with different arithmetic and precision characteristics.

**BookStep** uses template parameters to handle multiple asset type combinations. The factory functions (`makeBookStepIi`, `makeBookStepIx`, etc.) instantiate the same `BookStep` template with the appropriate type parameters based on the input and output asset types.

Flow executes in one of two contexts, determined by the transaction type that invokes it:

- **[Payment](../payments/README.md)**: Triggered by Payment transactions (and similar operations like check cashing or cross-chain transfers). Flow moves value from a source account to a destination account, potentially through multiple intermediate steps.

- **[Offer Crossing](../offers/README.md)**: Triggered when creating or modifying offers (OfferCreate/OfferCancel transactions). When a new offer is placed, it may immediately match against existing offers in the order book, causing Flow to execute the exchange.

The context affects step behavior throughout the flow process. Steps receive the context through the `offerCrossing` field in `StrandContext`, which factory functions use to instantiate either payment-specific or offer-crossing-specific implementations (e.g., `DirectIPaymentStep` vs `DirectIOfferCrossingStep`).

## 1.1. Step Catalog

| Alias                        | Concrete class (definition)    | Input       | Output      | Factory                 |
|------------------------------|--------------------------------|-------------|-------------|-------------------------|
| DirectStepI (payment)        | `DirectIPaymentStep`[^directipaymentstep]           | `IOUAmount`[^iouamount] | `IOUAmount` | `makeDirectStepI`[^make-directstepi]      |
| DirectStepI (offer crossing) | `DirectIOfferCrossingStep`[^directioffercrossingstep]     | `IOUAmount` | `IOUAmount` | `makeDirectStepI`      |
| BookStepII (payment)         | `BookPaymentStep`[^bookpaymentstep]              | `IOUAmount` | `IOUAmount` | `makeBookStepIi`[^make-bookstepii]       |
| BookStepII (offer crossing)  | `BookOfferCrossingStep`[^bookoffercrossingstep]        | `IOUAmount` | `IOUAmount` | `makeBookStepIi`       |
| BookStepIX (payment)         | `BookPaymentStep`              | `IOUAmount` | `XRPAmount`[^xrpamount] | `makeBookStepIx`[^make-bookstepix]       |
| BookStepIX (offer crossing)  | `BookOfferCrossingStep`        | `IOUAmount` | `XRPAmount` | `makeBookStepIx`       |
| BookStepXI (payment)         | `BookPaymentStep`              | `XRPAmount` | `IOUAmount` | `makeBookStepXi`[^make-bookstepxi]       |
| BookStepXI (offer crossing)  | `BookOfferCrossingStep`        | `XRPAmount` | `IOUAmount` | `makeBookStepXi`       |
| BookStepMM (payment)         | `BookPaymentStep`              | `MPTAmount`[^mptamount] | `MPTAmount` | `makeBookStepMm`[^make-bookstepmm]       |
| BookStepMM (offer crossing)  | `BookOfferCrossingStep`        | `MPTAmount` | `MPTAmount` | `makeBookStepMm`       |
| BookStepMX (payment)         | `BookPaymentStep`              | `MPTAmount` | `XRPAmount` | `makeBookStepMx`[^make-bookstepmx]       |
| BookStepMX (offer crossing)  | `BookOfferCrossingStep`        | `MPTAmount` | `XRPAmount` | `makeBookStepMx`       |
| BookStepXM (payment)         | `BookPaymentStep`              | `XRPAmount` | `MPTAmount` | `makeBookStepXm`[^make-bookstepxm]       |
| BookStepXM (offer crossing)  | `BookOfferCrossingStep`        | `XRPAmount` | `MPTAmount` | `makeBookStepXm`       |
| BookStepMI (payment)         | `BookPaymentStep`              | `MPTAmount` | `IOUAmount` | `makeBookStepMi`[^make-bookstepmi]       |
| BookStepMI (offer crossing)  | `BookOfferCrossingStep`        | `MPTAmount` | `IOUAmount` | `makeBookStepMi`       |
| BookStepIM (payment)         | `BookPaymentStep`              | `IOUAmount` | `MPTAmount` | `makeBookStepIm`[^make-bookstepim]       |
| BookStepIM (offer crossing)  | `BookOfferCrossingStep`        | `IOUAmount` | `MPTAmount` | `makeBookStepIm`       |
| XRPEndpoint (payment)        | `XRPEndpointPaymentStep`[^xrpendpointpaymentstep]       | `XRPAmount` | `XRPAmount` | `make_XRPEndpointStep`[^make-xrpendpointstep]  |
| XRPEndpoint (offer crossing) | `XRPEndpointOfferCrossingStep`[^xrpendpointoffercrossingstep] | `XRPAmount` | `XRPAmount` | `make_XRPEndpointStep`  |
| MPTEndpoint (payment)        | `MPTEndpointPaymentStep`[^mptendpointpaymentstep]       | `MPTAmount` | `MPTAmount` | `make_MPTEndpointStep`[^make-mptendpointstep]  |
| MPTEndpoint (offer crossing) | `MPTEndpointOfferCrossingStep`[^mptendpointoffercrossingstep] | `MPTAmount` | `MPTAmount` | `make_MPTEndpointStep`  |

[^directipaymentstep]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L231-L279)
[^directioffercrossingstep]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L282-L337)
[^bookpaymentstep]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L282-L369)
[^bookoffercrossingstep]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L373-L557)
[^xrpendpointpaymentstep]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L167-L187)
[^xrpendpointoffercrossingstep]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L190-L241)
[^mptendpointpaymentstep]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L242-L281)
[^mptendpointoffercrossingstep]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L284-L326)
[^iouamount]: [`IOUAmount.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/IOUAmount.h#L24-L91)
[^xrpamount]: [`XRPAmount.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/XRPAmount.h#L19-L237)
[^mptamount]: [`MPTAmount.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/protocol/MPTAmount.h#L16-L83)
[^make-directstepi]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L922-L947)
[^make-bookstepii]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1521-L1525)
[^make-bookstepix]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1527-L1531)
[^make-bookstepxi]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1533-L1537)
[^make-bookstepmm]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1540-L1544)
[^make-bookstepmx]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1558-L1562)
[^make-bookstepxm]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1564-L1568)
[^make-bookstepmi]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1546-L1550)
[^make-bookstepim]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1552-L1556)
[^make-xrpendpointstep]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L394-L415)
[^make-mptendpointstep]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L911-L936)

## 1.2. Class Relationships

```mermaid
classDiagram
    class Step {
        <<abstract>>
        +rev()
        +fwd()
        +qualityUpperBound()
    }
    class StepImp {
        <<abstract>>
        #revImp()
        #fwdImp()
    }
    Step <|-- StepImp
    class DirectStepI
    StepImp <|-- DirectStepI
    DirectStepI <|-- DirectIPaymentStep
    DirectStepI <|-- DirectIOfferCrossingStep
    class BookStep
    StepImp <|-- BookStep
    BookStep <|-- BookPaymentStep
    BookStep <|-- BookOfferCrossingStep
    class XRPEndpointStep
    StepImp <|-- XRPEndpointStep
    XRPEndpointStep <|-- XRPEndpointPaymentStep
    XRPEndpointStep <|-- XRPEndpointOfferCrossingStep
    class MPTEndpointStep
    StepImp <|-- MPTEndpointStep
    MPTEndpointStep <|-- MPTEndpointPaymentStep
    MPTEndpointStep <|-- MPTEndpointOfferCrossingStep
```

`StepImp` is a CRTP (Curiously Recurring Template Pattern) base class template that implements the polymorphic `rev()` and `fwd()` interface methods by dispatching to type-specific `revImp()` and `fwdImp()` implementations in the derived classes.

## 1.3. Methods

The key methods implemented by each step class and used during [strand evaluation](README.md#7-single-strand-evaluation-strandflow) are:

### `revImp` (Reverse Pass)

The reverse pass calculates required inputs given a desired output and simulates the transfer by updating the payment sandbox. When the Flow engine needs to deliver a specific amount to the destination, it calls `revImp` on each step in the strand, working backwards from the destination. Each step determines available liquidity, applies quality adjustments and transfer fees, executes the transfer on the sandbox, caches the calculated amounts, and returns the actual input and output amounts. If the step cannot provide the full requested output due to liquidity constraints, it becomes a limiting step and returns reduced amounts. 

### `fwdImp` (Forward Pass)

When the reverse pass encounters a limiting step, the Flow engine calls `fwdImp` on each step after the limiting step, working forwards from the limiting step toward the destination. The step recalculates available liquidity, applies quality adjustments and transfer fees based on the input amount, executes the transfer on the sandbox, and updates the cache to the minimum of the forward calculation and the cached reverse pass values. This limiting mechanism ensures the forward pass never delivers more than the reverse pass calculated, preventing over-delivery due to rounding differences. The step returns the cached input and output amounts.

### `qualityUpperBound`

The `qualityUpperBound` method estimates a step's exchange rate without executing it. Each step implements this method to return its quality based on current ledger state. During [iterative strand evaluation](README.md#62-qualityupperbound), the Flow engine's `qualityUpperBound` function (in StrandFlow.h) calls each step's `qualityUpperBound` method and combines them to calculate the composite strand quality, allowing it to sort strands and prioritize better paths first. The actual quality achieved during execution may be worse (due to rounding or liquidity constraints) but should never be significantly better, making this a safe upper bound for ranking strands.

### `check`

The `check` method validates step requirements before the strand executes. If `check` returns an error, the entire strand is rejected before any liquidity evaluation occurs.

# 2. DirectStepI

DirectStepI handles [IOU](../glossary.md#iou) transfers along a single trust line between two accounts without currency conversion. The step manages the transfer while accounting for trust line quality settings, transfer fees, credit limits, and debt direction. The step must determine whether the source is issuing new IOUs to the destination or whether the source is redeeming IOUs back to the destination, as this affects which fees and quality adjustments apply.

The `DirectStepI<TDerived>` template base class provides common implementation for `revImp`, `fwdImp`, `qualityUpperBound`, and base `check` validation. The concrete derived classes (`DirectIPaymentStep` and `DirectIOfferCrossingStep`) provide specialized implementations for `quality`, `maxFlow`, and context-specific `check` validation.

The pseudocode below refers to a shared set of step state:

```python
# DirectStepI state (one trust-line hop moving `currency_` from src_ to dst_)
src_, dst_: AccountID      # the two ends of the trust line this step moves value across
currency_:  Currency       # the IOU being moved
prevStep_:  Step or None   # upstream step, used to propagate quality and debt direction
isLast_:    bool           # whether this is the strand's final step (caps destination quality)
cache_:     RevResult      # reverse-pass result (in, srcToDst, out, debtDir), reused by fwdImp
# Qualities are ratios where QUALITY_ONE means 1:1. roundUp/roundDown pick the rounding direction.
```

## 2.1. Common Implementation

### 2.1.1. `revImp` Implementation

Given a requested output amount, this function[^directstepi-revimp]:
1. Determines the maximum available liquidity on the trust line (`maxSrcToDst`) and the debt direction via [`maxFlow`](#222-maxflow-implementation) (payment - enforces trust line limits) or [`maxFlow`](#232-maxflow-implementation) (offer crossing - may exceed limits for destination step)
2. Calculates quality adjustments from both source and destination accounts via [`qualities`](#213-quality-functions) which calls [`quality`](#221-quality-implementation) (payment - respects QualityIn/QualityOut in certain conditions) or [`quality`](#231-quality-implementation) (offer crossing - always returns QUALITY_ONE)
3. Computes how much must flow on the trust line to produce the desired output, accounting for destination quality
4. Calculates the required input amount, accounting for source quality and transfer fees
5. Makes the payment by calling `directSendNoFee` to update the trust line in the sandbox
6. Returns the actual input required and output delivered

#### 2.1.1.1. `revImp` Pseudo-Code

```python
def revImp(sb, out):
    maxSrcToDst, debtDir = maxFlow(sb, out)

    # srcQOut stands for Source Quality Out
    #   - the quality the source applies for sending funds
    # dstQIn stands for Destination Quality In
    #   - the quality the destination applies for receiving funds
    # Each quality can be a discount or a premium.
    srcQOut, dstQIn = qualities(sb, debtDir, REVERSE)
    if maxSrcToDst <= 0:
        cache_ = zero
        return (0, 0)

    # This calculates how much needs to flow on the trust line to produce the desired output,
    # accounting for the destination's quality-in adjustment
    #   - QUALITY_ONE is 1'000'000'000 and dstQIn is presented in terms where 95% is 950'000'000
    srcToDst = roundUp(out * QUALITY_ONE / dstQIn)

    if srcToDst <= maxSrcToDst:  # Non-limiting case, we have enough liquidity
        # Calculate how much input is required for the promised output. The full transformation is:
        #   in --[srcQOut]-> srcToDst --[dstQIn]-> out
        in = roundUp(srcToDst * srcQOut / QUALITY_ONE)
        move(sb, src_ -> dst_, srcToDst)
        cache_ = (in, srcToDst, out, debtDir)
        return (in, out)

    # We don't have enough liquidity, so we will require as much input as will produce maximum output we can have
    in = roundUp(maxSrcToDst * srcQOut / QUALITY_ONE)
    actualOut = roundDown(maxSrcToDst * dstQIn / QUALITY_ONE)
    move(sb, src_ -> dst_, maxSrcToDst)
    cache_ = (in, maxSrcToDst, actualOut, debtDir)
    return (in, actualOut)
```

[^directstepi-revimp]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L503-L567)

### 2.1.2. `fwdImp` Implementation

The `fwdImp` method[^directstepi-fwdimp] executes the forward pass for a DirectStep, determining how much output can be produced from a given input amount. This method is always called after `revImp` has been executed.

The implementation mirrors `revImp` but works in the opposite direction. Given an input amount, it recalculates liquidity and qualities, computes the output, and makes the payment. The key difference is that `fwdImp` uses the cached `srcToDst` from the reverse pass as a constraint: it calls `maxFlow` with this cached value, uses `setCacheLimiting` to ensure the forward pass doesn't exceed what the reverse pass promised, and makes the payment using the cached `srcToDst` rather than the freshly calculated amount. This ensures consistency between passes despite potential rounding differences.

#### 2.1.2.1. `fwdImp` Pseudo-Code

```python
def fwdImp(sb, in):
    # This method should never be run before revImp
    assert cache_ is set

    # Recalculate liquidity based on cached srcToDst from reverse pass
    maxSrcToDst, debtDir = maxFlow(sb, cache_.srcToDst)
    srcQOut, dstQIn = qualities(sb, debtDir, FORWARD)
    if maxSrcToDst <= 0:
        cache_ = zero
        return (0, 0)

    # Now, we are calculating srcToDst differently, since we care about the `in` amount
    srcToDst = roundDown(in * QUALITY_ONE / srcQOut)
    if srcToDst <= maxSrcToDst:  # Non-limiting case, we have enough liquidity
        # Calculate how much output is provided for the required input.
        out = roundDown(srcToDst * dstQIn / QUALITY_ONE)
    else:
        # We don't have full liquidity
        in = roundUp(maxSrcToDst * srcQOut / QUALITY_ONE)
        out = roundDown(maxSrcToDst * dstQIn / QUALITY_ONE)

    # There could be a difference, due to rounding, in reverse and forward pass. setCacheLimiting
    # updates cache.in, cache.srcToDst and cache.out, but uses smaller (more conservative) values
    # if they changed from rev to fwd passes.
    setCacheLimiting(in, srcToDst, out, debtDir)
    # Make the payment using the cached srcToDst value
    move(sb, src_ -> dst_, cache_.srcToDst)

    # This ensures that the forward pass never returns better rates than reverse pass promised
    return (cache_.in, cache_.out)
```

[^directstepi-fwdimp]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L617-L682)

### 2.1.3. Quality Functions

The `qualities` function[^directstepi-qualities] is called by both `revImp` and `fwdImp` to determine the quality adjustments that apply to this step.

It returns two quality values:
- `srcQOut`: Source qualityOut - the quality the source applies when sending funds
- `dstQIn`: Destination qualityIn - the quality the destination applies when receiving funds

These qualities are used to calculate how input amounts transform to output amounts accounting for transfer fees and quality adjustments.

The `strandDir` (strand direction) parameter controls whether the previous step's debt direction is fetched from cache or calculated fresh:
- `StrandDirection::Reverse` (reverse pass): Calculate previous step's debt direction from current ledger state
- `StrandDirection::Forward` (forward pass): Use cached debt direction from the reverse pass

The `qualities` function delegates to two specialized helpers based on the current step's debt direction:

**`qualitiesSrcRedeems()`**[^directstepi-qualitiessrcredeems]: Called when the source is redeeming (holder sending to issuer)

[^directstepi-qualitiessrcredeems]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L738-L749)

When a holder returns IOUs to their issuer:

- **Destination quality (`dstQIn`)**: Always `QUALITY_ONE` because issuers receive their own IOUs at face value
- **Source quality (`srcQOut`)**: Must be at least as high as the previous step's qualityIn to prevent quality improvement along the path

Quality must not improve as value flows through the path. If the previous step has qualityIn = 1.02 (needs 102 to deliver 100), those 102 units become this step's input. If this step's qualityOut = 1.0, it would only require 100 at its source to transfer 100 across the trust line, making 2 units vanish. By setting `srcQOut = max(prevStepQIn, this.qualityOut)`, we ensure this step requires at least 102 at its input to match what the previous step delivered at its output, preventing the path from offering better quality than any individual step provides.

**`qualitiesSrcIssues(prevStepDebtDirection)`**[^directstepi-qualitiessrcissues]: Called when the source is issuing (issuer sending to holder)

[^directstepi-qualitiessrcissues]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L754-L771)

When an issuer sends IOUs to a holder:

- **Source quality (`srcQOut`)**: Applies the issuer's transfer rate (an account-level transfer fee set via the `TransferRate` field on the issuer's account) only when the previous step redeemed to this issuer. This charges the fee when IOUs flow through the issuer as an intermediary (holder-to-holder transfers), but not when IOUs originate from the issuer (direct issuer-to-holder payments). The transfer rate is retrieved via `transferRate(sb, src_)`, which reads the issuer's account settings.
- **Destination quality (`dstQIn`)**: Uses the trust line's qualityIn setting (a trust-line-level quality adjustment that allows the receiving holder to demand better than 1:1 exchange rates). However, if this is the final step in the path (`isLast_`), we cap this at `QUALITY_ONE` to prevent the payment recipient from demanding better than face value.

#### 2.1.3.1. Quality Functions Pseudo-Code

```python
def qualities(sb, debtDir, strandDir):
    if debtDir == REDEEMS:
        # Source is holder sending to issuer
        return qualitiesSrcRedeems(sb)
    # Source is issuer sending to holder
    # Need to check previous step's debt direction to determine if transfer fee applies
    prevDebtDir = prevStep_.debtDirection(sb, strandDir) if prevStep_ else ISSUES
    return qualitiesSrcIssues(sb, prevDebtDir)

def qualitiesSrcRedeems(sb):
    # This is the first step
    if not prevStep_:
        return (QUALITY_ONE, QUALITY_ONE)

    prevStepQIn = prevStep_.lineQualityIn(sb)
    srcQOut = quality(sb, OUT)
    # Use the worse of the two qualities (higher value = worse for sender)
    if prevStepQIn > srcQOut:
        srcQOut = prevStepQIn
    return (srcQOut, QUALITY_ONE)

def qualitiesSrcIssues(sb, prevDebtDir):
    # Charge transfer rate when issuing and previous step redeems
    if prevDebtDir == REDEEMS:
        srcQOut = transferRate(sb, src_)
    else:
        srcQOut = QUALITY_ONE

    dstQIn = quality(sb, IN)
    # If this is the last step, cap destination quality at QUALITY_ONE
    if isLast_ and dstQIn > QUALITY_ONE:
        dstQIn = QUALITY_ONE
    return (srcQOut, dstQIn)
```

[^directstepi-qualities]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L776-L792)

### 2.1.4. `qualityUpperBound` Implementation

The `qualityUpperBound` method[^directstepi-qualityupperbound] estimates the quality (exchange rate) of this step without actually executing it. This is used during [strand sorting and filtering](README.md#62-qualityupperbound) to rank strands by their potential quality.

The implementation is the same for both `DirectIPaymentStep` and `DirectIOfferCrossingStep`. The difference in behavior comes from the `quality()` method that `qualityUpperBound` calls:
- **Payment**: Respects trust line quality settings (can apply discounts/premiums)
- **Offer Crossing**: Always returns `QUALITY_ONE` (ignores quality settings for backward compatibility)

#### 2.1.4.1. `qualityUpperBound` Pseudo-Code

```python
def qualityUpperBound(v, prevStepDir):
    # Determine debt direction for this step
    dir = debtDirection(v, FORWARD)

    # Calculate qualities based on whether source is redeeming or issuing
    if dir == REDEEMS:
        srcQOut, dstQIn = qualitiesSrcRedeems(v)
    else:
        srcQOut, dstQIn = qualitiesSrcIssues(v, prevStepDir)

    iss = Issue(currency_, src_)
    # Quality is the composite of srcQOut and dstQIn
    # Rate = srcQOut / dstQIn (because Input * dstQIn / srcQOut = Output)
    # getRate(offerOut, offerIn) returns offerIn/offerOut, so we pass parameters reversed:
    # getRate(dstQIn, srcQOut) returns srcQOut/dstQIn
    q = getRate(iss(dstQIn), iss(srcQOut))
    return (q, dir)
```

[^directstepi-qualityupperbound]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L804-L819)

### 2.1.5. `check` Implementation

The base class `DirectStepI` provides[^directstepi-check] common validation that runs before the derived class checks. Called during strand construction. The base class validation runs first, then calls the derived class's specific `check()` implementation.

**Common failure conditions**:

- `temBAD_PATH`[^check-bad-path-null][^check-bad-path-same]: Source or destination account is null, or source equals destination
- `terNO_ACCOUNT`[^check-no-account]: Source account does not exist
- `terNO_LINE`[^check-freeze]: Currency is frozen via global freeze, trust line freeze, or LP token freeze (unless this is both first and last step - pure issue/redeem)
- `tecINTERNAL`[^check-tecinternal]: AMM account exists but AMM ledger entry does not exist (should not happen)
- `terNO_LINE`[^check-no-ripple-line]: The previous step is also a DirectStep (two consecutive DirectSteps). `checkNoRipple` reads the two trust lines that connect to the intermediate account (the source of the current step): one from the source of the previous step and one to the destination of the current step. This check fails if either trust line does not exist.
- `terNO_RIPPLE`[^check-no-ripple]: The previous step is also a DirectStep and the intermediate account has NoRipple set on **both** trust lines. If either trust line has NoRipple cleared on the intermediate account's side, rippling is allowed and this check passes.
- `temBAD_PATH_LOOP`[^check-bad-path-loop-book][^check-bad-path-loop-seen]: Path loop detected (the asset already appears in the `seenDirectAssets` loop-detection set for this direction)

[^check-bad-path-null]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L829)
[^check-bad-path-same]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L835)
[^check-no-account]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L842)
[^check-freeze]: [`StepChecks.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/detail/StepChecks.h#L26) (also lines 34, 40, 55) - called from [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L848)
[^check-tecinternal]: [`StepChecks.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/detail/StepChecks.h#L51) - called from [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L848)
[^check-no-ripple-line]: [`StepChecks.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/detail/StepChecks.h#L78) - called from [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L859)
[^check-no-ripple]: [`StepChecks.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/detail/StepChecks.h#L86) - called from [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L859)
[^check-bad-path-loop-book]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L885)
[^check-bad-path-loop-seen]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L894)
[^directstepi-check]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L823-L899)

## 2.2. DirectIPaymentStep (Payment-Specific Implementation)

### 2.2.1. `quality` Implementation

The `quality` method[^directipaymentstep-quality] returns the quality adjustment that applies to this trust line transfer. For payments, this reads the trust line's QualityIn or QualityOut settings, which allow accounts to specify discounts or premiums for sending or receiving tokens. If the source and destination are the same account, quality is always `QUALITY_ONE` (1:1). Otherwise, the method reads the trust line and returns the appropriate quality setting based on the quality direction parameter.

#### 2.2.1.1. `quality` Pseudo-Code

```python
def quality(sb, qDir):
    # Same account, no quality adjustment
    if src_ == dst_:
        return QUALITY_ONE

    # Read trust line
    sle = sb.read(keylet.line(dst_, src_, currency_))
    if not sle:
        return QUALITY_ONE

    # Determine which quality field to read based on direction and account ordering
    if qDir == IN:
        # Destination quality in
        field = sfLowQualityIn if dst_ < src_ else sfHighQualityIn
    else:
        # Source quality out
        field = sfLowQualityOut if src_ < dst_ else sfHighQualityOut

    if not sle.isFieldPresent(field):
        return QUALITY_ONE
    q = sle[field]
    if not q:
        return QUALITY_ONE
    return q
```

[^directipaymentstep-quality]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L342-L380)

### 2.2.2. `maxFlow` Implementation

The `maxFlow`[^directipaymentstep-maxflow] method determines the maximum amount of IOU tokens that can flow from source to destination on this trust line, which `revImp` uses to constrain its calculations. The method also returns the debt direction, indicating whether the source is issuing or redeeming IOUs.

[^directipaymentstep-maxflow]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L391-L394) (calls [`maxPaymentFlow`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L478-L488))

In DirectIPaymentStep, `maxFlow` returns:

- If source is the issuer and destination is holder: the remaining limit the holder has for that currency (how many more tokens the holder can receive)
- If source is the holder and destination is issuer: the holder's balance in the token (how many tokens the holder can redeem)

#### 2.2.2.1. `maxFlow` Pseudo-Code

```python
# The out parameter is unused
def maxFlow(sb, out):
    # Get the balance of src_ on the trustline
    srcOwed = accountHolds(sb, src_, currency_, dst_, FreezeHandling.IgnoreFreeze)

    if srcOwed > 0:
        # Holder is sending to issuer, they can only send as much as was issued to them
        return (srcOwed, REDEEMS)

    # Issuer is sending to holder
    # creditLimit2 returns the trust line limit the holder (dst_) set for the issuer (src_)
    # Adding srcOwed (which is negative or zero) gives us remaining capacity
    return (creditLimit2(sb, dst_, src_, currency_) + srcOwed, ISSUES)
```

### 2.2.3. `check` Implementation

The `check` method[^directipaymentstep-check] validates payment-specific constraints:

- `terNO_LINE`[^check-payment-no-line]: Trust line does not exist between source and destination
- `terNO_AUTH`[^check-payment-no-auth]: Source account has `lsfRequireAuth` flag, but trust line is not authorized and has zero balance
- `terNO_RIPPLE`[^check-payment-no-ripple]: Previous step was a BookStep and the trust line has NoRipple flag set
- `tecPATH_DRY`[^check-payment-path-dry]: Trust line has no liquidity available (at limit)

[^check-payment-no-line]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L427)
[^check-payment-no-auth]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L437)
[^check-payment-no-ripple]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L445)
[^check-payment-path-dry]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L458)
[^directipaymentstep-check]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L418-L463)

## 2.3. DirectIOfferCrossingStep (Offer Crossing-Specific Implementation)

### 2.3.1. `quality` Implementation

This function[^directioffercrossingstep-quality] always returns `QUALITY_ONE`, ignoring trust line quality settings.

[^directioffercrossingstep-quality]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L383-L388)

### 2.3.2. `maxFlow` Implementation

If this is the last step, `maxFlow`[^directioffercrossingstep-maxflow] returns the desired amount, allowing unlimited flow to destination. This effectively ignores trust line limits during offer crossing.
[^directioffercrossingstep-maxflow]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L397-L415)

The system presumes that when someone creates an offer, they intend to fill as much of that offer as possible, even if it exceeds the trust line limit. Trust line limits exist to protect against unwanted incoming tokens, but creating an offer is an explicit expression of intent to receive those tokens, so the limit restriction is waived.

Otherwise, it returns using the same logic as DirectIPaymentStep.

### 2.3.3. `check` Implementation

The function[^directioffercrossingstep-check] has no additional failure conditions beyond the [base class checks](#215-check-implementation). Offer crossing does not require a pre-existing trust line for `takerPays`, but placing the offer would fail if `takerGets` trust line did not exist for the holder.
[^directioffercrossingstep-check]: [`DirectStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/DirectStep.cpp#L466-L472)

# 3. XRPEndpointStep

The pseudocode below refers to a shared set of step state:

```python
# XRPEndpointStep state (moves XRP to/from the strand at acc_)
acc_:    AccountID   # the real account XRP enters from or exits to
isLast_: bool        # True if this is the destination endpoint, False if the source
# xrpAccount() is a zero sentinel marking where XRP enters/leaves the flow, not a real account.
# XRP transfers are 1:1, so quality is always QUALITY_ONE.
```

## 3.1. `revImp` Implementation

[^xrpendpointstep-revimp]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L261-L279)

`XRPEndpointStep::revImp`[^xrpendpointstep-revimp] transfers XRP from or to the user account. Uses `xrpAccount()` (a zero-valued sentinel account) as a bookkeeping marker to indicate where XRP enters or exits the payment flow - it's not a real account. If this is the last step (destination), accepts all requested XRP. If this is the first step (source), limits transfer to available balance after accounting for reserves. For offer crossing, when this is the first step (XRP source) and the trust line or MPToken for the delivered asset does not yet exist, reduces owner count by 1 when calculating available balance.

### 3.1.1. `revImp` Pseudo-Code

```python
def revImp(sb, out):
    # xrpLiquid calculates available XRP balance accounting for reserves
    # For payment steps: no reserve reduction
    # For offer crossing steps: if this is the first step and the trust line/MPToken
    #   for the delivered asset doesn't yet exist, reduces owner count by 1 before calculating reserves
    balance = xrpLiquid(sb)

    # If this is the last step (destination), accept all XRP requested
    # If this is the first step (source), limited by available balance
    result = out if isLast_ else min(balance, out)

    sender, receiver = (xrpAccount(), acc_) if isLast_ else (acc_, xrpAccount())
    if accountSend(sb, sender, receiver, result) != tesSUCCESS:
        return (0, 0)
    cache_ = result
    return (result, result)  # 1:1, so in == out
```

## 3.2. `fwdImp` Implementation

[^xrpendpointstep-fwdimp]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L283-L302)

`XRPEndpointStep::fwdImp`[^xrpendpointstep-fwdimp] has the same logic as `revImp` but starts from the input amount.

### 3.2.1. `fwdImp` Pseudo-Code

```python
def fwdImp(sb, in):
    assert cache_ is set
    balance = xrpLiquid(sb)
    # If destination, accept all input
    # If source, limited by balance
    result = in if isLast_ else min(balance, in)
    sender, receiver = (xrpAccount(), acc_) if isLast_ else (acc_, xrpAccount())
    if accountSend(sb, sender, receiver, result) != tesSUCCESS:
        return (0, 0)
    cache_ = result
    return (result, result)
```

## 3.3. `qualityUpperBound` Implementation

[^xrpendpointstep-qualityupperbound]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L254-L257)

`XRPEndpointStep::qualityUpperBound`[^xrpendpointstep-qualityupperbound] always returns quality of `QUALITY_ONE` (1:1) and "issues" direction. 

### 3.3.1. `qualityUpperBound` Pseudo-Code

```python
def qualityUpperBound(prevStepDir):
    return (QUALITY_ONE, ISSUES)                     # XRP transfers are always 1:1
```

## 3.4. `check` Implementation

[^xrpendpointstep-check]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L338-L375)

`XRPEndpointStep::check`[^xrpendpointstep-check] verifies the account exists, ensures the step is at a strand boundary (first or last position), checks for an account/trust-line freeze, and detects path loops. Can return one of the following errors:

- `temBAD_PATH`[^xrpcheck-bad-path-null][^xrpcheck-bad-path-boundary]: Account is null or step is not first or last in strand (XRP endpoints must be at strand boundaries)
- `terNO_ACCOUNT`[^xrpcheck-no-account]: Account does not exist
- `terNO_LINE`[^xrpcheck-freeze]: a freeze applies to the endpoint account (global freeze, directional/deep trust-line freeze, or LP-token freeze), as determined by `checkFreeze`
- `temBAD_PATH_LOOP`[^xrpcheck-bad-path-loop]: Path loop detected (the XRP asset already appears in the `seenDirectAssets` loop-detection set for this direction)

[^xrpcheck-bad-path-null]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L343)
[^xrpcheck-bad-path-boundary]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L357)
[^xrpcheck-no-account]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L352)
[^xrpcheck-freeze]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L362) - calls [`StepChecks.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/detail/StepChecks.h#L26)
[^xrpcheck-bad-path-loop]: [`XRPEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/XRPEndpointStep.cpp#L371)

# 4. MPTEndpointStep

MPTEndpointStep handles Multi-Purpose Token (MPT) transfers at the source or destination of a payment strand. Like XRPEndpointStep, it can only appear at strand boundaries (first or last step)[^mpt-boundary-check]. Unlike XRP which has no issuer, each MPTEndpointStep must have exactly one endpoint be the issuer[^mpt-issuer-check] - either the source or destination must be the issuer account. This means individual steps either issue (issuer -> holder) or redeem (holder -> issuer)[^mpt-debt-direction], which determines whether transfer fees apply and affects authorization checks.

[^mpt-boundary-check]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L886-L890)
[^mpt-issuer-check]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L893-L897)
[^mpt-debt-direction]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L459-L464)

For holder-to-holder payments, the payment path contains two MPTEndpointSteps: the first step redeems from holder to issuer, and the last step issues from issuer to holder. Transfer fees are applied during the issuing step (when `qualitiesSrcIssues` sees that the previous step redeemed). See [Direct MPT Payment Execution](../payments/README.md#4-payment-execution-paths) for detailed holder-to-holder transfer mechanics.

For non-issuer accounts, authorization is validated via `requireAuth`[^mpt-require-auth], which checks the [`lsfMPTRequireAuth`](../mpts/README.md#2121-flags) flag on the issuance and the [`lsfMPTAuthorized`](../mpts/README.md#2221-flags) flag on the holder's MPToken. When the issuance has a [DomainID](../mpts/README.md#11-domainid-and-authorization) set, credentials are validated against that domain. DEX operations validate the [`lsfMPTCanTrade`](../mpts/README.md#2121-flags) flag via [`canTrade`](../mpts/README.md#361-cantrade); issuance- and holder-level lock flags are checked separately by `isGlobalFrozen` and `isIndividualFrozen`.

[^mpt-require-auth]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L342-L349)

The pseudocode below refers to a shared set of step state:

```python
# MPTEndpointStep state (moves MPT `mptIssue_` between src_ and dst_; one end is the issuer)
src_, dst_: AccountID    # the two ends; exactly one is the MPT issuer
mptIssue_:  MPTIssue     # the MPT being moved
prevStep_:  Step or None # upstream step, used to propagate quality and debt direction
cache_:     RevResult    # reverse-pass result, reused by fwdImp
# MPTs have no trust-line quality, so dstQIn is always QUALITY_ONE (quality reduces to srcQOut).
```

## 4.1. `revImp` Implementation

[^mptendpointstep-revimp]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L469-L549)

`MPTEndpointStep::revImp`[^mptendpointstep-revimp] starts by determining the maximum flow by calling `accountFunds`, which returns different values depending on the source: for holders, it reads the `MPTAmount` from the holder's MPToken entry; for the issuer, it calculates available capacity as `MaximumAmount - OutstandingAmount`. The `accountFunds` call uses `FreezeHandling::IgnoreFreeze` and `AuthHandling::IgnoreAuth` flags to skip freeze and authorization checks, since those validations are performed separately in the `check()` method during path validation.

When a holder is the source (redeeming), flow is limited by the holder's balance. When the issuer is the source, the behavior depends on whether this is a direct payment or part of a multi-step path. For direct issuer-to-holder payments (no previous step, such as when the issuer sends MPT directly to a holder without path finding or currency conversion), flow is limited by available capacity (`MaximumAmount - OutstandingAmount`). For multi-step paths where the issuer is the last step, the implementation cannot determine the actual capacity limit yet because the previous step's direction (issuing or redeeming) affects the net change to `OutstandingAmount`. In holder-to-holder transfers, the previous step redeems (decreasing `OutstandingAmount`) before this step issues (increasing it), so the net effect may be zero or even a decrease despite this step appearing to "issue" tokens. Therefore, when there's a previous step, `maxPaymentFlow` returns `MaximumAmount` to avoid prematurely limiting flow, letting the previous step determine the actual constraint.

**Example**: Consider a holder-to-holder transfer of 100 MPT where `MaximumAmount = 1000` and `OutstandingAmount = 950`. The path contains two steps:
1. Step 1 (first): Holder A -> Issuer (redeems 100, will decrease `OutstandingAmount` to 850)
2. Step 2 (last): Issuer -> Holder B (issues 100, will increase `OutstandingAmount` to 950)

During the reverse pass, Step 2 is evaluated first. At this point, `accountFunds` for the issuer returns `MaximumAmount - OutstandingAmount = 50`, suggesting only 50 MPT can flow. However, this is incorrect because Step 1 hasn't been evaluated yet. Once Step 1 redeems 100 MPT, there will be sufficient capacity. By returning `MaximumAmount = 1000` from `maxPaymentFlow`, Step 2 doesn't artificially limit the flow, and Step 1's balance (which limits to 100) becomes the actual constraint.

Transfer fees apply when the issuer issues and the previous step redeems.

For offer crossing, `checkCreateMPT` may create the MPToken entry for the offer owner if needed. When this is the last step (TakerPays side), the offer owner is receiving MPT and needs an MPToken entry to hold the balance. The check verifies authorization and creates the MPToken if it doesn't exist, allowing the offer to be crossed even if the owner hasn't previously authorized the MPT. For payments, `checkCreateMPT` always returns success since MPToken creation is handled separately through the [MPTokenAuthorize transaction](../mpts/README.md#34-mptokenauthorize-transaction).

### 4.1.1. `revImp` Pseudo-Code

```python
def revImp(sb, out):
    # Determine max flow and direction
    maxSrcToDst, debtDir = maxPaymentFlow(sb)
    srcQOut, dstQIn = qualities(sb, debtDir, REVERSE)
    if maxSrcToDst <= 0:
        cache_ = zero
        return (0, 0)

    # Offer crossing: create MPToken if needed
    if checkCreateMPT(sb, debtDir) != tesSUCCESS:
        return (0, 0)

    # dstQIn is always QUALITY_ONE for MPT
    srcToDst = out
    if srcToDst <= maxSrcToDst:
        # Non-limiting
        in = roundUp(srcToDst * srcQOut / QUALITY_ONE)
        move(sb, src_ -> dst_, srcToDst)
        cache_ = (in, srcToDst, srcToDst, debtDir)
        return (in, out)

    # Limiting
    in = roundUp(maxSrcToDst * srcQOut / QUALITY_ONE)
    move(sb, src_ -> dst_, maxSrcToDst)
    cache_ = (in, maxSrcToDst, maxSrcToDst, debtDir)
    return (in, maxSrcToDst)
```

**Helper: `maxPaymentFlow()`**

```python
def maxPaymentFlow(sb):
    maxFlow = accountFunds(src_, mptIssue_)

    # Holder to issuer (redeeming)
    if src_ != issuer:
        return (maxFlow, REDEEMS)

    # Issuer to holder (issuing)
    sle = sb.getIssuance(mptIssue_)
    if sle:
        # Direct from issuer (only step)
        if not prevStep_:
            return (maxFlow, ISSUES)

        # Last step with issuer as source
        # Allow temporary overflow - previous step limits flow
        maxAmount = sle[sfMaximumAmount] or kMaxMpTokenAmount
        return (maxAmount, ISSUES)

    return (0, ISSUES)
```

## 4.2. `fwdImp` Implementation

[^mptendpointstep-fwdimp]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L599-L681)

`MPTEndpointStep::fwdImp`[^mptendpointstep-fwdimp] processes the forward pass using cached state from reverse to ensure consistency.

### 4.2.1. `fwdImp` Pseudo-Code

```python
def fwdImp(sb, in):
    assert cache_ is set                                 # revImp always runs first
    maxSrcToDst, debtDir = maxPaymentFlow(sb)
    srcQOut, _ = qualities(sb, debtDir, FORWARD)
    if maxSrcToDst <= 0:
        cache_ = zero; return (0, 0)
    if checkCreateMPT(sb, debtDir) != tesSUCCESS:
        return (0, 0)

    srcToDst = roundDown(in * QUALITY_ONE / srcQOut)
    if srcToDst <= maxSrcToDst:
        out = srcToDst
    else:                                                # liquidity-limited
        in = roundUp(maxSrcToDst * srcQOut / QUALITY_ONE)
        out = maxSrcToDst
    setCacheLimiting(in, srcToDst, out, debtDir)         # never exceed what revImp promised
    # Transfer MPT using cached amount from reverse pass
    move(sb, src_ -> dst_, cache_.srcToDst)
    return (cache_.in, cache_.out)
```

## 4.3. `qualityUpperBound` Implementation

[^mptendpointstep-qualityupperbound]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L801-L817)

`MPTEndpointStep::qualityUpperBound`[^mptendpointstep-qualityupperbound] calculates quality as `srcQOut / dstQIn`, representing the exchange rate (how much input is needed per unit of output). Since MPTs have no trust-line-level quality adjustments, `dstQIn` is always `QUALITY_ONE`, so the quality simplifies to `srcQOut`.

The `srcQOut` value depends on the debt direction:

- **Source redeems (holder -> issuer)**: `srcQOut = max(QUALITY_ONE, prevStep.lineQualityIn())` to prevent quality improvement along the path
- **Source issues (issuer -> holder)**:
  - If previous step redeems: `srcQOut = transferRate(mptID)` (an issuance-level transfer fee set via the MPToken issuance's `TransferFee` field)
  - Otherwise: `srcQOut = QUALITY_ONE`

Transfer fees only apply when tokens move between holders through the issuer as an intermediary. See [Holder-to-Holder Transfer](../payments/README.md#4-payment-execution-paths) for the two-step transfer fee mechanism.

### 4.3.1. `qualityUpperBound` Pseudo-Code

```python
def qualityUpperBound(sb, prevStepDir):
    dir = debtDirection(sb, FORWARD)

    if dir == REDEEMS:
        srcQOut, dstQIn = qualitiesSrcRedeems(sb)
    else:
        srcQOut, dstQIn = qualitiesSrcIssues(sb, prevStepDir)

    # dstQIn always QUALITY_ONE for MPT
    # getRate(offerOut, offerIn) returns offerIn/offerOut
    # For direct step, rate = srcQOut/dstQIn, so pass dstQIn first
    quality = getRate(QUALITY_ONE, srcQOut)
    return (quality, dir)
```

```python
def qualitiesSrcRedeems():
    if not prevStep_:
        return (QUALITY_ONE, QUALITY_ONE)

    prevStepQIn = prevStep_.lineQualityIn()
    srcQOut = QUALITY_ONE
    if prevStepQIn > srcQOut:
        srcQOut = prevStepQIn
    return (srcQOut, QUALITY_ONE)
```

```python
def qualitiesSrcIssues(prevDebtDir):
    # Charge transfer rate when issuing and previous step redeems
    if prevDebtDir == REDEEMS:
        srcQOut = transferRate(mptID)
    else:
        srcQOut = QUALITY_ONE

    # Unlike trustline, MPT doesn't have line quality field
    return (srcQOut, QUALITY_ONE)
```

## 4.4. `check` Implementation

[^mptendpointstep-check]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L821-L900)

**Common checks[^mptendpointstep-check] (both payment and offer crossing):**
[^mptendpointstep-check-bad-path-null]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L824-L828)
[^mptendpointstep-check-bad-path-same]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L830-L834)
[^mptendpointstep-check-no-account]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L836-L841)
[^mptendpointstep-check-bad-path-loop]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L877-L883)
[^mptendpointstep-check-bad-path-boundary]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L885-L890)
[^mptendpointstep-check-bad-path-issuer]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L892-L897)
[^mpt-common-freeze]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L849-L856)
[^mpt-common-seenbookouts]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L858-L873)

- `temBAD_PATH`:
  - Source/destination is null, or source equals destination[^mptendpointstep-check-bad-path-null][^mptendpointstep-check-bad-path-same]
  - Step not at strand boundary (MPT endpoints must be first or last)[^mptendpointstep-check-bad-path-boundary]
  - Invalid src/dst (exactly one must be issuer)[^mptendpointstep-check-bad-path-issuer]
- `terNO_ACCOUNT`[^mptendpointstep-check-no-account]: Source account does not exist
- `terLOCKED`[^mpt-common-freeze]: the MPT is frozen at the endpoint account. A global freeze applies on the first step, or the holder's MPToken is individually frozen. This check is skipped when the step is both first and last (a pure issuer/holder issue or redeem).
- `temBAD_PATH_LOOP`[^mptendpointstep-check-bad-path-loop]: MPT issue appears multiple times in the same role (source or destination) within a strand. Since MPTEndpointStep can only appear at strand boundaries, each strand tracks seen assets separately for source-side (`seenDirectAssets[0]`) and destination-side (`seenDirectAssets[1]`). If the same MPT issue appears twice on the same side, it would count the same issuer's liquidity pool multiple times, leading to incorrect flow calculations. `temBAD_PATH_LOOP` is also returned if this MPT issue was already produced as a book step's output earlier in the strand, tracked in `seenBookOuts`[^mpt-common-seenbookouts]

**Payment-specific (`MPTEndpointPaymentStep`):**[^mptendpointstep-check-payment]

[^mptendpointstep-check-payment]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L331-L393)
[^mptendpointstep-check-payment-requireauth]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L340-L350)
[^mptendpointstep-check-payment-frozen]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L361-L366)
[^mptendpointstep-check-payment-dex]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L374-L376)
[^mptendpointstep-check-payment-dry]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L383-L390)

- Authorization[^mptendpointstep-check-payment-requireauth]: `requireAuth` for non-issuer src and dst
  - `tecOBJECT_NOT_FOUND`: Issuance doesn't exist
  - `tecINTERNAL`: Recursion depth exceeded
  - `tecNO_AUTH`: No MPToken or not authorized
  - `tecEXPIRED`: From credential domain validation
- Direct holder-to-holder[^mptendpointstep-check-payment-frozen]: `isFrozen` and `canTransfer` checks
  - `tecLOCKED`: MPToken is frozen
  - `tecOBJECT_NOT_FOUND`: Issuance doesn't exist
  - `tecNO_AUTH`: Can't transfer between holders
- DEX crossing[^mptendpointstep-check-payment-dex]: [`canTrade`](../mpts/README.md#361-cantrade) when crossing order books
  - `tecOBJECT_NOT_FOUND`: Issuance doesn't exist
  - `tecNO_PERMISSION`: `lsfMPTCanTrade` flag not set
- `tecPATH_DRY`[^mptendpointstep-check-payment-dry]: Source balance is zero or negative; for an issuer source, no remaining issuance capacity (first step only)

**Offer crossing-specific (`MPTEndpointOfferCrossingStep`):**[^mptendpointstep-check-offer]

[^mptendpointstep-check-offer]: [`MPTEndpointStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/MPTEndpointStep.cpp#L396-L402)

- No additional MPT checks: offer crossing doesn't require a pre-existing MPToken, so it relies on the standard `MPTEndpointStep` checks (above) and returns `tesSUCCESS`.

# 5. BookStep

BookStep is a templated class instantiated in multiple variants covering combinations of XRP, IOUs, and MPT:

- `BookStepII` (IOU -> IOU)
- `BookStepIX` (IOU -> XRP)
- `BookStepIM` (IOU -> MPT)
- `BookStepMM` (MPT -> MPT)
- `BookStepMI` (MPT -> IOU)
- `BookStepMX` (MPT -> XRP)
- `BookStepXI` (XRP -> IOU)
- `BookStepXM` (XRP -> MPT)

BookStep converts currencies by consuming liquidity from two sources: the Central Limit Order Book (CLOB) and Automated Market Maker (AMM) pools. The CLOB consists of standing offers placed by users, stored as ledger entries sorted by quality. AMM pools provide synthetic offers calculated on-demand from pool balances. BookStep compares CLOB and AMM and consumes liquidity from whichever source offers the best deal. The quality comparison is performed by the [`tipOfferQuality()` helper function](#552-tipofferquality-helper-function), which retrieves the best CLOB quality via the BookTip iterator and the best AMM quality via synthetic offer generation, then returns whichever is better.

The reverse pass (`revImp`) works backwards from the desired output amount, iterating through offers in quality order until it accumulates enough liquidity to satisfy the request. The forward pass (`fwdImp`) works forwards from the available input, consuming the same offers in a reset sandbox while ensuring it never delivers more than the reverse pass calculated.

**Implementation Architecture:**

In the actual implementation, both `revImp`[^revimp-eachoffer] and `fwdImp`[^fwdimp-eachoffer] create lambda functions called `eachOffer` that capture local variables (like `remainingOut`/`remainingIn`, `savedIns`, `savedOuts`, and `result`) by reference. Each method calls `forEachOffer()`, passing its lambda as the callback parameter.

[^revimp-eachoffer]: `revImp` lambda callback creation: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1019-L1064)
[^fwdimp-eachoffer]: `fwdImp` lambda callback creation: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1128-L1225)

Inside `forEachOffer`, the first CLOB offer is fetched from the order book[^foreachoffer-traverse], then a single AMM synthetic offer is generated if the AMM's spot price can beat the CLOB quality[^foreachoffer-getammoffer] and processed first via the `eachOffer` callback. CLOB offers are then iterated one by one. For each offer (whether AMM or CLOB), `forEachOffer` calculates the offer amounts accounting for transfer fees and owner funding, then invokes the `eachOffer` callback with the calculated amounts.

[^foreachoffer-traverse]: Order book directory traversal via `FlowOfferStream`: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L705)
[^foreachoffer-getammoffer]: AMM offer generation for each CLOB offer: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L831)

The `eachOffer` callback first checks if `remainingOut <= 0` and returns `false` if the payment is already satisfied. Otherwise, it compares `stpAmt.out` (the amount that will flow through this step after accounting for transfer fees and owner funding)[^stpamt-calculation] against `remainingOut`. If the offer's output is less than or equal to the remaining output needed (`stpAmt.out <= remainingOut`), the offer is fully consumable - it consumes the offer and records its contribution to the step's total input and output, then returns `true` to continue (even if this offer satisfied the full amount, the callback returns `true` to ensure the offer is properly consumed; the next iteration will detect the zero remaining and stop).
If the offer's output exceeds the remaining output needed (`stpAmt.out > remainingOut`), it calls `limitStepOut()` to adjust the amounts down to exactly what's needed, then partially consumes the offer and records the adjusted contribution, returning whether to continue based on whether the offer is fully consumed. When the callback returns `false`, `forEachOffer` stops iterating and returns control back to `revImp` or `fwdImp`.

[^stpamt-calculation]: Step amount calculation with transfer fees and owner funding: [`BookStep.cpp:769-792`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L769-L792)

This callback architecture separates concerns: `forEachOffer` handles iteration mechanics - traversing the order book directory, generating AMM synthetic offers via `getAMMOffer`, comparing CLOB vs AMM quality to select the better source, performing asset-specific authorization checks, and verifying offer funding. The callback handles consumption decisions - comparing offer amounts against remaining needs and calling `limitStepOut` (adjusts offer amounts down when output exceeds what's needed) or `limitStepIn` (adjusts offer amounts down when input is limited) to scale offers appropriately. After the callback determines the amounts, it invokes `consumeOffer`, which performs the actual ledger updates: for CLOB offers, it adjusts the offer's TakerPays/TakerGets amounts and marks fully consumed offers for deletion; for AMM offers, it updates the AMM pool balances by transferring assets to/from the AMM account using `accountSend`.

The following is a high-level overview of the entire process:

```python
def revImp(sb, out):
    result = 0
    remainingOut = out

    def eachOffer(offer, stpAmt):
        if stpAmt.out <= remainingOut:
            result += stpAmt.in
            remainingOut -= stpAmt.out
            consumeOffer(offer, stpAmt)
            return CONTINUE
        else:
            adjusted = limitStepOut(offer, remainingOut)
            result += adjusted.in
            consumeOffer(offer, adjusted)
            return offer.fullyConsumed() ? CONTINUE : STOP

    forEachOffer(sb, eachOffer)
    return result


def forEachOffer(sb, callback):
    offers = OfferStream(sb, book)

    # Try AMM once against best CLOB quality, then iterate CLOB offers
    ammOffer = getAMMOffer(sb, offers.tipQuality())
    if ammOffer:
        stpAmt = applyTransferFees(ammOffer)
        if callback(ammOffer, stpAmt) == STOP:
            return

    for offer in offers:
        if shouldRemove(offer):
            offers.remove(offer)
            continue

        ownerFunds = getOwnerFunds(sb, offer)
        ofrAmt = adjustForFunds(offer, ownerFunds)
        stpAmt = applyTransferFees(ofrAmt)

        if callback(offer, stpAmt) == STOP:
            break


# Real signature: consumeOffer(sb, offer, ofrAmt, stepAmt, ownerGives);
# the overview passes a single stpAmt for brevity.
def consumeOffer(sb, offer, stpAmt):
    # No isCLOB() branch: CLOB vs AMM is polymorphic, handled inside
    # the offer.send() and offer.consume() methods.
    offer.checkInvariant(stpAmt)   # AMM pool-product invariant (fixAMMOverflowOffer)
    offer.send(...)                # two transfers: owner receives input, pays output (with fees)
    offer.consume(sb, stpAmt)      # CLOB: reduce/delete offer; AMM: update pool balances
```

**Offer Iteration and Funding:**

The order book traversal in `forEachOffer` is implemented through `FlowOfferStream`[^flowofferstream-class], which extends `TOfferStreamBase`[^tofferstreambase-class] with permanent offer removal tracking. `TOfferStreamBase` contains a `BookTip` member and delegates directory iteration to it.[^offerstream-implementation] `BookTip` provides directory traversal sorted by quality, while `TOfferStreamBase` adds expiration checks, funding verification, and frozen asset handling. When processing each offer, `TOfferStreamBase::step()` calls `tip_.step()` to advance through the directory, then retrieves the offer entry via `tip_.entry()` and quality via `tip_.quality()`.

[^flowofferstream-class]: [`OfferStream.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/OfferStream.h#L129-L149)
[^tofferstreambase-class]: [`OfferStream.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/OfferStream.h#L15-L109)
[^offerstream-implementation]: `TOfferStreamBase` implementation with `BookTip` delegation: [`OfferStream.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/OfferStream.cpp#L205-L232)

`FlowOfferStream` adds the `permToRemove` collection, which tracks offers that should be permanently removed even if the strand is not applied. This is used by `forEachOffer` to track self-crossed offers and other invalid offers that need removal regardless of transaction outcome.

During iteration, `TOfferStreamBase::step()` determines which offers to remove from the order book.[^offerstream-step] It marks offers for permanent removal when the ledger entry is missing, the offer has expired, either amount is zero, the asset is deep frozen, the offer owner's account is no longer in the offer's domain (for domain-restricted offers), or the owner has zero balance. For unfunded offers and tiny offers with reduced quality under `fixReducedOffersV1`, it distinguishes between offers that were already in that state versus offers that became that way during the current transaction by comparing balances in the current view against the pristine `cancelView`.
Only offers that were already unfunded or tiny are permanently removed from the ledger; offers that became unfunded or tiny during execution are simply skipped for this transaction but remain in the order book.

[^offerstream-step]: Offer removal logic in `TOfferStreamBase::step()`: [`OfferStream.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/OfferStream.cpp#L214-L327)

For each valid offer that passes these checks, `TOfferStreamBase` verifies whether the offer owner has sufficient balance to cover their takerGets obligation.[^offerstream-funds-helper]
For IOU issuers, this returns the full requested amount directly since they can issue unlimited amounts.
For MPT issuers, this returns the available minting capacity (`MaximumAmount - OutstandingAmount`) via `issuerFundsToSelfIssue`; if this amount is zero or negative (MaximumAmount reached), the offer is treated as unfunded and removed.
For non-issuers, `TOfferStreamBase` queries the owner's actual available balance through `accountHolds` and stores it in the `ownerFunds_` member.

[^offerstream-funds-helper]: Owner funds verification via `accountFundsHelper`: [`OfferStream.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/OfferStream.cpp#L104-L130)

If the owner's balance is less than takerGets, the offer is underfunded but can still partially fill. The `TOfferStreamBase::ownerFunds()` method returns this actual available balance, which `forEachOffer` uses to proportionally adjust the offer amounts via `Quality::ceilOutStrict()`.
This adjustment maintains the offer's exchange rate while scaling down the size to match available funds, enabling offers to partially fill when the owner has insufficient balance. 
The adjustment rounds down under the `fixReducedOffersV1` amendment to prevent order book blocking where dust amounts could prevent offer consumption.

**Asset-Specific Authorization:**

BookStep handles three asset types, each with different authorization requirements for offer owners:[^bookstep-auth]

- **XRP**: No authorization checks required. Any account can hold XRP.
- **Tokens (IOUs)**: Checked via `requireAuth`, which verifies the offer owner either has a trust line to the token issuer, or the issuer doesn't require authorization (`lsfRequireAuth` flag). If the issuer requires auth and the owner lacks the appropriate auth flag on their trust line, the offer is marked for removal.
- **MPTs**: Require `requireAuth` (which checks the [`lsfMPTAuthorized`](../mpts/README.md#2221-flags) flag on the holder's MPToken) and `checkMPTDEX`, the MPT DEX permission check (it runs [`canTrade`](../mpts/README.md#361-cantrade) on both book assets and, where the owner is not the issuer, [`canTransfer`](../mpts/README.md#362-cantransfer) for the [`lsfMPTCanTransfer`](../mpts/README.md#2121-flags) flag). When crossing offers where the owner will receive an MPT, if the owner doesn't have an MPToken entry, BookStep automatically creates it via `checkCreateMPT`.

[^bookstep-auth]: Asset authorization checks and MPToken creation in BookStep: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L729-L760)

**Pseudocode notes:**

Like other step implementations, BookStep maintains state between reverse and forward pass executions. Because BookStep iterates through potentially many offers in the order book and integrates with AMM liquidity, it maintains significantly more state than simpler step types.

Throughout this section, member variables suffixed with `_` (e.g., `book_`, `ownerPaysTransferFee_`) represent object state that persists across method calls. 

```python
# BookStep state (converts book_.in -> book_.out via CLOB offers and an AMM pool)
book_:                  Book                 # asset pair (+ optional domain) for this order book
strandSrc_, strandDst_: AccountID            # the strand's overall source and destination
prevStep_:              Step or None
ownerPaysTransferFee_:  bool                 # True for offer crossing (owner pays the output fee)
ammLiquidity_:          AMMLiquidity or None # synthetic AMM offers for this pair, if a pool exists
ammContext:             AMMContext           # shared single/multi-path flag + 30-iteration AMM cap
cache_:                 RevResult            # reverse-pass (in, out), reused by fwdImp
inactive_:              bool                 # set when too many offers consumed (DoS guard)

# BookOfferCrossingStep-only state
defaultPath_:           bool                 # True for the direct path and False for auto-bridging
qualityThreshold_:      Quality              # minimum quality accepted from a resting offer
```

## 5.1. `revImp` Implementation

[^bookstep-revimp]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L998-L1105)

`BookStep::revImp`[^bookstep-revimp] executes the reverse pass, determining how much input is required to produce a desired output amount by consuming offers from the order book and AMM liquidity. Working backwards from the desired output, the method iterates through offers sorted by quality (best rates first), calculating how much of each offer can be consumed after accounting for transfer fees. It integrates AMM liquidity when available and competitive with CLOB offers, accumulating the total input required as it processes each liquidity source. The iteration continues until the desired output is satisfied or all available liquidity is exhausted, tracking which offers should be removed (expired, unfunded, or fully consumed) along the way.

Given a requested output amount, this function:
1. Iterates through offers in the order book (sorted by quality)
2. For each offer, calculates how much can be consumed accounting for transfer fees
3. Integrates AMM liquidity when available and better than CLOB offers
4. Tracks which offers should be removed (expired, unfunded, or fully consumed)
5. Accumulates the total input required and output delivered
6. Stops when the desired output is satisfied or the book is exhausted

### 5.1.1. `revImp` Pseudo-Code

```python
def revImp(sb, out):
    remainingOut = out
    result = (in=0, out=0)

    # Callback invoked by forEachOffer for each offer in the book
    def eachOffer(offer, offerAmount, stepAmount, ownerGives, rateIn, rateOut):
        # Stop if we've satisfied the desired output
        if remainingOut <= 0:
            return STOP
        # Check if we can consume entire offer with remaining output needed
        if stepAmount.out <= remainingOut:
            # Consume entire offer
            result += stepAmount
            remainingOut -= stepAmount.out
            consumeOffer(sb, offer, offerAmount, stepAmount, ownerGives)
            # Return true to continue even if payment satisfied, because we need to consume the offer
            return CONTINUE
        # Consume partial offer - limit by remaining output needed
        limitStepOut(offer, offerAmount, stepAmount, ownerGives, rateIn, rateOut, remainingOut)
        result.in += stepAmount.in; result.out = out; remainingOut = 0
        consumeOffer(sb, offer, offerAmount, stepAmount, ownerGives)
        return CONTINUE if offer.fully_consumed() else STOP

    # Determine debt direction for transfer fee calculation
    prevDir = prevStep_.debtDirection(sb, REVERSE) if prevStep_ else ISSUES
    # Iterate through offers (both CLOB and AMM)
    removed, offersUsed = forEachOffer(sb, prevDir, eachOffer)
    offersToRemove.union(removed)
    # Check if we consumed too many offers. Prevents DoS attacks where malicious actors
    # create many tiny unfunded offers to force expensive iteration
    if offersUsed >= kMaxOffersToConsume:
        # Use the liquidity we found but mark this path as "dry" so it won't be tried
        # again in future iterations of this payment
        inactive_ = True
    cache_ = result
    return result
```


## 5.2. `fwdImp` Implementation

[^bookstep-fwdimp]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L1109-L1267)

`BookStep::fwdImp`[^bookstep-fwdimp] executes the forward pass, consuming available input to produce output. Unlike `revImp`, it asserts that the cache is already set from the reverse pass. A key aspect is handling rounding differences: the forward pass may produce more output than the reverse pass while consuming the same input (or less). When this occurs, `fwdImp` computes the input required to produce the cached output (from the reverse pass). If that input equals the input consumed in the forward pass, it uses the cached output; otherwise it keeps the forward calculation result.

### 5.2.1. `fwdImp` Pseudo-Code

```python
def fwdImp(sb, in):
    assert cache_ is set
    remainingIn = in
    result = (in=0, out=0)
    savedInputs, savedOutputs = [], []
    lastOutput = None

    # Callback invoked by forEachOffer for each offer in the book
    def eachOffer(offer, offerAmount, stepAmount, ownerGives, rateIn, rateOut):
        # We've consumed all available input, nothing left to spend
        if remainingIn <= 0:
            return STOP
        # Check if we can consume entire offer with remaining input
        if stepAmount.in <= remainingIn:
            # Consume entire offer
            savedInputs.append(stepAmount.in)
            lastOutput = savedOutputs.append(stepAmount.out)
            result = (sum(savedInputs), sum(savedOutputs))
            keepGoing = CONTINUE
        else:
            # Consume partial offer - limit by remaining input
            limitStepIn(offer, offerAmount, stepAmount, ownerGives, rateIn, rateOut, remainingIn)
            savedInputs.append(remainingIn)
            lastOutput = savedOutputs.append(stepAmount.out)
            result.out = sum(savedOutputs); result.in = in
            keepGoing = STOP

        # Forward/Reverse coherence check
        # The step produced more output in the forward pass than the reverse pass
        # while consuming the same input (or less). If we compute the input required
        # to produce the cached output and the input equals the input consumed in the
        # forward step, then consume the input provided in the forward step and produce
        # the output requested from the reverse step.
        if result.out > cache_.out and result.in <= cache_.in:
            savedOutputs.remove(lastOutput)
            neededOut = cache_.out - sum(savedOutputs)
            revOffer, revStep, revOwner = offerAmount, stepAmount, ownerGives
            limitStepOut(offer, revOffer, revStep, revOwner, rateIn, rateOut, neededOut)
            if revStep.in == remainingIn:
                # Perfect match - use cached output
                result = (in, cache_.out)
                savedInputs, savedOutputs = [in], [cache_.out]
                offerAmount, ownerGives = revOffer, revOwner
                stepAmount = (in=remainingIn, out=neededOut)
            else:
                # Can't match exactly - this is (likely) a problem case
                # and will be caught with later checks. Restore last output.
                savedOutputs.append(lastOutput)

        remainingIn = in - result.in
        # Consume the offer
        consumeOffer(sb, offer, offerAmount, stepAmount, ownerGives)
        return keepGoing or offer.fully_consumed()

    # Determine debt direction for transfer fee calculation
    prevDir = prevStep_.debtDirection(sb, FORWARD) if prevStep_ else ISSUES
    # Iterate through offers (both CLOB and AMM)
    removed, offersUsed = forEachOffer(sb, prevDir, eachOffer)
    offersToRemove.union(removed)
    # Check if we consumed too many offers. Prevents DoS attacks
    if offersUsed >= kMaxOffersToConsume:
        inactive_ = True
    if remainingIn == 0:
        # Due to normalization, remainingIn can be zero without result.in == in
        # Force result.in == in for this case
        result.in = in
    cache_ = result
    return result
```

## 5.3. `forEachOffer`

[^bookstep-foreachoffer]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L686-L853)

`BookStep::forEachOffer`[^bookstep-foreachoffer] iterates through available liquidity sources (order book offers and AMM offers) in quality order, calling a provided callback for each valid offer until the payment requirements are satisfied.

Given a callback function and the previous step's debt direction, this function:
1. Calculates transfer rates: input rate applies when the previous step redeems; output rate applies when ownerPaysTransferFee_ is set (true for offer crossings, checks and bridges)
2. Iterates through order book offers sorted by quality (best prices first)
3. Generates AMM offers when available and compares their quality with CLOB offers (skips AMM if domain filtering is active)
4. Before calculating amounts, detects resting offers that would directly self-cross during default-path offer crossing and marks them for permanent deletion
5. For each remaining offer, performs asset-specific validation:
   - Creates MPToken for the offer owner if input is MPT (if it does not already exist) 
   - Validates authorization via `requireAuth` (Tokens check trust lines, MPTs check holder authorization)
   - For MPTs, validates DEX/transfer permission via `checkMPTDEX` (`canTrade` plus `canTransfer`)
   - Checks offer funding and calculates transfer fees
   - For MPT input on the first step, limits input amount to prevent issuer overflow. `MaximumAmount - OutstandingAmount` should never become negative.
6. Calls the callback with calculated amounts (offer amount, step amount, owner gives, transfer rates)
7. Tracks which offers should be removed (self-crossable, expired, unfunded, invalid, or fully consumed)
8. Stops when the callback returns false or the book is exhausted

The callback determines how much liquidity to consume from each offer and whether to continue processing more offers.

During default-path offer crossing, `BookOfferCrossingStep` checks whether the current resting offer belongs to the account on both ends of the strand and meets the quality threshold set by the new offer. When both conditions hold, the offer is treated as a direct self-cross and marked for removal. `BookPaymentStep` and auto-bridged paths do not perform this check.[^bookstep-self-cross]

The check runs before normal offer processing, so the engine does not calculate a fill or invoke the callback. It deletes the entire resting offer without moving assets or reducing the new offer. Deleting the offer through `BookTip` initially changes only the working view. To make the removal permanent, the self-cross branch adds the offer key to `FlowOfferStream::permToRemove`. The Flow result reports that key to `OfferCreate`, which deletes the offer from both the crossing sandbox and the cleanup sandbox.[^booktip-permanent-removal]

If self-cross removal occurs before any non-self offer has been attempted, the engine also clears `tipQuality`. The deleted offer no longer restricts iteration to its quality level, so the next offer can be considered even when its quality differs.

[^bookstep-self-cross]: Self-cross detection and permanent removal in `BookOfferCrossingStep::limitSelfCrossQuality`: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L399-L454)

[^booktip-step]: `TOfferStreamBase` stores a `BookTip` and delegates advancement to it: [`OfferStream.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/OfferStream.h#L49-L59), [`OfferStream.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/OfferStream.cpp#L190-L206). `BookTip::step()` deletes the current offer before finding the next one: [`BookTip.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookTip.cpp#L20-L67)

[^booktip-permanent-removal]: `OfferCreate` deletes each offer reported in `removableOffers` from both sandboxes: [`OfferCreate.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/transactors/dex/OfferCreate.cpp#L459-L465)

### 5.3.1. `forEachOffer` Pseudo-Code

```python
# This predicate captures the condition in BookOfferCrossingStep::limitSelfCrossQuality.
# BookPaymentStep always returns False for this check.
def isSelfCross(offer):
    if stepType is not BookOfferCrossingStep:
        return False

    return (
        defaultPath_
        and offer.quality() >= qualityThreshold_
        and strandSrc_ == offer.owner()
        and strandDst_ == offer.owner()
    )


def forEachOffer(sb, prevStepDir, callback):
    # Calculate transfer rates for this book step.
    # These rates determine fees charged when crossing offers.

    # Input transfer rate: only charged when previous step redeems (pulls from issuer)
    rateIn  = transferRate(book_.in)  if redeems(prevStepDir) else QUALITY_ONE
    # Output transfer rate: charged when ownerPaysTransferFee_ is set.
    # Always charge the transfer fee, even if the owner is the issuer.
    rateOut = transferRate(book_.out) if ownerPaysTransferFee_ else QUALITY_ONE
    offers = FlowOfferStream(sb, book_)
    tipQuality = None
    offerAttempted = False

    # Lambda to execute each offer (both CLOB and AMM)
    def execOffer(offer):
        if tipQuality is None:
            tipQuality = offer.quality()
        elif tipQuality != offer.quality():
            # Stop when quality changes - only process same quality offers per iteration
            return STOP
        # Delete a resting self-offer instead of executing a trade against it.
        # No amount comparison is involved.
        if isSelfCross(offer):
            offers.permRemove(offer)
            if not offerAttempted:
                # No liquidity has been attempted, so iteration may move to a
                # different quality after deleting the current offer.
                tipQuality = None
            return CONTINUE  # offers.step() deletes the current offer and advances

        owner = offer.owner()
        assetIn, assetOut = offer.assetIn(), offer.assetOut()
        # Create MPToken for the offer's owner. No need to check for the reserve since the
        # offer is removed if it is consumed, so the owner count remains the same.
        if assetIn is MPT and checkCreateMPT(sb, assetIn, owner) != tesSUCCESS:
            return CONTINUE
        # Make sure offer owner has authorization to own Assets from issuer if IOU. An account
        # can always own XRP or their own Assets. If MPT then MPTDEX should be allowed.
        if not requireAuth(assetIn, owner) or not checkMPTDEX(sb, owner):
            # Offer owner not authorized to hold IOU/MPT from issuer.
            # Remove this offer even if no crossing occurs.
            offers.permRemove(offer)
            if not offerAttempted:
                # Change quality only if no previous offers were tried.
                tipQuality = None
            return CONTINUE  # causes offers.step() to delete the offer
        if not checkQualityThreshold(offer.quality()):
            return STOP

        # adjustRates handles CLOB offers (rates unchanged) and AMM offers (returns
        # (offerInRate, QUALITY_ONE) because AMM never pays a transfer fee on output).
        rIn, rOut = offer.adjustRates(getOfrInRate(prevStep_, owner, rateIn),
                                      getOfrOutRate(prevStep_, owner, strandDst_, rateOut))
        ofrAmt = offer.amount()
        # stpAmt represents what the step consumes/produces (includes transfer fees)
        stpAmt = (in = roundUp(ofrAmt.in * rIn / QUALITY_ONE), out = ofrAmt.out)
        # What the offer owner actually gives (output amount adjusted for transfer fee)
        ownerGives = roundDown(ofrAmt.out * rOut / QUALITY_ONE)

        # For funded offers (owner is issuer), use ownerGives directly. Otherwise check the
        # owner's actual balance, and limit the offer if the owner doesn't have enough funds.
        funds = ownerGives if offer.isFunded() else offers.ownerFunds()
        if funds < ownerGives:
            ownerGives = funds
            stpAmt.out = roundDown(ownerGives * QUALITY_ONE / rOut)
            # Rounding down here prevents order book blocking (an amendment-guarded change).
            ofrAmt = offer.limitOut(ofrAmt, stpAmt.out)
            stpAmt.in = roundUp(ofrAmt.in * rIn / QUALITY_ONE)

        # Limit offer's input if MPT, BookStep is the first step (an issuer is making a
        # cross-currency payment), and this offer is not owned by the issuer. Otherwise,
        # OutstandingAmount may overflow.
        if assetIn is MPT and not prevStep_ and owner != assetIn.issuer:
            available = accountFunds(sb, assetIn.issuer, assetIn)
            if stpAmt.in > available:
                limitStepIn(offer, ofrAmt, stpAmt, ownerGives, rIn, rOut, available)

        offerAttempted = True
        return callback(offer, ofrAmt, stpAmt, ownerGives, rIn, rOut)

    # At any payment engine iteration, an AMM offer can only be consumed once.
    def tryAMM(clobQuality):
        # AMM doesn't support permissioned DEX yet
        if book_.domain:
            return CONTINUE
        ammOffer = getAMMOffer(sb, clobQuality)
        return STOP if (ammOffer and execOffer(ammOffer) == STOP) else CONTINUE

    # Main loop: try AMM first, then iterate through CLOB offers
    if offers.step():
        if tryAMM(offers.tip().quality()) == CONTINUE:
            while execOffer(offers.tip()) == CONTINUE and offers.step():
                pass
    else:
        # Might have AMM offer if there are no CLOB offers.
        tryAMM(None)
    return (offers.permToRemove(), offersConsumed)
```

### 5.3.2. Transfer Rate Helper Functions

The `getOfrInRate` and `getOfrOutRate` functions[^bookstep-transfer-rate-helpers] calculate transfer rates to be applied when consuming offers in `forEachOffer`. These functions exist in both `BookPaymentStep` and `BookOfferCrossingStep` but have different implementations:

[^bookstep-transfer-rate-helpers]: BookPaymentStep: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L326-L336), BookOfferCrossingStep: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L486-L508)

**BookPaymentStep**: Always returns the input rates unchanged. No transfer fee waiving occurs during payment execution - all fees are applied as specified.

**BookOfferCrossingStep**: Conditionally waives transfer fees to avoid charging someone to send assets to themselves. Specifically:
- Waives input transfer fee if the offer owner is the previous step's source account (only applies when the previous step is a DirectStep or MPTEndpointStep)
- Waives output transfer fee if the previous step is a BookStep and the offer owner is the strand's destination account

These functions are called in `forEachOffer` when processing each offer, and the results are passed to `offer.adjustRates()` which handles AMM-specific fee logic (AMM offers never pay transfer fees on output).

#### 5.3.2.1. `getOfrInRate` Pseudo-Code

```python
# BookPaymentStep version (always returns input unchanged)
def getOfrInRate(prevStep, offerOwner, transferRate):
    return transferRate

# BookOfferCrossingStep version (conditionally waives fee)
def getOfrInRate(prevStep, offerOwner, transferRate):
    # Waive fee if offer owner is previous step's account
    # (prevents charging owner to send to themselves)
    if prevStep and prevStep.account() == offerOwner:
        return QUALITY_ONE
    return transferRate
```

#### 5.3.2.2. `getOfrOutRate` Pseudo-Code

```python
# BookPaymentStep version (always returns input unchanged)
def getOfrOutRate(prevStep, offerOwner, strandDst, transferRate):
    return transferRate

# BookOfferCrossingStep version (conditionally waives fee)
def getOfrOutRate(prevStep, offerOwner, strandDst, transferRate):
    # Waive fee if prevStep is a BookStep and offer owner is strand destination
    # (prevents charging owner to receive their own funds)
    if prevStep and prevStep.bookStepBook() and strandDst == offerOwner:
        return QUALITY_ONE
    return transferRate
```

## 5.4. AMM Integration

AMMs integrate with the XRP Ledger's payment and trading infrastructure through the Payment engine, providing liquidity alongside traditional order book (CLOB) offers. This section explains the AMM-specific mechanisms that enable this integration. For general information about AMMs, see [AMM Documentation](../amms/README.md). For AMM helper functions and formulas, see [AMM Helpers Documentation](../amms/helpers.md).

### 5.4.1. Integration with BookStep

AMMs are integrated into the payment execution system through `BookStep`, which handles both order book and AMM liquidity for a given asset pair. When a `BookStep` is created for an asset pair that has an AMM pool, a single `AMMLiquidity` instance is constructed and attached to that `BookStep`. This instance maintains pool information (account ID, asset pair, trading fee, initial balances) and generates synthetic AMM offers on-demand.

The `BookStep::forEachOffer()` function (see [section 5.3](#53-foreachoffer)) iterates through available liquidity in quality order:

1. Fetch the first valid CLOB offer via `FlowOfferStream`
2. Generate a single AMM offer via `AMMLiquidity::getOffer()` using the CLOB quality as threshold. If the AMM's spot price can beat it, the AMM offer is consumed first via the callback.
3. Iterate through CLOB offers one by one, consuming each via the callback
4. Stop when the callback returns false or the book is exhausted

This interleaving allows AMMs and order books to compete based on quality. When multiple strands are providing liquidity, AMM does not immediately provide full liquidity. As AMM's quality is changing depending on the size, sometimes it can provide better quality the the best CLOB alternative, but sometimes it cannot.    

Unlike order book offers which exist as ledger objects, AMM offers are computed on-demand from the current pool state. 

### 5.4.2. Spot Price Quality and CLOB Comparison

AMM offers have a **spot price quality**[^amm-spot-price-quality] determined by the current pool composition:

```
SpotPriceQuality = AssetIn / AssetOut
```

[^amm-spot-price-quality]: [`Quality.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/protocol/Quality.cpp#L17-L19)

Before generating an AMM offer, `AMMLiquidity::getOffer()`[^amm-spot-price-comparison] compares the AMM's spot price quality with the best available CLOB quality. Since the spot price represents the AMM's best possible quality before any swap, and any actual offer will have worse quality due to slippage, checking the spot price provides an early filter: if the spot price quality already cannot beat the CLOB, then any sized AMM offer will also fail to compete.

[^amm-spot-price-comparison]: [`AMMLiquidity.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/AMMLiquidity.cpp#L184-L190)

- If `SpotPriceQuality <= CLOBQuality` or the two qualities are within 1e-7 relative distance, the AMM returns `std::nullopt` (cannot compete)
- Otherwise, an AMM offer is generated

### 5.4.3. Offer Generation Strategies

Once `AMMLiquidity::getOffer()`[^amm-getoffer-sizing] determines the AMM can compete (spot price quality beats CLOB), it must decide how much liquidity to offer. The sizing strategy depends on whether the payment uses multiple paths or a single path.

[^amm-getoffer-sizing]: [`AMMLiquidity.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/AMMLiquidity.cpp#L192-L222)

The sizing strategy branches based on `ammContext_.multiPath()`:

- **Multi-path mode** (payment has multiple parallel strands competing): Uses Fibonacci sequence sizing to generate small, incrementally larger offers across iterations
- **Single-path mode** (only one strand): Sizes offers aggressively to match or exceed CLOB quality via `changeSpotPriceQuality()`

#### 5.4.3.1. `getOffer` Pseudo-Code

The pseudocode assumes all relevant amendments are enabled (fixAMMv1_1, fixAMMv1_2, fixAMMOverflowOffer). The actual implementation includes fallback behavior for pre-amendment ledgers. See source code for complete amendment-conditional logic.

```python
def getOffer(view, clobQuality):
    if ammContext.maxItersReached():
        return None                                         # AMM already used in 30 iterations
    balances = fetchBalances(view)
    if not balances:
        return None
    if clobQuality and spot_price(balances) not_better_than clobQuality:
        return None                                         # AMM's best case can't beat CLOB

    if ammContext.multiPath():
        # Small base (0.025% of pool) scaled by the Fibonacci number for this iteration.
        out = swapAssetIn(initialBalances, 0.00025 * initialBalances.in, fee) * fib[iteration]
        amounts = (in = swapAssetOut(balances, out, fee), out = out)
        if clobQuality and Quality(amounts) < clobQuality:
            return None
    elif clobQuality:
        # Single path: size so the post-swap spot price exactly matches CLOB (else max offer).
        amounts = changeSpotPriceQuality(balances, clobQuality, fee) or maxOffer_if_better(clobQuality)
        if not amounts:
            return None
    else:
        amounts = maxOffer(balances)                        # no CLOB competition: offer the most
    return AMMOffer(amounts, balances, quality = amounts.in / amounts.out)
```

#### 5.4.3.2. Multi-Path Mode: Fibonacci Sequence Sizing

Multi-path payments have multiple strands competing to provide liquidity. The Payment engine uses an iterative algorithm: in each iteration, it evaluates all available strands, ranks them by quality (exchange rate), and executes the highest-quality strand. This process repeats until the payment is fulfilled, ensuring the sender gets the best overall rate by consuming the most efficient liquidity sources first. `BookStep::forEachOffer()` alternates between CLOB and AMM offers, selecting whichever provides better quality.

When ammContext is `multiPath`, AMM offers are sized using a Fibonacci sequence. In the first iteration, as tracked in `ammContext`, AMM creates a synthetic offer that is 0.025% of the pool size.[^multipath-iter0] Each subsequent offer from a single AMM pool gets multiplied by the next number in the Fibonacci sequence.

[^multipath-iter0]: The source returns this base (iteration-0) offer directly; the Fibonacci scaling and the `swapAssetOut` input recompute shown in the pseudo-code apply only from iteration 1 onward (`kFib` is indexed by `curIters() - 1`): [`AMMLiquidity.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/AMMLiquidity.cpp#L63-L100)

As the desired offer output increases, the required input progressively increases, degrading the exchange rate for the taker. See [Swap Formulas](../amms/helpers.md#313-slippage-and-quality-degradation) for slippage examples.

By offering small amounts (starting at 0.025% of pool), the AMM maintains competitive quality, allowing the AMM strand to win a portion of the multi-path allocation.

This allows the AMM to provide **some** liquidity at good quality, rather than providing **no** liquidity because a large synthetic AMM offer would have poor quality.

If there are no CLOB offers that can compete, AMM pool will quickly offer a large portion of liquidity due to Fibonacci sizing, allowing a reasonable limit of iterations as 30.

**Algorithm**:[^amm-generate-fib-seq]

[^amm-generate-fib-seq]: [`AMMLiquidity.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/AMMLiquidity.cpp#L65-L100)

Each `AMMLiquidity` instance tracks the pool's initial balances (when the BookStep was created) and fetches current balances before each offer generation. This allows the algorithm to maintain consistent base sizing while accounting for liquidity changes as offers are consumed.

1. Calculate base offer: `0.00025 * initialBalances.in` (0.025% of the pool's input asset balance at BookStep creation)
2. Apply trading fee to base: `baseOut = `[`swapAssetIn`](../amms/helpers.md#311-swapassetin)`(initialBalances, baseOfferIn, fee)` - calculates output for the base-sized input using the initial pool state
3. Scale by Fibonacci: `scaledOut = baseOut * fib[iteration]` where `fib = [1, 1, 2, 3, 5, 8, 13, ...]` - grows the offer size exponentially across iterations
4. Recalculate input with current state: `scaledIn = `[`swapAssetOut`](../amms/helpers.md#312-swapassetout)`(currentBalances, scaledOut, fee)` - determines required input for the scaled output using current pool balances (which may have changed as previous offers were consumed)

#### 5.4.3.3. Single-Path Mode: CLOB-Matching Sizing

Single-path payments have no alternative liquidity sources to balance. The AMM should provide as much liquidity as competitive with the best CLOB offer. After consuming this offer, the AMM's spot price equals CLOB, preventing further AMM consumption.

**Case 1: No CLOB Quality Available**[^amm-no-clob]

[^amm-no-clob]: [`AMMLiquidity.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/AMMLiquidity.cpp#L202-L209)

If there is no CLOB offer to compete against (`clobQuality == std::nullopt`):

- Return `maxOffer()` directly (see "Max Offer Sizing" below)
- The offer will be limited by BookStep based on payment constraints (SendMax, DeliverMin, etc.)
- This provides maximum AMM liquidity when there's no order book competition

**Case 2: CLOB Quality Available**[^amm-change-spot-price]

[^amm-change-spot-price]: [`AMMLiquidity.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/AMMLiquidity.cpp#L210-L215)

The [`changeSpotPriceQuality()`](../amms/helpers.md#314-changespotpricequality) function calculates an offer size such that after the swap, the AMM's new spot price quality equals the CLOB quality:

**Fallback to Max Offer:**

If [`changeSpotPriceQuality()`](../amms/helpers.md#314-changespotpricequality) fails (e.g., cannot match quality without exceeding pool size):

- (With [`fixAMMv1_2`](https://xrpl.org/resources/known-amendments#fixammv1_2) amendment): Falls back to `maxOffer()` if the max offer's quality is better than CLOB quality

**Max Offer Sizing:**

The `maxOffer()` function provides the largest possible offer from the pool:

- (With [`fixAMMOverflowOffer`](https://xrpl.org/resources/known-amendments#fixammoverflowoffer) amendment): Returns 99% of pool balance as output (`balances.out * 0.99`), calculates required input via [`swapAssetOut()`](../amms/helpers.md#312-swapassetout)

## 5.5. `qualityUpperBound` Implementation

The `qualityUpperBound` method[^bookstep-qualityupperbound] estimates the best quality the BookStep can achieve. It retrieves the best offer quality (from either CLOB or AMM) and adjusts it for transfer fees.

[^bookstep-qualityupperbound]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L572-L586)

This quality estimate is used for [strand sorting](README.md#6-iterative-strands-evaluation-strandsflow) during multi-path payment evaluation and, when a `limitQuality` constraint is specified, for calculating the maximum output amount that maintains the required quality threshold in single-path payments.

To estimate quality, the system first identifies the best available exchange rate from the order book, comparing both CLOB offers and AMM pool pricing to find whichever offers the better rate (`tipOfferQuality`).
However, this raw exchange rate doesn't account for transfer fees that token issuers may charge when tokens move between accounts. The system must adjust the quality estimate to reflect these fees (`adjustQualityWithFees`).
For payments, input transfer fees are applied when the previous step redeems, and output transfer fees are applied for CLOB offers (but waived for AMM offers).
For offer crossing, the adjustment strategy differs: CLOB offers and multi-path AMM offers apply no transfer fees to maintain `qualityUpperBound` as a true upper bound. Single-path AMM offers apply input transfer fees (when the previous step redeems) but waive output transfer fees.

### 5.5.1. `qualityUpperBound` Method

The `qualityUpperBound` method[^bookstep-qualityupperbound-impl] for BookStep retrieves the best available offer quality from the order book (either the top CLOB offer or AMM spot price), determines the debt direction for this step, and adjusts the quality to account for transfer fees. For AMM offers, transfer fees are waived on output.

[^bookstep-qualityupperbound-impl]: [`BookStep.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookStep.cpp#L572-L586)

The method calls the polymorphic `adjustQualityWithFees()` function, which has different implementations for [payment steps](#553-adjustqualitywithfees---bookpaymentstep-implementation) versus [offer crossing steps](#554-adjustqualitywithfees---bookoffercrossingstep-implementation).

#### 5.5.1.1. `qualityUpperBound` Pseudo-Code

```python
def qualityUpperBound(prevStepDir):
    # Determine debt direction for this step
    dir = debtDirection(FORWARD)

    # Get the best available offer quality (AMM or CLOB), if any
    res = tipOfferQuality()
    if not res:
        return (None, dir)

    # Determine if transfer fee should be waived
    waiveFee = (res.type == AMM)

    # Adjust quality with transfer fees (polymorphic call)
    return (adjustQualityWithFees(res.quality, prevStepDir, waiveFee), dir)
```

### 5.5.2. `tipOfferQuality` Helper Function

The `tipOfferQuality` method returns the best quality available at the tip of the order book along with its source type (AMM or CLOB). The method calls the `tip()` helper function, which compares both CLOB and AMM offer qualities and returns whichever provides a better exchange rate.

For CLOB offers, the quality is retrieved using the BookTip iterator class[^booktip-class]. BookTip traverses offers in an order book from the highest quality to lowest quality by navigating the directory structure where qualities are encoded in the 8 rightmost bytes of directory index keys. The BookTip constructor[^booktip-constructor] takes a `Book` parameter, which contains the asset pair (in/out currencies and issuers) and an optional `domain` field. When a domain is specified, BookTip looks up the domain-specific order book directory[^booktip-domain] computed as `hash(BOOK_NAMESPACE, asset_in, asset_out, domainID)`, ensuring only domain offers are traversed. The `step()` method[^booktip-step] searches the directory for the first offer page, extracts the quality from the index[^booktip-extract-quality], and retrieves the corresponding offer ledger entry. The `quality()` method[^booktip-quality-method] then returns this extracted quality value.

[^booktip-class]: [`BookTip.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/BookTip.h#L15-L61)

[^booktip-constructor]: [`BookTip.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookTip.cpp#L15-L18)

[^booktip-domain]: [`Indexes.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/protocol/Indexes.cpp#L107-L109)

[^booktip-step]: [`BookTip.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookTip.cpp#L20-L67)

[^booktip-extract-quality]: [`BookTip.cpp`](https://github.com/XRPLF/rippled/blob/3.2.0/src/libxrpl/tx/paths/BookTip.cpp#L49)

[^booktip-quality-method]: [`BookTip.h`](https://github.com/XRPLF/rippled/blob/3.2.0/include/xrpl/tx/paths/BookTip.h#L43-L47)

For AMM offers, a synthetic offer is generated based on the current pool state (see [section 5.4.3](#543-offer-generation-strategies) for the complete algorithm). The AMM offer generation takes the CLOB quality as a threshold parameter - if the AMM's spot price quality cannot beat this threshold, no AMM offer is generated.

After retrieving both qualities, the `tip()` function compares them and returns whichever is better (higher quality = better exchange rate). The returned `OfferType` enum indicates whether the best offer comes from an AMM or CLOB, which determines whether transfer fees should be waived (AMM offers never pay transfer fees on output, while CLOB offers may be subject to transfer fees).

### 5.5.3. `adjustQualityWithFees` - BookPaymentStep Implementation

For payments, apply transfer fees conditionally based on debt direction and offer type:

#### 5.5.3.1. `adjustQualityWithFees` Pseudo-Code

```python
def adjustQualityWithFees(ofrQ, prevStepDir, waiveFee):
    def rate(asset):
        if isXRP(asset):
            return QUALITY_ONE
        if asset.isToken():  # IOU
            if asset.getIssuer() == strandDst_:
                return QUALITY_ONE
            return transferRate(asset.getIssuer())
        # MPT: parity only when this is the strand's delivered asset AND its issuer is the destination
        if asset == strandDeliver_ and asset.getIssuer() == strandDst_:
            return QUALITY_ONE
        return transferRate(asset.getMptID())

    # Input transfer rate: only charged when previous step redeems
    trIn = rate(book_.in) if redeems(prevStepDir) else QUALITY_ONE

    # Output transfer rate: charged when owner pays and fee not waived
    if ownerPaysTransferFee_ and not waiveFee:
        trOut = rate(book_.out)
    else:
        trOut = QUALITY_ONE

    # Compose transfer fees with offer quality
    return composedQuality(getRate(trOut, trIn), ofrQ)
```

### 5.5.4. `adjustQualityWithFees` - BookOfferCrossingStep Implementation

The quality adjustment strategy:

- **CLOB offers** and **Multi-path AMM offers**: Return quality unchanged. CLOB offers have fixed quality at creation time. Multi-path AMM offers use [Fibonacci-sized offers](#5432-multi-path-mode-fibonacci-sequence-sizing) that maintain competitive quality across iterations.

- **Single-path AMM offers**: Require special handling. These use [CLOB-matching sizing](#5433-single-path-mode-clob-matching-sizing) which can consume large portions of the pool, causing significant quality degradation via the slippage curve. Must compose the transfer rate on the input currency (charged when the previous step redeems) with the offer quality to get an accurate upper bound estimate.

The quality adjustment logic:

#### 5.5.4.1. `adjustQualityWithFees` Pseudo-Code

```python
def adjustQualityWithFees(ofrQ, prevStepDir, offerType):
    # Offer crossing doesn't charge transfer fee when offer owner is strand destination.
    # For qualityUpperBound to be an upper bound, we assume no fee is charged.

    # If fixAMMv1_1 amendment not enabled, no adjustment
    if not fixAMMv1_1_enabled:
        return ofrQ

    # CLOB offers and multi-path AMM offers: no adjustment needed
    # Multi-path AMM offers already account for quality degradation through Fibonacci sizing
    if offerType == CLOB or (ammLiquidity_ and ammLiquidity_.multiPath()):
        return ofrQ

    # Single-path AMM offers: must factor in transfer-in rate
    # Single-path AMM offer quality is not constant (changes as pool depletes)
    def rate(asset):
        if isXRP(asset):
            return QUALITY_ONE
        if asset.isToken():  # IOU
            if asset.getIssuer() == strandDst_:
                return QUALITY_ONE
            return transferRate(asset.getIssuer())
        # MPT: parity only when this is the strand's delivered asset AND its issuer is the destination
        if asset == strandDeliver_ and asset.getIssuer() == strandDst_:
            return QUALITY_ONE
        return transferRate(asset.getMptID())

    trIn = rate(book_.in) if redeems(prevStepDir) else QUALITY_ONE
    trOut = QUALITY_ONE  # AMM doesn't pay transfer fee on output

    return composedQuality(getRate(trOut, trIn), ofrQ)
```

## 5.6. `check` Implementation

The `check` method validates the BookStep configuration and ensures no loops exist in the payment path.

**Common checks:**
- `temBAD_PATH`: Book input and output assets are the same
- `temBAD_PATH`: Currency is inconsistent with issuer (XRP with non-XRP issuer or vice versa)
- `temBAD_PATH_LOOP`: Book output asset already seen in another BookStep or DirectStep (prevents offer unfunding loops)
- `tecNO_ISSUER`: Input or output issuer account does not exist
- For MPT assets (checked unconditionally, regardless of the previous step): `tecOBJECT_NOT_FOUND` if an MPT issuance does not exist, and a DEX permission check via [`canTrade`](../mpts/README.md#361-cantrade) on both the book input and output assets

**Asset-specific checks (when previous step is DirectStep or MPTEndpointStep):**

For **Tokens**:
- `terNO_LINE`: Trust line does not exist between previous step source and book input issuer
- `terNO_RIPPLE`: Trust line has NoRipple flag set

For **MPTs**: the previous-step-gated block performs no additional check (it returns no error for MPT).
