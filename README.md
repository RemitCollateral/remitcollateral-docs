# RemitCollateral

> Crypto-collateralized lending for local beneficiaries who never touch crypto.

A diaspora member locks USDC on Stellar to guarantee a loan. The beneficiary — a
relative or business contact back home — receives local currency through a bank
transfer or mobile money, repays through that same channel, and never needs a
wallet, a seed phrase, or any blockchain literacy at all. The guarantor's
collateral secures the loan; the beneficiary's repayment behaviour determines
whether that collateral comes back.

This repository is the **protocol documentation**. The running system lives in
three sibling repositories.

---

## The problem

Diaspora remittance is a large, reliable flow of money that produces no credit.
A worker abroad may have sent money home every month for five years, and the
recipient still cannot borrow against that history — the record sits inside a
remittance operator, is not portable, and is not legible to any lender.

Meanwhile the recipient is exactly the person a lender should be able to price:
their inbound cash flow is regular, observable, and already intermediated by a
licensed partner.

RemitCollateral turns the guarantor's on-chain collateral into the credit
enhancement that makes the loan possible, and turns the remittance record into
the signal that makes it cheaper over time.

## The approach

Three ideas carry the design:

**The beneficiary is never a crypto user.** They have no wallet and never appear
on-chain as an address. They are identified by a 32-byte handle derived from
their phone number and the off-ramp partner's KYC reference with a keyed hash, so
no personally identifying data reaches the ledger. They receive naira, cedis or shillings, and
they repay in naira, cedis or shillings.

**Collateral is isolated, not pooled.** Every guarantor has their own vault. A
default on one relationship can never reach another guarantor's funds. There is
no shared pool, no socialised loss, and no correlation between unrelated
borrowers.

**Default costs what was actually lost.** Collateral is posted at 110–150% of
principal. On default the protocol forfeits collateral equal to the
*outstanding balance only* and returns the excess to the guarantor. Seizing the
whole position would take more than the protocol lost.

---

## Repositories

| Repository | Stack | Responsibility |
|------------|-------|----------------|
| [`remitcollateral-contract`](https://github.com/RemitCollateral/remitcollateral-contract) | Rust · Soroban SDK 22 | Settlement layer — collateral custody, loan state, liquidation cranks |
| [`remitcollateral-backend`](https://github.com/RemitCollateral/remitcollateral-backend) | Node.js · TypeScript · Express 4 | Orchestration — API, reputation engine, off-ramp adapters, lifecycle sweep |
| [`remitcollateral-frontend`](https://github.com/RemitCollateral/remitcollateral-frontend) | Next.js 14 · React 18 · Tailwind | Guarantor dashboard — collateral, loans, beneficiaries, risk disclosure |
| `remitcollateral-docs` (this repo) | Markdown | Protocol documentation and integration guides |

There is no repository for the beneficiary. That is the point — they interact
with the system through the off-ramp partner's existing channel and SMS.

---

## How a loan works

**1. Collateral.** The guarantor deposits USDC into their own vault. Nothing is
pooled with anyone else's funds.

**2. Origination.** The guarantor opens a loan, priced at the off-ramp
partner's exchange rate, and signs it in their own wallet. The protocol computes
the required LTV from the beneficiary's reputation — 150% for a stranger, down to
a 110% floor for a well-established relationship — and locks that multiple of the
principal in the vault. Only then is the off-ramp partner instructed to disburse
local currency.

**3. Repayment.** The beneficiary repays in local currency through their normal
channel. The partner servicing the loan and an independent verifier co-sign
each repayment attestation, and collateral is released in proportion to principal
repaid, less a safety buffer held back until the loan closes.

**4. Closing.** The final attested installment returns all remaining collateral,
buffer included, and the beneficiary's reputation improves — which lowers the
LTV on their next loan.

**5. Default.** If an installment is missed, the loan enters a grace period. If
grace expires unpaid, liquidation forfeits collateral equal to the outstanding
balance and returns the rest. The missed installments lower the reputation
score, which raises the LTV required next time.

```
deposit ──▶ originate ──▶ [ repay ─▶ release ]* ──▶ close, all collateral returned
                 │                    ▲
                 │                    │ attested repayment
                 ▼                    │
             miss a payment ──▶ grace ┘
                                  │ grace expires
                                  ▼
                             default ──▶ forfeit outstanding, return the excess
```

---

## Protocol parameters

Every parameter below is configurable. The defaults are what an unconfigured
deployment runs.

| Parameter | Default | Meaning |
|-----------|---------|---------|
| Base LTV | `150%` | Collateral required of a beneficiary with no reputation |
| Minimum LTV | `110%` | Floor — no reputation score goes below this |
| LTV reduction | `0.004` per point | Reduction per point of composite reputation score |
| Safety buffer | `5%` | Collateral retained until the loan closes completely |
| Grace period | `7 days` (backend), `14 days` (testnet ledger) | Time after a missed installment before default. The ledger's is fixed at deployment, so set the backend's `GRACE_PERIOD_DAYS` to match |
| Remittance weight | `0.40` | Weight of remittance history in the composite score |
| Repayment weight | `0.60` | Weight of repayment history in the composite score |
| Minimum remittance history | `6 months` | History needed before remittances influence LTV |

The LTV curve is linear between the bounds:

```
required_ltv = max(min_ltv, base_ltv − score × reduction_factor)
```

A composite score of 0 pays the base rate of 150%. A perfect score of 100 earns
the 110% floor. The contracts express the same curve in basis points, so the
two implementations agree at every point — see [ARCHITECTURE.md](ARCHITECTURE.md#reputation-and-ltv).

---

## Running the stack locally

The three repositories are cloned side by side and run independently. The
frontend ships with a mock backend, so you can start at whichever layer you care
about.

### Prerequisites

| Tool | Version | Needed for |
|------|---------|-----------|
| Node.js | ≥ 22 (backend), ≥ 18.17 (frontend) | Backend and frontend |
| pnpm | 8.x | Frontend |
| Rust | latest stable | Contracts |
| `wasm32v1-none` target | — | Contracts |
| Stellar CLI | latest | Contract deployment |
| [Freighter](https://www.freighter.app/) | — | Wallet connection in `live` mode |

### Frontend only — no backend required

```bash
git clone https://github.com/RemitCollateral/remitcollateral-frontend
cd remitcollateral-frontend
pnpm install
cp .env.example .env.local     # ships with NEXT_PUBLIC_API_MODE=mock
pnpm dev
```

Open <http://localhost:3000>. A simulated wallet stands in for Freighter and a
seeded in-memory dataset drives every screen, with the real protocol maths —
reputation scoring, LTV adjustment, schedule generation, proportional collateral
release — running client-side.

### Backend

```bash
git clone https://github.com/RemitCollateral/remitcollateral-backend
cd remitcollateral-backend
npm install
cp .env.example .env
npm run dev                    # http://localhost:4000
```

The backend runs against in-memory stores and mock adapters by default, so an
empty `.env` boots a complete working API. Health check at
<http://localhost:4000/health>; the API is served under `/api/v1`.

To connect it to the testnet contracts, set the three contract IDs and the chain
settings listed in the [deployment section](ARCHITECTURE.md#deployment-and-wiring).
`GET /api/v1/chain` then reports `enabled: true`, and deposits, withdrawals and
new loans are signed in the guarantor's wallet.

To point the frontend at it, set in `.env.local`:

```env
NEXT_PUBLIC_API_MODE=live
NEXT_PUBLIC_API_URL=http://localhost:4000/api/v1
```

### Contracts

```bash
git clone https://github.com/RemitCollateral/remitcollateral-contract
cd remitcollateral-contract/contracts
rustup target add wasm32v1-none
cargo build --target wasm32v1-none --release
cargo test
```

Artifacts land in `contracts/target/wasm32v1-none/release/`:

```
rc_guarantor_vault.wasm
rc_loan_ledger.wasm
rc_liquidation_engine.wasm
```

The three contracts reference each other by address and **must be wired after
deployment**. The contract repository's `scripts/deploy.sh` deploys and wires
them in order; the [deployment sequence](ARCHITECTURE.md#deployment-and-wiring)
explains each step and lists the current testnet addresses.

---

## Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)** — layer-by-layer design, the contract
  interfaces, the trust boundaries, the data model, the loan state machine, and
  the current integration gaps between repositories.
- **[CONTRIBUTING.md](CONTRIBUTING.md)** — how to pick an issue, the development
  workflow across the three repositories, per-stack coding standards, branching
  and commit conventions, and the pull request checklist.
- **[ROADMAP.md](ROADMAP.md)** — what has shipped and what is left, ordered by
  dependency: finishing testnet, the first partner integration, production
  hardening, and the trust assumptions v1 accepts for now.
- **[SECURITY.md](SECURITY.md)** — how to report a vulnerability privately, what
  we treat as severe in a custody protocol, and the limitations already known.

## Status

The contracts are deployed on Stellar testnet with a 2-of-3 multisig admin and
a 48-hour timelock, and are covered by unit tests and a smoke test against the
live deployment. The backend connects to them: sign-in, deposits, withdrawals and
loan origination run on chain with the guarantor's wallet signature, and the
dashboard signs in Freighter.

Repayments and liquidation still run only in the backend's own records, the
off-ramp partner is a mock, state is held in memory, and the contracts have not
been audited, so this is not ready for real money. See
[Integration status](ARCHITECTURE.md#integration-status) for exactly what is
wired and what is not.

## License

MIT — see [LICENSE](LICENSE).
