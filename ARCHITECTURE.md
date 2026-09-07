# Architecture

RemitCollateral lends local currency to a person with no wallet, secured by
crypto collateral posted by someone else, and repaid through a channel the chain
cannot observe. Every structural decision in the system follows from that one
sentence.

This document describes what each layer does, why the boundaries fall where they
do, and where the current implementation is complete versus stubbed.

---

## Core principle: the chain custodies, the partner observes

The protocol splits along a single seam — **what the chain can verify** versus
**what only a licensed partner can witness**.

The chain can verify that USDC is in a vault, that a given address authorized a
call, and what time it is. It cannot verify that a beneficiary in Lagos walked
into a mobile money agent and paid back ₦50,000. That event is only ever
witnessed by the off-ramp partner who processed it.

So the settlement layer holds the money and enforces the rules, and the
orchestration layer supplies the facts the chain cannot see — but *only* facts
attributable to a registered, revocable partner identity. The protocol never
accepts a repayment on the beneficiary's word, nor on the guarantor's.

```
┌─────────────────────────────────────────────────────────────────────┐
│  Guarantor (diaspora)                    Beneficiary (local)        │
│  Stellar wallet · Freighter              No wallet · SMS + agent    │
└───────────┬─────────────────────────────────────────┬───────────────┘
            │                                         │
            ▼                                         ▼
┌───────────────────────────┐            ┌────────────────────────────┐
│  Frontend                 │            │  Off-ramp partner          │
│  Next.js guarantor        │            │  Mobile money / bank       │
│  dashboard                │            │  Disburses · collects      │
└───────────┬───────────────┘            └────────────┬───────────────┘
            │  /api/v1                    OffRampAdapter │  attestations
            ▼                                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Backend — orchestration                                            │
│  Reputation engine · loan lifecycle · audit trail · sweep job       │
└───────────────────────────────┬─────────────────────────────────────┘
                                │  ContractGateway
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Soroban contracts — settlement                                     │
│  GuarantorVault · LoanLedger · LiquidationEngine                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Settlement layer (Soroban)

Three contracts in a Cargo workspace, `#![no_std]`, Soroban SDK v22, built to
`wasm32-unknown-unknown`. They reference each other by address and are wired
after deployment.

| Crate | Contract | Responsibility |
|-------|----------|----------------|
| `rc-guarantor-vault` | `GuarantorVaultContract` | Isolated USDC vault per guarantor |
| `rc-loan-ledger` | `LoanLedgerContract` | Loan records, schedules, reputation-adjusted LTV, attested repayments |
| `rc-liquidation-engine` | `LiquidationEngineContract` | Permissionless default cranks and collateral settlement |

### GuarantorVault

The vault is deliberately **passive**: it holds USDC and tracks how much of it is
spoken for, but it does not know what a loan is. It cannot originate, cannot
compute an LTV, and cannot decide that a payment was missed. Those decisions
belong to contracts that can be reasoned about independently.

Its entire state per guarantor is two numbers:

```rust
pub struct Vault {
    pub guarantor: Address,
    pub collateral_balance: i128,  // total USDC deposited
    pub locked_amount: i128,       // portion backing active loans
}
```

Available collateral is `collateral_balance − locked_amount`. That subtraction is
the only invariant the vault enforces on withdrawal, and it is why a guarantor
can never withdraw collateral that is backing a live loan.

Vaults are keyed by address in persistent storage, so **funds are never pooled**.
There is no path in the contract by which one guarantor's default touches
another's balance.

| Function | Caller | Effect |
|----------|--------|--------|
| `deposit` | Guarantor | Transfers USDC in, credits `collateral_balance` |
| `withdraw` | Guarantor | Transfers out, bounded by available balance |
| `lock_collateral` | LoanLedger only | Reserves collateral against a new loan |
| `release_collateral` | LoanLedger **or** LiquidationEngine | Returns collateral to available |
| `forfeit_collateral` | LiquidationEngine only | Seizes collateral, sends USDC to settlement |

The asymmetry in that table is the security model. Locking is the ledger's job
because only the ledger knows an LTV. Forfeiting is the engine's job because only
the engine knows a grace period expired. Releasing is shared, because both the
happy path (repayment) and the unhappy path (excess returned after liquidation)
need it.

### LoanLedger

Holds the loan records and the protocol configuration:

```rust
pub struct Config {
    pub base_ltv_bps: u32,        // 15000 = 150%
    pub min_ltv_bps: u32,         // 11000 = 110%
    pub safety_buffer_bps: u32,   //   500 =   5%
    pub grace_period_secs: u64,
}
```

A loan carries the guarantor's `Address` and the beneficiary's `BytesN<32>`
handle — never a beneficiary address, because there is no beneficiary wallet.

**Origination** locks `principal × ltv` before anything is disbursed, and allows
only one live loan per guarantor–beneficiary pair at a time (tracked by the
`OpenLoan(guarantor, beneficiary)` key). Ordering matters: collateral is locked
first, and the backend only instructs the partner to disburse afterwards. A
contract rejection therefore stops the loan before any money moves.

**Repayment** is `attest_repayment(partner, loan_id, amount_usd)`, gated on the
caller being a registered partner. Collateral released is:

```
releasable  = collateral × (total_repaid / principal) × (1 − safety_buffer)
release_now = releasable − collateral_already_released
```

with one exception: the installment that brings `total_repaid` up to the
principal releases **everything still locked**, safety buffer included.

The closing condition is worth stating explicitly, because it is a deliberate
defence rather than an accident of implementation:

> Closing is decided by **principal actually repaid**, never by how many
> attestations have arrived.

Counting attestations would let a partner close a loan — and release all of its
collateral — with a handful of token payments. `installments_paid` is tracked for
reporting, but it does not gate closure.

**Default transitions** (`mark_grace`, `mark_defaulted`) are restricted to the
LiquidationEngine, and each independently re-checks the clock. The ledger does
not take the engine's word that a loan is overdue.

### LiquidationEngine

Both entry points are **permissionless cranks**. Anyone may call `flag_overdue`
or `liquidate`; what they do is determined entirely by the loan's own state and
the ledger clock, so there is nothing for a caller to influence.

This is the answer to an obvious failure mode: if only the platform could record
a default, then a platform outage — or a platform that simply chose not to run
its job — would leave missed payments unrecorded, and reputation scores would
silently overstate every beneficiary. Making the crank public means the protocol
advances whether or not the operator is paying attention.

`liquidate` never decides how much is owed. It reads the outstanding balance from
the ledger and settles exactly that:

```
forfeited = min(outstanding_principal, collateral_remaining)
returned  = collateral_remaining − forfeited
```

Collateral is posted at 110–150% LTV, so seizing the whole position would take
more from the guarantor than the protocol actually lost. The excess goes back.

`poke(loan_id)` is a convenience crank that advances whichever transition the
loan is due for, returning `true` if it did anything.

### Roles

| Role | Held by | May |
|------|---------|-----|
| Guarantor | Stellar wallet | Deposit, withdraw unlocked collateral, originate against their own vault |
| Off-ramp partner | Registered address | Attest repayments |
| Oracle | Registered address | Publish beneficiary reputation scores |
| Admin | Stellar wallet | Wire the contracts, register/revoke partners, set oracle and settlement address |
| Anyone | — | Run the liquidation cranks |

Partner registration is admin-controlled and revocable, and revocation takes
effect on the next invocation.

---

## 2. Orchestration layer (backend)

Node.js, TypeScript, Express 4, served under `/api/v1`. Its job is everything the
chain cannot do: score reputation, talk to off-ramp partners, notice that time
has passed, and keep an audit trail.

### Structure

```
src/
  config/       Environment & protocol configuration
  types/        Domain entities, DTOs, adapter types
  adapters/     OffRampAdapter interface + MockOffRampAdapter
  contracts/    ContractGateway interface + MockContractGateway
  stores/       In-memory data stores (v1)
  services/     Loan, vault, liquidation, reputation, remittance, notification, audit
  jobs/         Scheduled loan lifecycle sweep
  middleware/   Wallet, partner API key, admin auth
  routes/       Express route modules
  index.ts      Composition root
```

Two interfaces define the layer's outward edges, and both ship with mocks:

- **`OffRampAdapter`** — `disburse`, `verifyAttestation`, `getDisbursementStatus`,
  `fetchRemittanceHistory`. One implementation per partner integration.
- **`ContractGateway`** — the Soroban calls, mirroring the adapter pattern:
  `depositCollateral`, `withdrawCollateral`, `lockCollateral`,
  `releaseCollateral`, `getCollateralPosition`, `recordLoan`, `recordRepayment`,
  `closeLoan`, `liquidateCollateral`.

Both are injected in `index.ts`. Swapping a mock for a live implementation is a
change at the composition root; nothing in `src/services` moves.

### Reputation and LTV

The composite score is a weighted blend on a 0–100 scale:

```
composite = remittance_score × 0.40 + repayment_score × 0.60
```

**Remittance consistency** rewards regularity, not volume. It blends three
components: frequency (how close the average gap is to 30 days), amount
consistency (via coefficient of variation, so a steady ₦40,000 scores better than
a lumpy average), and duration of history. Below `MIN_REMITTANCE_MONTHS` of
history the component is capped at 20 — enough to distinguish "some history" from
"none", not enough to move the LTV meaningfully.

Critically, **only `partner_reported` remittances count**. See
[Trust boundaries](#trust-boundaries).

**Repayment score** is the ratio of on-time installments to total resolved
installments, as a percentage. It is recomputed from the schedule rather than
stored as a running penalty, which means a default's effect on the score stays
reproducible from the underlying records.

The LTV follows linearly:

```
required_ltv = max(min_ltv, base_ltv − composite × reduction_factor)
             = max(1.10,    1.50     − composite × 0.004)
```

The contracts express the identical curve in basis points, interpolating across
the span between `base_ltv_bps` and `min_ltv_bps` by a score in basis points of a
perfect score:

```
ltv_bps = base_ltv_bps − (base_ltv_bps − min_ltv_bps) × score_bps / 10000
```

These agree at every point. A backend score of 100 maps to `score_bps = 10000`
and both yield 110%; a score of 0 yields 150% in both. The reduction per score
point is `0.004` in the backend and `4000 bps / 100 = 40 bps` on-chain — the same
number.

### Loan lifecycle and the sweep

Origination, repayment and collateral release are **request-driven**. Default is
not: nobody calls an endpoint when a payment fails to arrive.

```
active ──── installment past due ────▶ grace ──── grace expires ────▶ defaulted
   ▲                                     │
   └────────── attested repayment ───────┘
```

So overdue installments, grace expiry and default are detected by a scheduled
sweep that runs every `LIFECYCLE_SWEEP_INTERVAL_MINUTES` and once at boot.
`POST /api/v1/admin/liquidation/review` runs the same sweep on demand; it takes
no action on loans that need none, so it is safe to run at any time.

The sweep catches per-loan errors and logs them rather than aborting — one bad
loan must not stop the sweep for every other guarantor.

Marking installments overdue is not cosmetic. It is the signal the repayment
score reads. Without it, a beneficiary who never pays would score identically to
one with no history at all.

> **Single-instance assumption.** The sweep runs in-process. Running more than
> one backend instance would run it more than once per tick, so a multi-instance
> deployment needs an external scheduler or a distributed lock.

### Origination rollback

Origination touches three systems in order — chain, local state, partner — and
the failure path unwinds all of it:

1. `contractGateway.lockCollateral(...)` — a rejection here aborts before any
   local state claims the collateral is spoken for.
2. Local vault lock and loan persistence.
3. `offRampAdapter.disburse(...)`. If this throws, the on-chain lock is released,
   the loan is closed on the ledger, the local lock is unwound, and the loan
   record is deleted.

Without step 3's rollback, a failed disbursement would leave the guarantor's
collateral locked against a loan that never existed.

---

## 3. Presentation layer (frontend)

Next.js 14 App Router, React 18, TypeScript in strict mode, Tailwind. The
guarantor's view of the system: what their collateral is doing, which loans it
backs, what is due next, and — stated plainly, before they commit — what they
stand to lose.

| Route | Purpose |
|-------|---------|
| `/` | Connect a Stellar wallet; explains the model and the risk before sign-in |
| `/dashboard` | Collateral position, open loans, upcoming installments, loans at risk |
| `/loans` | Every loan the guarantor's collateral backs, filterable by status |
| `/loans/new` | Originate a loan, with a live quote of the collateral it costs |
| `/loans/[id]` | Repayment progress, installment schedule, partner attestations |
| `/beneficiaries`, `/beneficiaries/[id]` | Linked beneficiaries and reputation breakdowns |
| `/vault` | Deposit and withdraw USDC; collateral utilisation |
| `/remittances` | Remittance history — the cold-start credit signal |

### The two-implementation data layer

Every screen imports a single `api` object satisfying the `RemitCollateralApi`
interface, where each method maps to one documented `/api/v1` endpoint. Which
implementation it resolves to is one environment variable:

- **`mock`** — an in-memory backend with seeded fixtures and *real protocol
  maths*: reputation scoring, LTV adjustment, schedule generation, proportional
  collateral release. Mutations persist for the tab's lifetime; a reload resets.
- **`live`** — `fetch` against `NEXT_PUBLIC_API_URL`, with the session token as a
  bearer credential and a `401` clearing the session.

This is not just a development convenience. Because the mock implements the same
protocol maths, the UI can be reviewed and demoed against realistic behaviour
without a chain, a partner, or a database.

The pre-origination quote is derived client-side rather than fetched — it
composes the reputation and vault responses the backend already exposes — so the
loan form reacts as the guarantor types without a round trip per keystroke. The
backend prices the loan authoritatively at origination.

### Trust model made visible

The protocol's trust boundaries are surfaced in the UI rather than buried:

- Repayments show **who attested them**.
- Self-declared remittances are labelled as such and shown as weighted at zero.
- Remittance history below the six-month minimum is marked as not yet counting.
- Default risk appears on the origination screen **before** the loan is created,
  with the exact figure at stake.

---

## Trust boundaries

Three rules are enforced in code rather than by convention. Each closes a path by
which a participant could otherwise improve their own terms.

**Repayments require a partner.** The chain cannot observe a local-currency
payment, so it accepts one only when a registered off-ramp partner authorizes the
attestation — never on the beneficiary's or the guarantor's word. On-chain this
is `partner.require_auth()` plus a registry check; in the backend it is the
partner API key.

**Attestations are attributed to the authenticated partner.** The partner
identifier on a repayment comes from the API key that authenticated the request,
never from the request body. Otherwise one partner could sign an attestation into
another partner's name.

**Remittance source is assigned, not accepted.** Anything recorded through
`POST /remittances` is stored as `self_declared` and carries 0.0 weight in
scoring. Only `POST /remittances/ingest`, behind the partner API key, writes
`partner_reported` records. A guarantor therefore cannot raise a beneficiary's
score — and so cannot lower their own required LTV — by declaring remittances
that never happened.

### What is deliberately not on-chain

The beneficiary's identity. They are a `BytesN<32>` handle derived from their
phone number and the partner's KYC reference, so no personally identifying data
reaches a public ledger.

Reputation *derivation*. Scoring runs off-chain over remittance and repayment
history; only the resulting score is published on-chain, by a registered oracle,
because the score is what sets the LTV. This is an explicit trust concession —
moving part of the derivation on-chain is on the roadmap so the LTV becomes
reproducible without trusting the oracle.

---

## Data model

The backend's domain entities, and how they relate to on-chain state:

| Entity | Key fields | On-chain counterpart |
|--------|-----------|---------------------|
| `Guarantor` | `walletAddress` | The `Address` that owns a vault |
| `Vault` | `collateralBalance`, `lockedAmount` | `Vault` struct, keyed by guarantor address |
| `Beneficiary` | `phoneNumber`, `localKycRef`, `reputationScore` | `BytesN<32>` handle + published score only |
| `Loan` | `principalUsd`, `ltvRatio`, `collateralLockedUsd`, `collateralReleasedUsd`, `collateralForfeitedUsd`, `schedule`, `status` | `Loan` struct |
| `InstallmentScheduleItem` | `dueAt`, `status`, `repaidAt` | Ledger tracks counts and timestamps, not the full schedule |
| `RemittanceRecord` | `amountUsd`, `source` | Not on-chain — scoring input only |
| `RepaymentAttestation` | `attestedBy`, `partnerSignature` | An authorized `attest_repayment` invocation |
| `AuditEvent` | `eventType`, `action`, `actor`, `details` | Not on-chain |

Note the asymmetry in the loan: the backend holds the full installment schedule
with per-item status, while the chain holds `installment_count`,
`installments_paid`, `total_repaid_usd` and `next_due`. The chain needs enough to
decide overdue-ness and proportional release; it does not need the schedule.

Collateral released is tracked **on the loan**, not derived from the vault's
locked balance. A vault may back several loans at once, so `lockedAmount` is the
sum across all of them and cannot attribute a release to any one loan.

---

## Deployment and wiring

The three contracts reference each other by address, so they must be wired after
deployment. **Order matters**, and a skipped step fails quietly at the worst
moment.

1. Deploy all three contracts.
2. `GuarantorVault::initialize(admin, usdc_token, settlement_address)`
3. `LoanLedger::initialize(admin, vault, base_ltv_bps, min_ltv_bps, safety_buffer_bps, grace_period_secs)`
4. `LiquidationEngine::initialize(admin, vault, loan_ledger)`
5. `GuarantorVault::set_loan_ledger(admin, ledger)` — **without this, origination cannot lock collateral.**
6. `GuarantorVault::set_liquidation_engine(admin, engine)` — **without this, liquidation cannot forfeit.**
7. `LoanLedger::set_liquidation_engine(admin, engine)` — without this, no loan can leave `Active`.
8. `LoanLedger::set_oracle(admin, oracle)` and `LoanLedger::set_partner(admin, partner, true)` for each partner.

Then populate the backend's `.env` with the deployed contract IDs:

```env
GUARANTOR_VAULT_CONTRACT_ID=
LOAN_LEDGER_CONTRACT_ID=
LIQUIDATION_ENGINE_CONTRACT_ID=
```

---

## Integration status

v1 is complete at each layer and mocked between two of them. What follows is an
honest account of the seams, so nobody mistakes a mock for a deployment.

### Complete

- All three Soroban contracts, with test suites and recorded snapshots.
- The full `/api/v1` surface: auth, guarantors, vaults, beneficiaries, loans,
  repayments, remittances, audit, admin.
- The reputation engine, the loan lifecycle sweep, and the audit trail.
- The frontend against both `mock` and `live` transports.

### Stubbed or pending

**The contract gateway is a mock.** `MockContractGateway` satisfies the
`ContractGateway` interface and simulates transaction hashes. The live Stellar
SDK implementation, invoking the deployed contracts, is the main piece of work
between v1 and a testnet deployment. Business logic in `src/services` should not
need to change.

**The off-ramp adapter is a mock.** `MockOffRampAdapter` stands in for a real
partner integration. Its interface is the contract each partner implements.

**Persistence is in-memory.** The backend depends on `pg` and the stores are
structured for a straightforward port, but v1 state does not survive a restart.

**Wallet auth is a header stub.** The backend's `walletAuth` reads
`x-wallet-address` rather than verifying a SEP-10 signature, and
`verifyChallenge` accepts any non-empty signature. Loan reads are scoped to the
owning guarantor regardless, so ownership is enforced correctly once real signing
lands — but the header must not be treated as authentication in any deployed
environment.

**FX is 1:1.** `principalUsd = principalLocal` in origination. A real FX rate
oracle is required before local-currency loans price correctly.

### Known cross-repository mismatches

These are real discrepancies between repositories as they currently stand, not
design intentions:

| Mismatch | Detail |
|----------|--------|
| **Auth header** | The frontend sends `Authorization: Bearer <token>`; the backend's middleware reads `x-wallet-address`. `live` mode will not authenticate until these agree. |
| **Default port** | The frontend defaults `NEXT_PUBLIC_API_URL` to port `3001`; the backend defaults to `4000`. Set the variable explicitly. |
| **Grace period** | The contract deployment guidance suggests 14 days for `grace_period_secs`; the backend and frontend both default to 7 days (`GRACE_PERIOD_DAYS`). These must be reconciled at deployment, or the chain and the sweep will disagree about when a loan defaults. |
| **Beneficiary listing** | The frontend derives the beneficiary list from the dashboard payload because v1 has no `GET /beneficiaries` list endpoint. |

---

## Roadmap

The forward plan — what blocks testnet, what blocks a partner integration, and
which trust assumptions v1 accepts deliberately — is maintained in
[ROADMAP.md](ROADMAP.md), so it stays in one place as items ship.
