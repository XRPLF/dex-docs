# Motivation

XRP Ledger (XRPL) is a mature and widely used blockchain that distinguishes itself by supporting native multi-currency payments and integrating a decentralized exchange (DEX) into its execution layer. Unlike other platforms where such functionalities are implemented through smart contracts, XRPL's payment engine utilizes built-in mechanisms like trust-lines, order book, and AMMs to facilitate complex multi-currency payment.

However, despite its sophistication and real-world usage, XRPL's payment engine lacks a formal specification. The logic underpinning key features-such as paths, trust-lines, and rippling is embedded deeply in the `xrpld` codebase.

This specification aims to provide a single point of reference that can be used as canonical documentation that explains how the payment engine works at a conceptual and architectural level.

The main goal of this work is to set the foundation for formally verifying the C++ implementation of the Payment Engine in `xrpld`.

## Target Audience

The target audience for this specification are:

- **Researchers** who delve into XRP Ledger protocol in order to provide a formal verification of the components of the payment system.
- New and experienced `xrpld` **contributors** who need a detailed explanation of the Payment Engine and surrounding payment components. As a source of truth on the payments functionality in XRP Ledger, new developers can use it to onboard to the `xrpld` codebase, and existing developers can use it to build new features and identify and fix bugs in the system.
- **Users** and **integrators** of XRP Ledger who want a comprehensive understanding of how to utilize the Payment Engine and how to send payments most effectively.
- Anyone interested in implementing their own XRP Ledger client and validator, in another language, or with different design choices.


## Existing Resources

XRP Ledger has existing resources that can be used to learn more about the Payment Engine. This includes:

- [XRP Ledger Developer Resources: Documentation](https://xrpl.org/docs) 
- [XRP Ledger Standards](https://github.com/XRPLF/XRPL-Standards)
- [Payment Engine System Design Overview](https://ripple.com/reports/Payment-Engine-System-Design.pdf)

This specification augments existing resources and provides additional value to interested readers in the following ways:

- It is a single, comprehensive source of truth for the entire payment system.
- While https://xrpl.org/docs provides an excellent reference to different transaction types and ledger entries, it does not go into detailed explanation of transaction validation and application logic. This code contains important invariants and is crucial to document to identify potential bugs and unexpected side-effects.
- While there is a good overview of the Payment Engine, this specification explains detailed logic and gives context to interaction between the Payment Engine and the pathfinding system.
- Pathfinding and path selection are complex topics and this specification provides explanation of both functional requirements and system design.