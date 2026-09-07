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
their phone number and the off-ramp partner's KYC reference, so no personally
identifying data reaches the ledger. They receive naira, cedis or shillings, and
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

**2. Origination.** The guarantor opens a loan. The protocol computes the
required LTV from the beneficiary's reputation — 150% for a stranger, down to a
110% floor for a well-established relationship — and locks that multiple of the
principal in the vault. Only then is the off-ramp partner instructed to disburse
local currency. If the disbursement fails, the lock is unwound.

**3. Repayment.** The beneficiary repays in local currency through their normal
channel. A registered off-ramp partner attests to each repayment, and collateral
is released in proportion to principal repaid, less a safety buffer held back
until the loan closes.

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
| Grace period | `7 days` | Time after a missed installment before default |
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
| Node.js | ≥ 18 (backend), ≥ 20 recommended | Backend and frontend |
| pnpm | 8.x | Frontend |
| Rust | latest stable | Contracts |
| `wasm32-unknown-unknown` target | — | Contracts |
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

To point the frontend at it, set in `.env.local`:

```env
NEXT_PUBLIC_API_MODE=live
NEXT_PUBLIC_API_URL=http://localhost:4000/api/v1
```

> The frontend's built-in default is port `3001`, which does not match the
> backend's default port of `4000`. Set `NEXT_PUBLIC_API_URL` explicitly.

### Contracts

```bash
git clone https://github.com/RemitCollateral/remitcollateral-contract
cd remitcollateral-contract/contracts
rustup target add wasm32-unknown-unknown
cargo build --target wasm32-unknown-unknown --release
cargo test
```

Artifacts land in `contracts/target/wasm32-unknown-unknown/release/`:

```
rc_guarantor_vault.wasm
rc_loan_ledger.wasm
rc_liquidation_engine.wasm
```

The three contracts reference each other by address and **must be wired after
deployment** — see the [deployment sequence](ARCHITECTURE.md#deployment-and-wiring)
in the architecture document. Skipping a wiring step leaves the protocol in a
state where origination or liquidation silently cannot proceed.

---

## Documentation

- **[ARCHITECTURE.md](ARCHITECTURE.md)** — layer-by-layer design, the contract
  interfaces, the trust boundaries, the data model, the loan state machine, and
  the current integration gaps between repositories.
- **[CONTRIBUTING.md](CONTRIBUTING.md)** — development workflow across the three
  repositories, per-stack coding standards, branching and commit conventions,
  and the pull request checklist.
- **[SECURITY.md](SECURITY.md)** — how to report a vulnerability privately, what
  we treat as severe in a custody protocol, and the limitations already known.

## Status

This is v1. The contracts are complete and tested; the backend runs on in-memory
stores with a mock contract gateway and a mock off-ramp adapter; the frontend
runs against either the mock or the live API. The seams where the live
implementations plug in are interfaces, not rewrites — see
[Integration status](ARCHITECTURE.md#integration-status) for exactly what is
wired and what is not.

## License

MIT — see [LICENSE](LICENSE).
