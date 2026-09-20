<div align="center">

# Featherpay

**Micropayments for the Rest of Us · Built on Stellar with Soroban**

Tip a creator. Pay per article. Settle a freelance invoice.
No wallet to install. No seed phrase to lose. No crypto vocabulary anywhere.

[Documentation](#) · [Architecture](#architecture) · [Contributing Guide](#contributing)

</div>

---

## Table of Contents

- [The Problem](#the-problem)
- [The Solution](#the-solution)
- [How It Works, End to End](#how-it-works-end-to-end)
- [Architecture](#architecture)
- [Repositories](#repositories)
- [Why Stellar](#why-stellar)
- [The Embedded Wallet Model](#the-embedded-wallet-model)
- [Fee Model & Batching](#fee-model--batching)
- [Security](#security)
- [Compliance & Regulatory Posture](#compliance--regulatory-posture)
- [Tech Stack](#tech-stack)
- [Roadmap](#roadmap)
- [FAQ](#faq)
- [Contributing](#contributing)
- [License](#license)

---

## The Problem

Small payments are broken on the traditional rails. A $1 tip to a blogger, a $0.50 pay-per-article unlock, a $5 freelance milestone — all of these get crushed by fixed-cost payment processing fees (often 30¢ plus a percentage per transaction). The economics simply don't work at that size, so creators and freelancers either don't offer micropayment options at all, or bundle everything into large, infrequent invoices that lose the "pay for exactly what you value" model entirely.

Crypto rails solve the *fee* problem — Stellar settlement costs a fraction of a cent — but historically introduce a *usability* problem instead: wallets, seed phrases, gas, exchanges. The audience for a $1 tip (a reader, a viewer, a casual supporter) is not going to install a browser extension and buy crypto to send it.

## The Solution

**Featherpay is a micropayment platform where the blockchain is invisible.** Under the hood, every payment settles on Stellar using Soroban-adjacent infrastructure. On the surface, a tipper sees a "Tip $1" button, pays with a card or Apple/Google Pay, and is done in one tap. A creator sees a dashboard with a balance and a "Withdraw to bank" button. Neither ever sees a wallet address, a private key, or the word "blockchain."

This is achieved through an **embedded, custodial wallet model**: Featherpay generates and manages a Stellar keypair for every user behind the scenes, tied to their email or social login, and abstracts all signing and settlement behind a standard web/app experience.

## How It Works, End to End

1. A creator signs up with email. Featherpay silently provisions them a Stellar wallet.
2. The creator gets a shareable tip link (`featherpay.io/@creatorname`) or an embeddable widget for their own site.
3. A reader/viewer/fan clicks "Tip $2," enters card details in a standard hosted payment form, and confirms.
4. Behind the scenes: the card charge is processed and converted to USDC — this conversion step may be pooled with other small tips over a short window to keep processing costs down.
5. The API layer builds a call into the `TipRouter` smart contract's `send_tip` function, sends it to the wallet custody service for signing (never exposing the private key), and submits it to the Stellar network. The contract itself pulls the converted funds from the tipper and forwards them to the creator — and to a Featherpay treasury account, if a protocol fee applies — atomically, in one transaction.
6. The creator's dashboard balance updates. They can withdraw to their bank account at any time via a fiat off-ramp.

From the outside, steps 4–5 are invisible. It looks like Venmo. It settles like Stellar — routed, atomically, through a smart contract.

## Architecture

```
                                   ┌─────────────────────┐
                                   │      frontend         │
                                   │  (tip widget + creator │
                                   │      dashboard)         │
                                   └──────────┬───────────┘
                                              │ HTTPS
                                              ▼
                                   ┌─────────────────────┐
                                   │        api             │
                                   │  (orchestration:        │
                                   │  accounts, payments,     │
                                   │  tips, batching, ledger) │
                                   └──────┬───────────┬───┘
                                          │           │
                         internal, authenticated      │ external
                                          ▼           ▼
                             ┌──────────────────┐  ┌──────────────────┐
                             │  wallet-service     │  │  Anchor / card     │
                             │  (key custody,       │  │  processor          │
                             │   signing only —      │  │  (fiat ↔ USDC       │
                             │   never exposed        │  │   conversion,        │
                             │   publicly)            │  │   bank payouts)      │
                             └─────────┬────────┘  └──────────────────┘
                                       │ signed call into
                                       │ TipRouter.send_tip(...)
                                       ▼
                             ┌──────────────────┐
                             │  contracts            │
                             │  (TipRouter contract —  │
                             │   pulls funds from        │
                             │   tipper, forwards to      │
                             │   creator + treasury,       │
                             │   atomically)                 │
                             └─────────┬────────┘
                                       │
                                       ▼
                             ┌──────────────────┐
                             │   Stellar Network    │
                             │   (Soroban RPC)        │
                             └──────────────────┘
```

**Design principle:** the service that talks to users (`frontend`) never talks to the service that holds keys (`wallet-service`). Every request to sign something is mediated by `api`, and `wallet-service` is unreachable from the public internet entirely. Every tip, without exception, is routed through the `TipRouter` contract in `contracts` — `api` never constructs a plain point-to-point transfer for a tip — so that the payment and any protocol fee split happen atomically, and every tip is independently verifiable from the contract's on-chain events.

## Repositories

| Repo | Description | Status |
|---|---|---|
| [`wallet-service`](https://github.com/featherpay/wallet-service) | Embedded wallet custody: keypair generation, encrypted storage, signing, recovery. The most security-sensitive repo in the org — internal-only, no public surface. | In development |
| [`api`](https://github.com/featherpay/api) | Orchestration layer: accounts, fiat on/off-ramp, tip processing, batching, ledger, reconciliation. | In development |
| [`frontend`](https://github.com/featherpay/frontend) | Tip widget, embeddable snippet, creator dashboard. No crypto terminology anywhere in the UI. | In development |
| [`contracts`](https://github.com/featherpay/contracts) | The `TipRouter` Soroban contract that every tip is routed through — receives the payment from the tipper and forwards it to the creator (and a treasury, if a fee applies) atomically. A required, active part of the core payment path, not an optional layer. | In development |

Each repo has its own `PLAN.md` (milestones, design decisions, risks) and `README.md` (setup, architecture, API reference).

## Why Stellar

- **Sub-cent settlement fees** — the fee floor that makes $0.50–$5 payments economically sane in the first place
- **Fast finality** (seconds, not minutes) — a tip should feel instant even if the underlying settlement is deliberately batched
- **Native asset issuance and anchors** — USDC is available natively on Stellar, and the anchor network provides established fiat on/off-ramp infrastructure rather than requiring Featherpay to build banking relationships from scratch
- **Soroban** — every tip is routed through the `TipRouter` smart contract, which atomically forwards a tip to its creator and splits off a protocol fee where applicable; Soroban is what makes that atomicity possible without a second, separate settlement step

## The Embedded Wallet Model

This is the architectural decision that makes Featherpay usable by a mainstream, non-crypto audience, and it deserves to be stated plainly:

- Every user — creator or tipper — gets a Stellar keypair generated on signup, tied to their email/social identity
- Private keys are encrypted at rest via envelope encryption (KMS-backed) and **never exposed** to any service other than `wallet-service` itself
- Signing happens inside `wallet-service`; `api` sends a transaction payload and receives a signed envelope back, never the key
- Recovery is email-based — no seed phrase for a user to write down, lose, or have stolen
- This is a **fully custodial** model for v1. Featherpay is functionally holding funds on behalf of users, which is a deliberate tradeoff of regulatory/security responsibility in exchange for the frictionless UX a mainstream tipper needs. See [Compliance & Regulatory Posture](#compliance--regulatory-posture) below — this is not a footnote.

## Fee Model & Batching

A common misconception worth correcting up front: **Stellar's fee is not the bottleneck for a $1 tip funded by a card.** Card network and processor fees (often 30¢ plus a percentage) dominate the cost stack at that size — Stellar's near-zero fee only delivers its promise once the fiat-processing side is accounted for.

Featherpay addresses this in two layers. First, individual tips are recorded immediately in the internal ledger, so creator balances update in near-real-time from the user's perspective, regardless of what's happening on-chain. Second, the **fiat-side** conversion and off-ramp operations are pooled across a short window to reduce processor overhead — this is where most of the cost savings actually come from at micropayment scale. On-chain settlement itself happens **per tip**, via a signed call into the `TipRouter` contract, since the contract requires each tipper's individual authorization; this is fine because Stellar's own transaction fee is already a fraction of a cent, batching or not. The atomicity the contract provides — tip and fee split succeeding or failing together — is worth more than pooling on-chain calls would save.

## Security

- `wallet-service` is internal-only, reachable exclusively by `api` over authenticated service-to-service channels — never exposed to the public internet
- No plaintext key material in logs, error messages, or version control, enforced via automated secret-scanning in CI
- Every signing request is audit-logged
- Security-sensitive code (anything in `wallet-service`, and payment-triggering endpoints in `api`) requires a second reviewer, not just standard PR review
- A dedicated penetration test on `wallet-service` is planned ahead of the broader protocol audit, given it carries the highest concentration of risk
- **Status:** Featherpay has **not yet undergone a formal security audit**. Do not deploy any component to production with real funds until an audit is complete and published.

Found a vulnerability? **Do not open a public issue.** Email `security@featherpay.io`.

## Compliance & Regulatory Posture

Featherpay's custodial model — holding user funds, converting fiat to crypto and back on their behalf — puts it in similar regulatory territory to a payments company, not just a blockchain app. This is flagged explicitly, in every relevant repo's plan, as a **hard gate**: legal review of money-transmitter licensing requirements (which vary significantly by jurisdiction) must happen before any component touches real funds at scale. This is not a formality to check off — it shapes which jurisdictions Featherpay can operate in at launch and how it structures custody.

## Tech Stack

| Layer | Choice |
|---|---|
| Smart contracts (optional) | Rust + Soroban SDK |
| Backend (`api`, `wallet-service`) | Node.js + TypeScript (proposed — confirm per repo) |
| Database | PostgreSQL |
| Key custody | Envelope encryption via cloud KMS (AWS KMS / GCP KMS) |
| Frontend | Next.js + TypeScript |
| Fiat on/off-ramp | Card processor (e.g. Stripe) + Stellar anchor for USDC conversion |
| Ledger network | Stellar (testnet during development, mainnet gated on audit + legal readiness) |

## Roadmap

High-level, org-wide view — see each repo's `PLAN.md` for full milestone detail:

1. **Foundations** — repo scaffolding, threat modeling for `wallet-service`, core data models
2. **Embedded wallet + signing** — keypair generation, encrypted storage, signing service
3. **`TipRouter` contract v1** — Soroban contract that atomically receives a tip and forwards it to the creator, with optional fee splitting
4. **Tip flow v1** — card payment → USDC conversion → signed call into `TipRouter` → creator balance update
5. **Fiat-side batching** — pool small tips' conversion/off-ramp operations to make the fee economics actually work
6. **Creator dashboard + withdrawals** — earnings view, bank payout flow
7. **Security hardening + audit** — internal review, third-party penetration test on `wallet-service`, dedicated audit on the `TipRouter` contract given it routes every tip's funds, broader audit before real funds
8. **Testnet beta** — real (small) fiat flows against testnet, real creators
9. **Mainnet launch** — gated on legal sign-off for custodial money-transmission questions, launched with conservative tip/withdrawal caps enforced at both the `api` and contract level

## FAQ

**Do users need a crypto wallet?**
No. Every account gets a Stellar wallet created automatically and invisibly. Users never see a private key or seed phrase.

**Is this custodial?**
Yes, by design, for v1 — see [The Embedded Wallet Model](#the-embedded-wallet-model) and [Compliance & Regulatory Posture](#compliance--regulatory-posture) for why, and what that implies.

**Why not just use existing payment rails (Stripe, PayPal)?**
Their fixed-cost fees make sub-$5 payments uneconomical. Featherpay exists specifically to make that size of payment viable, using Stellar's near-zero settlement cost combined with batching to also absorb the fiat-side processing costs.

**Is there an on-chain smart contract?**
Yes — every tip is routed through the `TipRouter` Soroban contract in `contracts`. It receives the tip payment from the tipper and forwards it to the creator, splitting off a protocol fee to a treasury account where applicable, all in one atomic transaction. `api` never builds a plain wallet-to-wallet transfer for a tip.

**Has this been audited?**
Not yet. See [Security](#security).

## Contributing

Each repo has its own contributing section with repo-specific conventions. Org-wide expectations:

- Branch from `main` in the relevant repo
- Security-sensitive changes (anything touching `wallet-service`, or payment-triggering endpoints in `api`) require a second reviewer
- No real key material, card numbers, or PII in test fixtures, ever — use clearly-fake deterministic test data
- Link the relevant `PLAN.md` milestone in your PR description

## License

MIT

---

<div align="center">

Built on [Stellar](https://stellar.org/) · Powered by [Soroban](https://soroban.stellar.org/)

</div>
