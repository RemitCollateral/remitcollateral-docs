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
signed both by the registered partner that witnessed them and by an independent
verifier. The protocol never accepts a repayment on the beneficiary's word, nor
on the guarantor's, nor on a single signature.

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
                                │  src/chain · Soroban RPC
                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Soroban contracts — settlement                                     │
│  GuarantorVault · LoanLedger · LiquidationEngine                    │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Settlement layer (Soroban)

Three contracts in a Cargo workspace, `#![no_std]`, Soroban SDK v22, built to
`wasm32v1-none`. Each is configured by a constructor that runs inside its own
deploy transaction, and the references between them are wired once, straight
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

Forfeited collateral goes to the vault's settlement address, and changing that
address waits out the [timelock](#administration).

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

**Origination** is `originate(guarantor, beneficiary, partner, principal_usd,
installment_count, interval_secs)`, signed by the guarantor. It names the
registered partner that will disburse and collect the loan, and binds the loan
to that partner for its whole life. It locks `principal × ltv` before anything is
disbursed, and allows only one live loan per guarantor–beneficiary pair at a time
(tracked by the `OpenLoan(guarantor, beneficiary)` key). Ordering matters:
collateral is locked first, and the backend only instructs the partner to
disburse afterwards. A contract rejection therefore stops the loan before any
money moves. The reverse case is not covered yet: see
[Origination rollback](#origination-rollback).

**Repayment** is `attest_repayment(partner, verifier, loan_id, amount_usd)`, and
both addresses must sign the same invocation. `partner` must be the registered
partner bound to that loan; `verifier` must be a registered verifier, and no
address can be both. Neither signature counts on its own. Collateral released
is:

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

The same rule sets the schedule. `next_due` advances only by the installments
that principal repaid actually covers, so a stream of small attestations cannot
push a due date past a missed installment, and a partial payment that leaves the
loan behind neither ends nor restarts its grace period.

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
| Off-ramp partner | Registered address | Co-sign repayment attestations on the loans it services |
| Verifier | Registered address, never also a partner | Co-sign every repayment attestation after checking it |
| Oracle | Registered address | Publish beneficiary reputation scores |
| Admin | Multisig account (2-of-3 on testnet) | Wire the contracts once, register and revoke partners and verifiers, reassign a loan's partner, set the oracle, schedule upgrades and settlement changes behind the timelock, hand the role over |
| Anyone | — | Run the liquidation cranks |

Partner and verifier registration is admin-controlled and revocable, and
revocation takes effect on the next invocation. When a partner is offboarded,
`reassign_partner` moves its open loans to another registered partner.

### Administration

Four mechanisms keep the admin role narrow and slow:

- **Constructors.** Each contract takes its admin and configuration in
  `__constructor`, which runs inside its own deploy transaction. There is no
  public initializer for someone else to call first.
- **One-time wiring.** The references between the contracts are set once after
  deployment and cannot be changed afterwards, except by an upgrade.
- **Timelock.** Upgrading a contract (`Upgrade(wasm_hash)`) and changing the
  vault's settlement address (`SetSettlement(address)`) are scheduled with
  `schedule_action`, wait out a delay fixed at deployment (48 hours by default),
  and only then run through `execute_action`. `cancel_action` withdraws a pending
  change, and `get_scheduled_action` shows it to anyone watching.
- **Multisig.** The admin is a Stellar account with its own key removed and 2 of
  its 3 signers required, which the network enforces on every `require_auth()`.
  The role moves in two steps, `propose_admin` then `accept_admin`, so it cannot
  be handed to an address nobody controls.

Every call that uses a stored entry extends its lifetime to about 120 days once
it has fallen below about 90, so live vaults and loans are not archived. An entry
left untouched for longer is archived, not lost, and can be restored.

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
  auth/         Wallet signature verification (SEP-53) and sessions
  adapters/     OffRampAdapter interface + MockOffRampAdapter
  chain/        Live Soroban client
  contracts/    ContractGateway interface + MockContractGateway
  stores/       In-memory data stores (v1)
  services/     Loan, vault, liquidation, reputation, remittance, notification, audit
  api/          Response serializers: the frontend's snake_case shapes
  jobs/         Scheduled loan lifecycle sweep
  middleware/   Wallet session, partner API key, admin auth
  routes/       Express route modules
  app.ts        Composition root
  index.ts      Starts the server
```

Three modules define the layer's outward edges:

- **`OffRampAdapter`** — `disburse`, `verifyAttestation`, `getDisbursementStatus`,
  `fetchRemittanceHistory`. One implementation per partner integration; v1 ships
  a mock.
- **`src/chain`** — the live Soroban client, connected when the three contract
  IDs are configured. It reads vault and loan state, builds the transactions a
  guarantor signs, publishes reputation as the oracle, and implements co-signed
  attestations as the verifier and the liquidation cranks.
- **`ContractGateway`** — the interface the services were first written against,
  with `MockContractGateway` behind it. It still carries the paths that are not
  on chain yet: repayments and liquidation. See
  [Integration status](#integration-status).

The backend holds three chain secrets, and none of them is an admin key: the
verifier key, the oracle key, and the key for the keyed hash that derives
beneficiary handles.

### Wallet-signed transactions

A guarantor's collateral moves only with their own wallet's signature, so the
backend never sends those transactions on its own. With the contracts connected
(`GET /api/v1/chain` reports `enabled: true`), deposits, withdrawals and loan
originations each take two calls:

1. `POST …/prepare` checks the request against the rules the contract will
   apply, builds the transaction and returns `{ xdr, hash, network_passphrase }`.
2. The guarantor's wallet signs it. The dashboard uses Freighter's
   `signTransaction`.
3. `POST …/submit` with `{ hash, signed_xdr }` accepts only the exact transaction
   it prepared, for the guarantor it prepared it for, within five minutes, then
   submits it and records the result.

Before preparing an origination, the backend publishes the beneficiary's
reputation if the chain's copy is out of date, so the LTV the ledger applies is
the LTV the backend quoted. Without the contracts configured, the direct
endpoints (`POST /vaults/deposit`, `/vaults/withdraw`, `/loans`) keep the
backend's own accounting instead.

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

### Exchange rates

A loan's principal is set in the beneficiary's local currency and priced in USD
at the off-ramp partner's own rate when it is originated, because that is the
rate the partner pays out at. The rate is stored on the loan, so its
installments and collateral releases are measured against it for the loan's
whole life. A currency the partner cannot pay out in is refused.

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

With the contracts connected, the sweep still acts on the backend's own records.
It does not yet drive the LiquidationEngine's cranks, and its grace period comes
from `GRACE_PERIOD_DAYS`, which must match the ledger's (14 days on testnet).

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

That rollback covers the backend's own accounting. On chain it is not possible
yet: the origination is the guarantor's own signed transaction, and the ledger
has no call that releases a loan's collateral before any repayment. A payout
that fails after an on-chain lock is recorded as `LOAN_DISBURSEMENT_FAILED` in the
audit trail for an operator to resolve. A cancellation co-signed by the partner
and a verifier, allowed only before any repayment, would close this gap and is
on the [roadmap](ROADMAP.md).

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
  bearer credential and a `401` clearing the session. When the backend reports
  the contracts connected, `depositCollateral`, `withdrawCollateral` and
  `createLoan` run the [prepare, sign, submit](#wallet-signed-transactions)
  sequence inside the same method, with Freighter signing, so no screen changes.

This is not just a development convenience. Because the mock implements the same
protocol maths, the UI can be reviewed and demoed against realistic behaviour
without a chain, a partner, or a database.

The pre-origination quote is computed client-side from the reputation and vault
responses and the partner's exchange rate (`GET /fx/rates/:currency`), so the
loan form reacts as the guarantor types without a round trip per keystroke, and
the collateral it shows is the collateral the backend will lock. The backend
prices the loan authoritatively at origination.

### Trust model made visible

The protocol's trust boundaries are surfaced in the UI rather than buried:

- Repayments show **who attested them**.
- Self-declared remittances are labelled as such and shown as weighted at zero.
- Remittance history below the six-month minimum is marked as not yet counting.
- Default risk appears on the origination screen **before** the loan is created,
  with the exact figure at stake.
- A beneficiary's partner KYC reference is required, because it is how the
  partner identifies them and what links two guarantors supporting the same
  person.

---

## Trust boundaries

Four rules are enforced in code rather than by convention. Each closes a path by
which a participant could otherwise improve their own terms.

**Repayments require the loan's partner and a verifier.** The chain cannot
observe a local-currency payment, so it accepts one only when the registered
partner servicing that loan and a registered verifier both sign the same
attestation — never on the beneficiary's or the guarantor's word, never from a
different partner, and never on one signature alone. On-chain this is
`require_auth()` on both addresses plus the registry and loan checks; in the
backend the partner is identified by its API key.

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

**Collateral moves only with the guarantor's signature.** The backend prepares
deposits, withdrawals and originations but holds no key that can sign them, and
it submits only the exact transaction it prepared once the guarantor's wallet
has signed it.

### What is deliberately not on-chain

The beneficiary's identity. They are a `BytesN<32>` handle derived from their
phone number and the partner's KYC reference with a keyed hash (HMAC-SHA256), so
no personally identifying data reaches a public ledger. The key matters: phone
numbers are short and patterned enough to enumerate, so a plain hash would let
anyone link on-chain loans to real people.

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
| `RepaymentAttestation` | `attestedBy`, `partnerSignature` | A partner-and-verifier co-signed `attest_repayment` invocation |
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

The contract repository's `scripts/deploy.sh` does all of this in order, and
refuses to target mainnet unless `CONFIRM_MAINNET=yes` is set:

1. Deploy `GuarantorVault` with its constructor
   `(admin, usdc_token, settlement_address, timelock_secs)`.
2. Deploy `LoanLedger` with `(admin, vault, config, timelock_secs)`, where
   `config` holds the LTV bounds, the safety buffer and the grace period.
3. Deploy `LiquidationEngine` with `(admin, vault, loan_ledger, timelock_secs)`.
4. Wire them, once: `GuarantorVault::set_loan_ledger`,
   `GuarantorVault::set_liquidation_engine` and
   `LoanLedger::set_liquidation_engine`. **Without these, origination cannot lock
   collateral, liquidation cannot forfeit it, and no loan can leave `Active`.**
5. Register the roles: `LoanLedger::set_oracle`, then
   `set_partner(admin, partner, true)` for each partner and
   `set_verifier(admin, verifier, true)` for each verifier.

Set up the admin multisig first and deploy with the settlement address pointing
at it, so forfeited collateral never lands in a single-key account. Then hand it
the admin role with `scripts/handover-to-multisig.sh` (`propose_admin`, then
`accept_admin` signed by the council). `scripts/smoke-testnet.sh` runs the whole
lifecycle against the result.

The backend connects with the contract IDs and its own chain settings:

```env
GUARANTOR_VAULT_CONTRACT_ID=
LOAN_LEDGER_CONTRACT_ID=
LIQUIDATION_ENGINE_CONTRACT_ID=
VERIFIER_SECRET_KEY=
ORACLE_SECRET_KEY=
BENEFICIARY_HANDLE_SECRET=
PARTNER_STELLAR_ADDRESS=
GRACE_PERIOD_DAYS=14   # must match the ledger's grace period
```

### Testnet

The current deployment runs the production defaults (150% base LTV, 110% floor,
5% safety buffer, 14-day grace), a 48-hour timelock and a 2-of-3 council as
admin, against a test asset rather than Circle's USDC:

| Contract | Address |
|----------|---------|
| GuarantorVault | `CD6TYOKK74XIACIS423QJ2XW3Z646AMMHEAPAIZR2SWKFTRA5F3FL3QR` |
| LoanLedger | `CDCS5WKQPSQKA65HNDT6MS3OFS36VCZDMBJZ575REFDZCSABEUQFQSIL` |
| LiquidationEngine | `CC25FFHO6CFCBZPV5J7IJV4LJWDIN2X2LIELKBBBZBAYQV42CKXWC4NU` |
| Test USDC (SAC) | `CAWDARLC5JRSXG52Q6RWJJZ5YNEI3KJJOGVNQHEFAEQMESGPXRFCSHI4` |
| Admin council (2-of-3) | `GAFDAOJ6UE3VIJ43W6WEW6D6T3MVDE5PISA74AS7AEIJE4KURQAJ7VMT` |

---

## Integration status

What follows is an honest account of what runs on chain and what does not yet,
so nobody mistakes a mock for a deployment.

### Complete

- All three Soroban contracts, with test suites and recorded snapshots, deployed
  on Stellar testnet and exercised end to end by a smoke test, including the
  multisig and the timelock.
- Wallet sign-in. The backend verifies a SEP-53 signed message and issues a
  session token, stored hashed, and every wallet route requires one.
- The live chain client. Vault figures are read from chain, and deposits,
  withdrawals and loan originations are signed in the guarantor's wallet. The
  backend has been run against the testnet deployment end to end: a deposit, a
  reputation update and an origination all settled on chain.
- Loans priced at the off-ramp partner's exchange rate.
- The full `/api/v1` surface, the reputation engine, the lifecycle sweep and the
  audit trail.
- The frontend against both `mock` and `live`, signing in Freighter when the
  contracts are connected.

### Pending

**Repayments and liquidation are not on chain yet.** With the contracts
connected, `POST /repayments/attest` updates only the backend's records, so no
collateral is released on chain, and the lifecycle sweep moves loans into grace
and default locally rather than through the LiquidationEngine. The chain client
already implements co-signed attestations and the cranks; the services do not
call them yet. Until they do, do not run with the contracts connected for real
users.

**The partner's half of an attestation.** On chain the partner signs its own
authorization entry, but the backend has no API yet for a partner to receive an
attestation, sign it and send it back.

**A failed payout cannot be undone on chain.** See
[Origination rollback](#origination-rollback).

**The off-ramp adapter is a mock.** `MockOffRampAdapter` stands in for a real
partner integration, including its exchange rates, which are fixed indicative
figures for NGN, GHS, XOF, KES and USD. Its interface is the contract each
partner implements.

**Persistence is in-memory.** State, sessions included, does not survive a
restart.

### Known cross-repository mismatches

| Mismatch | Detail |
|----------|--------|
| **Grace period** | The ledger's grace period is fixed at deployment: 14 days in `deploy.sh` and on testnet. The backend and frontend default to 7 days. Set `GRACE_PERIOD_DAYS=14` on a backend connected to that ledger, or the chain and the sweep will disagree about when a loan defaults. |

The auth header, default API port and beneficiary list mismatches recorded here
earlier are resolved.

---

## Roadmap

The forward plan — what blocks testnet, what blocks a partner integration, and
which trust assumptions v1 accepts deliberately — is maintained in
[ROADMAP.md](ROADMAP.md), so it stays in one place as items ship.
