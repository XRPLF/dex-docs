Credentials

PermissionedDomains

MPT:

#7077 - ReferenceHolding field
#7040 - Implicitly authorize Vault, LoanBroker, and AMM pseudo-accounts (make sure we covered everything)
#6712 - Disallow MPTClearRequireAuth if is set
#7037 - fixCleanup3_2_0 (only enables amendment, but leaving here for reference)
#7117 - Fix non-canonical MPT amount
#5285 - Add MPT support to DEX (integrated before, there was a couple of commits since our last edits)


Offers:

#7087 - Quality on hybrid offers
#5935 - Fix directory limit
#5285 - Also covered offers, as well as MPTs
#7362 - Invariant check, but the underlying code was already covered
#6716 - fixCleanup3_1_3 (empty AdditionalBooks check in hybrid offer invariant; was fixSecurity3_1_3)





Payments:

#7117 - Fix non-canonical MPT amount
#5978 - DepositPreauth retired
#6568 - Decouple reserve from fee in delegate payment
#6056 - Retire deletable accounts
#7040 - Pseudo-account implicit-authorization

Trust lines:

#5989 - Retire fixTRustLinesToSelf
#6045 - Retire DisallowIncoming amendment
#5270 - Lending Protocol (and subsequent PRs) 
#5935 - Remove directory limit size


Path finding:

#6571 - Enable clang-tidy readability-identifier-naming check
#6226 - Modularise HashRouter, Conditions, and OrderBookDB


Flow:

#5285 - Add MPT support to DEX (mostly covered already, missing latest edits)
#6571 - Enable clang-tidy readability-identifier-naming check
#6580 - Rename transactor files/classes to tx name
#6676 - Called rename non-functional uses of ripple(d) to xrpl(d), but hides deeper refactoring
#7040 - Add unconditional canTrade on both book assets and stricter MPT transfer-rate parity
#7120 - Rename static constants
#7284 - Rename account_ to accountID_

AMMS:

README.md

#5285 - Add MPT support to DEX (Mostly covered already, latest changes)
#7040 - Frozen-check refactor
#6453 - Refactoring - TokenHelpers/RippleStateHelpers/AccountRootHelpers
#6571 - Style refactoring
#6676 - rippleCredit -> directSendNoFee
#7120 - Rename static constants
#6138 - Rename info() to header()
#6733 - Combine AMMHelpers and AMMUtils
#7284 - Rename account_ to accountID_
