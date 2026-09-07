# Contributing to RemitCollateral

Thank you for your interest in contributing. Bug reports, feature work,
documentation, and protocol review are all welcome.

RemitCollateral spans four repositories. This guide covers the workflow common to
all of them, then the standards specific to each stack.

---

## Contents

- [Where does my change go?](#where-does-my-change-go)
- [Getting started](#getting-started)
- [Development workflows](#development-workflows)
  - [Smart contracts (Rust / Soroban)](#smart-contracts-rust--soroban)
  - [Backend (TypeScript / Express)](#backend-typescript--express)
  - [Frontend (Next.js / React)](#frontend-nextjs--react)
- [Coding standards](#coding-standards)
- [Protocol invariants](#protocol-invariants)
- [Branching and commits](#branching-and-commits)
- [Submitting a pull request](#submitting-a-pull-request)
- [Cross-repository changes](#cross-repository-changes)
- [Code of conduct](#code-of-conduct)

---

## Where does my change go?

| If your change is about… | Repository |
|--------------------------|------------|
| Collateral custody, loan state, liquidation, on-chain authorization | `remitcollateral-contract` |
| API endpoints, reputation scoring, off-ramp partners, the lifecycle sweep | `remitcollateral-backend` |
| Guarantor screens, wallet connection, the mock backend | `remitcollateral-frontend` |
| Protocol design, integration guides, this document | `remitcollateral-docs` |

If a change touches more than one, read
[Cross-repository changes](#cross-repository-changes) before you start.

## Getting started

1. **Fork** the repository you are changing.
2. **Clone** your fork:
   ```bash
   git clone https://github.com/<your-username>/<repository>.git
   cd <repository>
   ```
3. **Track upstream** so you can pull in changes:
   ```bash
   git remote add upstream https://github.com/RemitCollateral/<repository>.git
   ```
4. **Branch.** Never work on `main`:
   ```bash
   git checkout -b feature/your-feature-name
   ```

---

## Development workflows

### Smart contracts (Rust / Soroban)

Source lives in `contracts/`, a Cargo workspace of three crates:
`guarantor_vault`, `loan_ledger`, `liquidation_engine`.

**Prerequisites:** Rust (latest stable), the `wasm32-unknown-unknown` target, and
the Stellar CLI.

```bash
rustup target add wasm32-unknown-unknown
cargo install --locked stellar-cli
```

**Build:**

```bash
cd contracts
cargo build --target wasm32-unknown-unknown --release
```

**Test:**

```bash
cd contracts
cargo test                          # all suites
cargo test -p rc-loan-ledger        # a single crate
```

The workspace records test snapshots under each crate's `test_snapshots/`. A diff
there means ledger behaviour changed — review it deliberately and include the
reasoning in your PR rather than regenerating it silently.

**Format and lint** before every commit:

```bash
cd contracts
cargo fmt --all
cargo clippy --all-targets --all-features -- -D warnings
```

Clippy warnings are errors in CI. Do not silence one with `#[allow]` without a
comment explaining why the lint does not apply.

### Backend (TypeScript / Express)

```bash
npm install
cp .env.example .env
npm run dev        # ts-node-dev, hot reload, http://localhost:4000
```

An empty `.env` boots a fully working API: every protocol parameter falls back to
its documented default, and the off-ramp adapter and contract gateway are mocks.

```bash
npm run build      # tsc
npm start          # node dist/index.js
```

Verify `npm run build` passes before opening a PR — the dev server transpiles
without type-checking, so a type error can hide until the build runs.

### Frontend (Next.js / React)

The frontend uses **pnpm**. Do not commit an `npm` or `yarn` lockfile.

```bash
pnpm install
cp .env.example .env.local
pnpm dev           # http://localhost:3000
```

| Command | Purpose |
|---------|---------|
| `pnpm dev` | Development server |
| `pnpm build` | Production build |
| `pnpm start` | Serve the production build |
| `pnpm lint` | ESLint via `next lint` |
| `pnpm typecheck` | `tsc --noEmit` |

Both `pnpm lint` and `pnpm typecheck` must pass.

Default mode is `mock`, so the app runs with no backend. To work against a local
backend:

```env
NEXT_PUBLIC_API_MODE=live
NEXT_PUBLIC_API_URL=http://localhost:4000/api/v1
```

---

## Coding standards

### Soroban contracts

- **Authorize explicitly.** Every state-changing function calls `require_auth()`
  on the acting address, and then checks that address against the stored role.
  Authentication and authorization are two separate steps; do not conflate them.
- **Respect the caller matrix.** Only the LoanLedger locks collateral; only the
  LiquidationEngine forfeits it. If a new function needs vault access, extend the
  matrix in [ARCHITECTURE.md](ARCHITECTURE.md#guarantorvault) deliberately rather
  than widening an existing guard.
- **State before transfer.** Update internal balances before invoking token
  transfers or external contracts.
- **Typed errors.** Add a variant to the crate's `Error` enum and use
  `panic_with_error!`. Never panic with a bare message, and never reuse an
  existing variant for a new failure mode — error codes are part of the contract's
  public surface, so append rather than renumber.
- **Basis points, not floats.** There is no floating point in `no_std`. Ratios are
  `u32` basis points against the `BPS` constant.
- **Watch integer division.** `principal_usd / installment_count` truncates. When
  you introduce a new division, state where the remainder goes.
- **Test the negative case.** A test that proves an unauthorized caller is
  rejected is worth more than one that proves the happy path works.

### Backend

- **Strict typing.** No `any` in new code. Domain entities and DTOs belong in
  `src/types`.
- **Services hold logic; routes hold plumbing.** A route validates input, calls a
  service, and shapes the response. Business rules do not live in `src/routes`.
- **Go through the interfaces.** All chain access goes through `ContractGateway`
  and all partner access through `OffRampAdapter`. Never import the Stellar SDK or
  call a partner API directly from a service — that is what keeps the mock and
  live implementations interchangeable.
- **Parameters are configurable.** Protocol constants are read from `config`, with
  a documented default. Do not inline a magic number that the architecture
  describes as tunable.
- **Audit the state changes.** Every meaningful transition calls `logAuditEvent`
  with an `eventType`, an `action`, and enough detail to reconstruct what
  happened.
- **Unwind on failure.** If an operation touches the chain, local state, and a
  partner, the failure path must unwind every step that already succeeded. See
  the origination rollback in `loan.service.ts` for the shape of this.
- **Round money at the boundary.** Monetary values are rounded to 2 decimal places
  where they are stored, not opportunistically mid-calculation.

### Frontend

- **Strict TypeScript.** No `any`. Domain types in `lib/types.ts` mirror the
  backend's data model.
- **One API surface.** Screens import the single `api` object. If you add an
  endpoint, add it to the `RemitCollateralApi` interface and implement it in
  **both** `lib/api/http.ts` and `lib/api/mock/` — an unimplemented mock breaks
  the no-backend workflow for everyone.
- **Keep the mock honest.** The mock implements real protocol maths, not
  hardcoded responses. If you change a formula, change it in
  `lib/api/mock/protocol.ts` too, and make sure it still agrees with the backend.
- **Tailwind, no raw colors.** Use theme utilities and CSS variables. Maintain
  responsiveness across mobile, tablet, and desktop.
- **Semantic and accessible markup.** Real landmark elements over nested `<div>`s;
  labelled form controls; keyboard-reachable interactive elements.
- **Never hide the risk.** Screens that commit a guarantor's collateral state the
  amount at stake before the action, not after.

---

## Protocol invariants

Some rules are load-bearing across all three implementations. A change that
breaks one is a protocol change, not a bug fix — raise it as an issue in
`remitcollateral-docs` first.

1. **Collateral is never pooled.** One vault per guarantor, always.
2. **The beneficiary never appears on-chain as an address.** They are a
   `BytesN<32>` handle. No phone number, name, or KYC reference reaches the
   ledger.
3. **Only a registered partner can attest a repayment.** Never the beneficiary,
   never the guarantor.
4. **Partner identity comes from the authenticated credential**, never from a
   request body.
5. **Self-declared remittances carry zero scoring weight.** Only
   `partner_reported` records influence the score, and therefore the LTV.
6. **Loan closure is decided by principal repaid**, never by the number of
   attestations received.
7. **Liquidation forfeits the outstanding balance only.** The excess collateral
   returns to the guarantor.
8. **The liquidation cranks stay permissionless.** Their behaviour is a function
   of loan state and the ledger clock alone.
9. **Reputation is recomputed from records**, not stored as a running penalty, so
   any score can be re-derived from the underlying history.

If a change requires the backend's maths and the contracts' maths to agree —
LTV, collateral release, grace period — update both, and say so in the PR.

---

## Branching and commits

### Branch names

- `feature/` — new capability (`feature/multi-partner-attestation`)
- `fix/` — bug fix (`fix/collateral-release-rounding`)
- `docs/` — documentation (`docs/deployment-wiring-order`)
- `refactor/` — restructuring with no behaviour change (`refactor/extract-gateway`)
- `chore/` — tooling, dependencies, CI

### Commit messages

```
<type>(<scope>): <imperative summary>
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`.
Scopes: `contracts`, `vault`, `ledger`, `engine`, `backend`, `frontend`, `api`,
`docs`.

Examples:

```
feat(ledger): reject attestations that would overpay the principal
fix(backend): unwind the on-chain lock when disbursement fails
docs: record the grace period mismatch between chain and sweep
refactor(frontend): move quote derivation out of the loan form
```

Explain **why** in the body when the reason is not obvious from the diff.
Security-relevant changes should always carry a body.

---

## Submitting a pull request

1. **Sync with upstream:**
   ```bash
   git fetch upstream
   git rebase upstream/main
   ```

2. **Run the checks for the repository you touched:**

   | Repository | Checks |
   |------------|--------|
   | Contracts | `cargo fmt --all`, `cargo clippy --all-targets --all-features -- -D warnings`, `cargo test` |
   | Backend | `npm run build` |
   | Frontend | `pnpm lint`, `pnpm typecheck`, `pnpm build` |

3. **Open the PR** and fill in:
   - **Summary** — what changed and why. If it alters protocol behaviour, say so
     in the first line.
   - **Testing** — how you verified it. "Ran the tests" is not testing; name the
     cases and the scenarios you exercised manually.
   - **Protocol impact** — does this change a formula, a parameter default, an
     authorization rule, or an API shape? Does another repository need a matching
     change?
   - **Related issues** — `Closes #12`.

4. **Address review feedback.** Push follow-up commits rather than force-pushing
   mid-review, so reviewers can see what changed.

### What reviewers look for

- Authorization checks on every new state-changing contract function.
- Failure paths that unwind cleanly, especially where chain, local state, and a
  partner are all involved.
- Mock and live implementations kept in step.
- No new magic number that the architecture describes as configurable.
- Documentation updated when behaviour changed — including
  [ARCHITECTURE.md](ARCHITECTURE.md) when an invariant or an integration gap
  moves.

---

## Cross-repository changes

Some work spans repositories — adding an endpoint, changing a shared formula,
altering the wire format. Sequence it so `main` is never broken in either place:

1. **Open an issue in `remitcollateral-docs` first** describing the change across
   all affected layers, and get agreement on the shape before writing code.
2. **Contracts first**, if involved. They are the slowest to change and the
   hardest to reverse once deployed.
3. **Backend second**, additively — add the new field or endpoint alongside the
   old one rather than replacing it.
4. **Frontend third**, consuming the new shape. Update the mock in the same PR.
5. **Remove the old path last**, once nothing consumes it.
6. **Update the docs** — particularly the
   [integration status](ARCHITECTURE.md#integration-status) section, which is
   meant to be an accurate account of what is wired and what is mocked. If you
   close one of the listed gaps, delete that row.

Link the PRs to one another so they can be reviewed together.

---

## Code of conduct

We are committed to a welcoming, collaborative, and inclusive environment. By
participating you agree to:

- Be respectful, constructive, and empathetic toward other contributors.
- Focus on what is best for the community and the project.
- Accept constructive criticism gracefully, and offer it kindly.

Report unacceptable behaviour to the maintainers through a private channel.

---

## Security

Do not open a public issue for a vulnerability, especially one affecting
collateral custody, authorization, or the attestation path. Contact the
maintainers privately and give them time to respond before any disclosure.
