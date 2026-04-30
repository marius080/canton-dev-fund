## Development Fund Proposal

**Author:** Marin Konjari / [LimeChain](https://limechain.tech)
**Status:** Draft
**Created:** 2026-04-06
**Label:** daml-tooling

**Champion:**
---

## Abstract

Canton x402 is an open-source SDK that integrates the [x402 protocol](https://github.com/coinbase/x402) — Coinbase's HTTP 402-based payment standard — with the Canton Network for streaming payments using USDCx. It provides TypeScript libraries for building pay-per-request APIs, metered streaming services, and autonomous agent-to-agent payment workflows, all settled privately on Canton. The SDK lowers the barrier for developers to add programmatic payments to any HTTP service on Canton, unlocking new use cases in AI inference billing, data marketplaces, and agent commerce.

## Motivation

The x402 protocol (backed by Coinbase) is gaining traction as a standard for programmatic HTTP payments. Canton's privacy model and institutional-grade settlement make it uniquely suited for x402 — but there is currently no SDK bridging the two.

This project delivers that bridge. It enables:

- **Paid APIs on Canton** — any HTTP service can charge per request or per streamed unit, settled in USDCx
- **Agent-to-agent commerce** — autonomous AI agents can pay each other for services without human intervention
- **Streaming micropayments** — metered billing for AI inference, data feeds, and compute resources
- **Institutional privacy** — Canton's sub-transaction privacy ensures payment details stay between parties

The SDK is designed as common infrastructure — open-source, modular, and reusable by any developer building on Canton.

## Rationale

**Why x402 on Canton?**
x402 revives HTTP 402 as a native payment protocol for the web. Canton provides the settlement layer with privacy, finality, and regulatory compliance that public blockchains cannot offer. The combination enables institutional-grade paid APIs — something neither technology delivers alone.

**Why a modular SDK?**
Developers should be able to use only the pieces they need. A server that charges for API access imports `@canton-x402/server`. A client paying for data imports `@canton-x402/client`. The facilitator is a standalone service. This modularity maximizes adoption surface.

**Why settlement strategies?**
One-size-fits-all settlement doesn't work. A slow data feed should settle on a timer. A fast token stream should settle in bundles. A one-shot analysis should settle at the end. The `SettlementStrategy` abstraction lets developers choose the right policy for their use case.

**Why on-ledger authorization contracts (the `PaymentAuthorization` templates)?**
EIP-3009 on EVM is a one-shot primitive — every draw requires a fresh off-chain signature because ERC-20 has no concept of an ongoing payer/payee relationship. Canton gives us persistent on-ledger state, so we can encode the ongoing authorization as a Daml contract and drain it cheaply. This cuts streaming payments from N wallet round-trips (one per tick) to 1 (at session open), without weakening the security model: cap, expiry, and recipient are enforced by Daml at the ledger rather than by the facilitator's honesty. The allowance variant matches the capital efficiency of a Permit2-style pull; the escrow variant adds upfront liquidity guarantees for long-running jobs where merchants need them.

**Why agent-to-agent examples?**
AI agent commerce is the fastest-growing use case for programmatic payments. The agent trading example demonstrates that x402 on Canton works for fully autonomous workflows — no human in the loop, just wallets and HTTP.


## Specification

### 1. Objective

There is no existing SDK for integrating x402 payments with Canton. Developers who want to build paid APIs or streaming services on Canton must implement the full payment negotiation, verification, and settlement flow from scratch — including Canton Ledger API integration, wallet signing, and metered billing logic.

This project delivers a production-ready, modular SDK that handles the entire x402 lifecycle on Canton:

- **Server-side**: Express middleware that gates endpoints behind x402 payment requirements
- **Client-side**: Fetch wrapper and streaming client that auto-negotiate and pay
- **Facilitator**: Trust anchor service that verifies authorizations and settles USDCx transfers on Canton
- **Settlement strategies**: Configurable policies for when metered payments are settled (timer, bundle, hybrid, end-of-stream)
- **Authorization mechanisms**: Three Canton-native payment-authorization primitives covering fixed-price, metered allowance, and long-running escrow (see Section 3)

The intended outcome is that any developer can add Canton-settled x402 payments to an HTTP service in under 10 lines of code.

### 2. Implementation Mechanics

The SDK is structured as a TypeScript monorepo with four packages plus a Daml package for on-ledger authorization contracts:

**`@canton-x402/core`** — Foundation types and utilities

- Canton payment scheme types (`CantonPaymentRequirements`, `CantonPaymentPayload`, `SettlementStrategy`)
- Atomic unit conversion for USDCx (6 decimals)
- Payment header encoding/decoding (base64 X-PAYMENT header)
- `SettlementController` class supporting four configurable strategies
- Retry with exponential backoff and error classification

**`@canton-x402/client`** — Client-side payment and wallet integration

- `createCantonX402Fetch()` — wraps `fetch()` to auto-handle 402 → sign → retry
- `CantonStreamingClient` — async generator for metered SSE streams with periodic settlement
- Wallet adapters: CIP-0103 dApp SDK (browser), Canton Wallet SDK (server), test/custom wallets

**`@canton-x402/server`** — Server-side Express middleware

- `x402PaymentGate()` — fixed-price payment gate using interactive submission (verify + settle before response)
- `x402Metered()` — metered streaming gate with session tracking, configurable settlement strategy, and `authorizationMode: "allowance" | "escrow"` selector
- `createFacilitatorClient()` — HTTP client for remote facilitator services

**`@canton-x402/facilitator`** — Facilitator service

- `verifyCantonPayment()` — signature, amount, recipient, time window, and balance verification
- `settleCantonPayment()` — submits USDCx transfers via Canton JSON Ledger API V2 using interactive submission for fixed-price flows and `PaymentAuthorization.Execute` for metered flows
- `CantonLedgerClient` — queries holdings via Token Standard (CIP-0056), orchestrates `/v2/interactive-submission/prepare` and `/v2/interactive-submission/execute` for party-signed transactions
- `MockCantonLedgerClient` — in-memory ledger for testing without a Canton node

**`canton-x402-daml`** — Daml package (delivered in Milestone 5)

- `PaymentAuthorization` templates (allowance and escrow variants) for on-ledger authorization contracts
- Packaged as a versioned DAR for deployment alongside the facilitator

Two self-contained examples demonstrate the SDK:

- **Streaming Payments** — pay-per-request and metered streaming with React dashboard
- **Agent Trading** — four independent agent servers (Trading Agent, Data Agent, Correlation Agent, Facilitator) demonstrating autonomous agent-to-agent commerce with continuous trading mode

### 3. Payment Authorization Mechanism

The x402 specification was designed around ERC-20 tokens and EIP-3009 (`transferWithAuthorization`), neither of which exists natively on Canton. This section documents exactly how the SDK reconstructs x402's signing semantics on Canton's Token Standard (CIP-56) and interactive submission APIs.

#### 3.1 Mapping ERC-20 / EIP-3009 onto CIP-56

| x402 on EVM                                                                      | x402 on Canton                                                                                                                                           |
| -------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ERC-20 account balance (`balanceOf`)                                             | Sum of `Holding` contracts via interface filter `#splice-api-token-holding-v1:Splice.Api.Token.HoldingV1:Holding` on `/v2/state/active-contracts`        |
| `transfer` / `transferFrom` function                                             | `Transfer` choice on the Holding interface (or TransferInstruction accept pattern)                                                                       |
| EIP-712 typed signature over `{from, to, value, validAfter, validBefore, nonce}` | Canton external signature — either over the canonical authorization JSON (PoC) or over a Canton transaction hash via interactive submission (production) |
| EVM address + secp256k1 key                                                      | Canton party ID + party signing key (dApp SDK CIP-0103 / Wallet SDK)                                                                                     |
| Nonce tracked in ERC-20 contract storage                                         | Nonce field in authorization + Canton's native `commandId` / `submissionId` deduplication                                                                |

The semantic content of an x402 payment authorization is preserved — the signed object still encodes (from, to, amount, nonce, validity window). The signing target and settlement primitive differ.

#### 3.2 Three mechanisms, selected by payment model

| Mechanism                                   | Used by                              | Capital locked? | Payer signatures per session | Merchant liquidity guarantee |
| ------------------------------------------- | ------------------------------------ | --------------- | ---------------------------- | ---------------------------- |
| **(a) Interactive submission per request**  | `x402PaymentGate` (fixed-price)      | No              | One per request              | Per request (immediate)      |
| **(b1) `PaymentAuthorization` — allowance** | `x402Metered` (default)              | No              | One per session              | Per tick (best-effort)       |
| **(b2) `PaymentAuthorization` — escrow**    | `x402Metered` (opt-in, long-running) | Yes, up to cap  | One per session              | Upfront, up to cap           |

All three mechanisms share the same `CantonWallet.sign()` surface in `@canton-x402/client`. The HTTP 402 negotiation, middleware surface, and facilitator endpoints are identical across the three. Developers select the mechanism by choosing the middleware and setting `authorizationMode` on `x402Metered`.

#### 3.3 Mechanism (a) — Fixed-price: interactive submission per request

- **Used by:** `x402PaymentGate` (fixed-price payment middleware)
- **Payer signatures:** one per request
- **Funds locked:** no
- **EVM analog:** EIP-3009 `transferWithAuthorization` — sign off-chain, relayer submits on-chain

**Flow (per request):**

1. Server returns `402` with payment requirements (`amount`, `payTo`, `synchronizerId`).
2. Client signs a `CantonAuthorization` struct, attaches it to `X-PAYMENT`, retries.
3. Facilitator's `/verify` checks signature, amount, recipient, time window, and USDCx balance (via the Token Standard interface query).
4. Facilitator calls `/v2/interactive-submission/prepare` to construct a `Transfer` exercise against a resolved Holding contract. The participant returns a canonicalized transaction hash plus any disclosed contracts.
5. Facilitator forwards the hash to the payer's wallet adapter (`CantonDappWallet` / `CantonWalletSdkWallet`) for signing with the payer's party key.
6. Facilitator calls `/v2/interactive-submission/execute` with the signature. The participant verifies the signature natively and submits the transaction.

#### 3.4 Mechanism (b1) — Metered default: `PaymentAuthorization` allowance

- **Used by:** `x402Metered` in default mode (`authorizationMode: "allowance"`)
- **Payer signatures:** one per _session_ (not per tick)
- **Funds locked:** no
- **EVM analog:** Permit2-style signed allowance with a pull worker

**Flow:**

- **Session open — one payer signature.** Client creates a `PaymentAuthorization` Daml contract via a single interactive-submission round. The contract encodes `(amountCap, validBefore, payee, facilitator)` and becomes the on-ledger authorization for the entire session.
- **Each settlement tick — no payer signature.** Facilitator submits `Execute(amount)` signed with its own party key. The Daml choice asserts `now ≤ validBefore` and `amountDrawn + amount ≤ amountCap`, exercises `Transfer` on one of the payer's USDCx holdings for `amount` (paying the merchant), and re-creates the contract with updated `amountDrawn`.
- **Failure mode.** If the payer has moved their USDCx elsewhere, the `Transfer` exercise fails at the ledger. The facilitator's `/settle` returns `success: false`, the next `trackUsage()` call returns `false`, and the stream terminates. Merchant loss is strictly bounded by `(tick size) × (ticks delivered since last successful settlement)`.
- **Session close.** Facilitator exercises `Close`, or the contract expires via `validBefore`. No funds to return — none were locked.

**Loss-bound tuning.** Merchant exposure is set by the `SettlementStrategy`. Default for allowance mode is `{ kind: "hybrid", intervalMs: 5_000, bundleSize: 5 }` — bounded at 5 seconds or 5 ticks, whichever fires first.

#### 3.5 Mechanism (b2) — Long-running opt-in: `PaymentAuthorization` escrow

- **Used by:** `x402Metered` with `authorizationMode: "escrow"`
- **Payer signatures:** one per session
- **Funds locked:** yes, up to `amountCap`
- **EVM analog:** Sablier-style escrowed stream, with the cap locked upfront

Mechanism (b2) uses the **same `PaymentAuthorization` template as (b1)** with one additional field (`lockedHolding : ContractId Holding.I`) and one additional action at session open: the payer locks `amountCap` of their USDCx to the authorization. The cap, expiry, recipient enforcement, and tick-by-tick resolution all behave identically to (b1). The only substantive difference is _where the money comes from_ during each `Execute` — the locked Holding instead of the payer's free holdings.

**Flow:**

- **Estimate + buffer.** Server's 402 response includes an `estimatedCap` hint. Client computes `amountCap = estimatedCap × (1 + buffer)` (configurable; default 10–20%).
- **Session open — one payer signature.** A single atomic Canton transaction (a) locks `amountCap` of the payer's USDCx and (b) creates the `PaymentAuthorization` contract referencing that locked Holding.
- **Each settlement tick.** `Execute(amount)` splits the locked Holding: `amount` is unlocked and transferred to the merchant (so the merchant receives real, unlocked USDCx per tick); the remainder stays locked under a re-created authorization. Escrow does _not_ mean "hold everything until close".
- **Cap overrun.** When `amountDrawn + amount > amountCap`, the Daml choice fails and the stream terminates. Resolution: client opens a new session, or a payer-controlled `TopUp` choice extends the cap (final design deferred to implementation).
- **Session close.** Facilitator exercises `Close`; the remaining locked Holding unlocks back to the payer. Unused estimate returns automatically.
- **Refund safety.** If the facilitator goes offline, the payer exercises a `Reclaim` choice after `validBefore` and recovers the locked funds unilaterally. Funds cannot be stranded.

**Key difference from (b1):** merchant gets a guaranteed liquidity pool up to `amountCap`, at the cost of the payer's capital being tied up for the session.

#### 3.6 Security and capital properties

- **Cap enforcement.** Daml rejects any `Execute(amount)` that would exceed `amountCap`. Facilitator trust is bounded by "what the ledger allows".
- **Expiry enforcement.** Daml rejects `Execute` after `validBefore`. No stranded authorizations.
- **Recipient enforcement.** The `payee` party is baked into the contract at creation. Facilitator cannot redirect funds.
- **Replay protection.** The `nonce` field plus Canton's native `commandId` deduplication prevent replay of either the authorization creation or individual `Execute` calls.
- **Refund safety (escrow).** Locked funds return to the payer via `Close` or via payer-controlled `Reclaim` after expiry.
- **Privacy.** Only payer, payee, and facilitator are disclosed parties on the `PaymentAuthorization` contract. Canton's sub-transaction privacy ensures no third party observes the payment flow.

### 4. Architectural Dependencies

- **Canton Ledger API V2** — the facilitator's `CantonLedgerClient` uses the JSON Ledger API for balance queries (`/v2/state/active-contracts`), fixed-price transfer submission (`/v2/interactive-submission/prepare` + `/execute`), and metered settlement (`Execute` choice on `PaymentAuthorization`).
- **Token Standard (CIP-0056)** — USDCx holdings are queried via the Holding interface filter. The `PaymentAuthorization` templates interact with the Token Standard's `Transfer` choice and (for escrow mode) lock semantics.
- **dApp SDK (CIP-0103)** — the `CantonDappWallet` adapter enables browser-based transaction-hash signing for interactive submission.
- **USDCx** — all payments denominated in USDCx (USDC-backed stablecoin on Canton via Circle xReserve).
- **Privacy** — Canton's sub-transaction privacy ensures payment details are visible only to involved parties, critical for institutional use cases.

### 5. Backward Compatibility

No backward compatibility impact. This is a new SDK that does not modify any existing Canton protocol or infrastructure. It communicates with Canton nodes via the standard JSON Ledger API.

## Milestones and Deliverables

### Milestone 1: Core SDK & Payment Primitives

- **Estimated Delivery:** 3 weeks from approval
- **Focus:** Foundation SDK with full x402 lifecycle support for Canton
- **Deliverables / Value Metrics:**
  - `@canton-x402/core` — types, scheme, encoding, retry, error handling
  - `@canton-x402/server` — `x402PaymentGate` and `x402Metered` Express middleware
  - `@canton-x402/client` — x402-aware fetch wrapper, streaming client, 4 wallet adapters (test, custom, CIP-0103, Wallet SDK)
  - `@canton-x402/facilitator` — verify/settle pipeline, Canton JSON Ledger API V2 client, mock ledger
  - Open-source repository with MIT license

### Milestone 2: Configurable Settlement Strategies

- **Estimated Delivery:** 2 weeks after M1
- **Focus:** Flexible settlement policies for metered payments
- **Deliverables / Value Metrics:**
  - `SettlementController` class with 4 strategies: timer, bundle, hybrid, end-of-stream
  - Shared controller used by both server middleware and client streaming
  - 20+ strategy-specific tests (timer firing, bundle thresholds, hybrid triggers, concurrency guards)

### Milestone 3: Streaming Payments Reference Example

- **Estimated Delivery:** 2 weeks after M2
- **Focus:** End-to-end demonstration of pay-per-request and metered streaming
- **Deliverables / Value Metrics:**
  - Self-contained example with harness server and React dashboard
  - Live visualization: animated sequence diagrams, balance cards, protocol console, transaction log
  - Pay-per-request (0.01 USDCx) and metered streaming (0.001 USDCx/token) demos
  - Insufficient funds handling and mid-stream balance drain testing
  - Docker compose for one-command startup

### Milestone 4: Agent-to-Agent Trading Reference Example

- **Estimated Delivery:** 2 weeks after M3
- **Focus:** Demonstrate autonomous agent commerce using x402 on Canton
- **Deliverables / Value Metrics:**
  - Four independent server processes: Facilitator, Data Agent (metered streaming), Correlation Agent (pay-per-request), Trading Agent (orchestrator)
  - Full trading pipeline: stream market data → stop when context window full → pay for correlation analysis → execute trade decision
  - Continuous trading mode: loops until budget depletes with randomized symbols and tick counts
  - React dashboard with pipeline visualization, trade history, agent balance cards
  - Docker compose with 5 services and proper dependency ordering

### Milestone 5: Production Readiness & Canton-Native Authorization

- **Estimated Delivery:** 6 weeks after M4
- **Focus:** Implement Canton Primitives (interactive submission + on-ledger contracts)
- **Deliverables / Value Metrics:**
  - `canton-x402-daml` package with `PaymentAuthorization` templates (allowance and escrow variants), published as a versioned DAR
  - Facilitator integration with Canton interactive submission — `/v2/interactive-submission/prepare` and `/v2/interactive-submission/execute`
  - CIP-0103 signature verification wired through the dApp SDK and Wallet SDK adapters (replaces accept-all PoC verifier)
  - `authorizationMode: "allowance" | "escrow"` selector in `x402Metered` with documented UX and wallet-prompt copy for each mode
  - `estimatedCap` hint in 402 responses for escrow flows and configurable client-side buffer
  - Real Canton Ledger API integration tested on Canton DevNet (replace mock ledger for end-to-end flows)
  - End-to-end tests on DevNet covering: pay-per-request (interactive submission), metered allowance (cap enforcement, expiry, payer-balance-drained stream termination), metered escrow (lock, tick resolution, refund on close, refund on timeout)
  - npm package publishing (`@canton-x402/core`, `client`, `server`, `facilitator`) and Daml DAR release
  - CI/CD pipeline with automated build, test, and lint

### Milestone 6: Developer Adoption & Ecosystem Growth

- **Estimated Delivery:** 4 weeks after M5
- **Focus:** Lower adoption barrier and grow developer community
- **Deliverables / Value Metrics:**
  - Comprehensive integration guide with step-by-step walkthroughs covering all three authorization mechanisms
  - Video tutorial demonstrating SDK usage from scratch
  - Deployment documentation for the facilitator Docker image (DevNet and production configurations)
  - Blog post and/or developer workshop
  - 6-month maintenance commitment (bug fixes, dependency updates, community support)

### Deliverables Summary

The project ships the following software and documentation artifacts, grouped by category and tagged with primary consumer. This table is intended as a checklist for committee evaluation — each milestone's deliverables map onto rows here.

| Category  | Artifact                                                     | Primary consumer     | Available in            | Distribution            |
| --------- | ------------------------------------------------------------ | -------------------- | ----------------------- | ----------------------- |
| SDK       | `@canton-x402/core`                                          | Developer            | M1 (repo), M5 (publish) | npm                     |
| SDK       | `@canton-x402/client`                                        | Developer            | M1 (repo), M5 (publish) | npm                     |
| SDK       | `@canton-x402/server`                                        | Developer            | M1 (repo), M5 (publish) | npm                     |
| SDK       | `@canton-x402/facilitator`                                   | Operator             | M1 (repo), M5 (publish) | npm                     |
| Daml      | `PaymentAuthorization` source (allowance + escrow templates) | Developer / reviewer | M5                      | Git repository          |
| Daml      | Compiled DAR (`canton-x402-daml-X.Y.Z.dar`)                  | Operator             | M5                      | GitHub Releases         |
| Daml      | Generated TypeScript bindings (`daml codegen js` output)     | Facilitator build    | M5                      | Git repository          |
| Container | Facilitator Docker image                                     | Operator             | M5                      | Git repository          |
| Container | Streaming-payments example image                             | Developer / reviewer | M3 (repo)               | Git repository          |
| Container | Agent-trading example images                                 | Developer / reviewer | M4 (repo)               | Git repository          |
| Example   | `examples/streaming-payments` source                         | Developer            | M3                      | Git repository          |
| Example   | `examples/agent-trading` source                              | Developer            | M4                      | Git repository          |
| Tests     | Vitest unit + integration test suites                        | Reviewer             | M1–M4                   | Git repository          |
| Tests     | Daml Script tests (cap, expiry, unauthorized party, reclaim) | Reviewer             | M5                      | Git repository          |
| Docs      | `README.md` + `INTEGRATION.md`                               | Developer            | M1, extended through M6 | Git repository          |
| Docs      | API reference (TypeDoc generated)                            | Developer            | M6                      | GitHub Pages            |
| Docs      | Daml template reference                                      | Operator / developer | M5                      | Git repository          |
| Docs      | Video tutorial                                               | Developer            | M6                      | Dev Blog / similar      |
| Docs      | Blog post / case study                                       | Ecosystem            | M6                      | LimeChain + Canton blog |

## Acceptance Criteria

The Tech & Ops Committee will evaluate completion based on:

- All SDK packages build, pass tests, and export documented APIs
- Examples run end-to-end via Docker compose with no manual setup
- Payment flows are verifiable: balances change correctly, settlements produce transaction IDs
- Settlement strategies behave as documented (timer fires on interval, bundle fires at threshold, etc.)
- `PaymentAuthorization` Daml templates deployed to Canton DevNet, with allowance and escrow flows both exercised in automated tests
- Interactive submission flow (prepare + execute) demonstrated end-to-end for fixed-price payments on DevNet
- Daml-level negative tests: over-draw rejected, expired `Execute` rejected, unauthorized party rejected, escrow `Reclaim` returns funds after timeout
- Canton Ledger API integration demonstrated on DevNet (Milestone 5)
- Documentation is sufficient for a developer unfamiliar with the project to build a paid API in under 30 minutes
- All code is open-source under MIT license

## Funding

**Total Funding Request:** €140,000 (equivalent in Canton Coin, computed at submission using the then-current spot rate)

### Payment Breakdown by Milestone

#### Delivery Milestones

- Milestone 1 (Core SDK & Payment Primitives): €15,000
- Milestone 2 (Settlement Strategies): €9,500
- Milestone 3 (Streaming Payments Example): €7,000
- Milestone 4 (Agent-to-Agent Trading Example): €14,000
- Milestone 5 (Production Readiness & Canton-Native Authorization): €45,000
- Milestone 6 (Developer Adoption): €31,500
- Maintenance (Github, NPM, 6 months): €18,000

#### Payment Milestones

| Payment Milestone | Delivery Milestone | Amount     | Trigger      |
| ----------------- | ------------------ | ---------- | ------------ |
| Milestone 1       | Deliverables 1-4   | CC 370,000 | On Delivery  |
| Milestone 2       | Deliverable 5      | CC 350,000 | On Delivery  |
| Milestone 3       | Deliverable 6      | CC 250,000 | On Delivery  |
| Milestone 4       | Maintenance        | CC 130,000 | On Delivery  |

### Volatility Stipulation

The total project duration is estimated at aproximately 18 weeks. Should the timeline extend beyond that timeline due to Committee-requested scope changes, remaining milestones will be renegotiated to account for significant USD/CC price volatility.

## Co-Marketing

Upon release, LimeChain will collaborate with the Canton Foundation on:

- Joint announcement of the Canton x402 SDK
- Technical blog post on agent-to-agent payments with Canton + x402
- Presentation at a Canton developer event or workshop
- Case study highlighting the SDK's role in enabling new Canton use cases
