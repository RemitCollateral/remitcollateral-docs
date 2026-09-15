# Roadmap

Where RemitCollateral is, and what stands between here and a protocol that can
hold real money.

**There are no dates here.** Items are ordered by dependency, not by calendar —
each phase mostly needs the one before it. Anything that looks like a commitment
to a date would be invented, and this document is more useful as an honest
account of what is actually left to do.

The authoritative record of what is currently wired versus mocked lives in
[ARCHITECTURE.md](ARCHITECTURE.md#integration-status). When an item here ships,
that section should shrink.

---

## Phase 0 — where we are

- ✅ Three Soroban contracts — `GuarantorVault`, `LoanLedger`,
  `LiquidationEngine` — with test suites and recorded ledger snapshots, deployed
  on Stellar testnet.
- ✅ Repayment attestations co-signed by the loan's own partner and an
  independent verifier.
- ✅ Constructors instead of a public initializer, one-time wiring, storage
  lifetimes that keep live loans from being archived, a 48-hour timelock on
  upgrades and settlement changes, and a 2-of-3 multisig admin.
- ✅ Wallet sign-in with verified SEP-53 signatures and sessions.
- ✅ A live chain client in the backend: deposits, withdrawals and loan
  origination signed in the guarantor's wallet and settled on testnet.
- ✅ Loans priced at the off-ramp partner's exchange rate.
- ✅ The full `/api/v1` surface: auth, guarantors, vaults, beneficiaries, loans,
  repayments, remittances, audit, admin.
- ✅ The reputation engine, the loan lifecycle sweep, and the audit trail.
- ✅ The guarantor dashboard, signing in Freighter, against either the mock or the
  live API.
- ✅ Protocol documentation, license, and security policy.

What this does **not** yet include: repayments or liquidation on chain, a single
real disbursement, or any state that survives a restart.

---

## Phase 1 — reconcile the repositories

The three repositories were built against a shared design but had drifted at
four seams. Three are closed; the last is documented under
[known cross-repository mismatches](ARCHITECTURE.md#known-cross-repository-mismatches).

- [x] **Align the auth header.** The backend verifies a signed message and
  issues a session token, which the frontend sends as `Authorization: Bearer`.
  The `x-wallet-address` header is no longer accepted.
- [x] **Align the default API port.** Both now default to `4000`.
- [ ] **Reconcile the grace period.** The testnet ledger uses 14 days; the
  backend and frontend default to 7. A backend connected to the chain must set
  `GRACE_PERIOD_DAYS` to match, and the defaults should follow. This one matters
  most: the chain and the lifecycle sweep would otherwise disagree about the
  moment a loan defaults, which is a disagreement about money.
- [x] **Add `GET /beneficiaries`.** The backend lists a guarantor's
  beneficiaries, including ones with no loan yet, and the frontend uses it.

---

## Phase 2 — testnet

The point of this phase is one real loan, originated and repaid end to end,
against deployed contracts on Stellar testnet.

- [x] **Live chain client.** `src/chain` in the backend reads vault and loan
  state from the deployed contracts, and deposits, withdrawals and originations
  are prepared by the backend, signed in the guarantor's wallet and submitted. It
  is a separate client rather than an implementation of `ContractGateway`,
  because a guarantor's transaction has to be signed by the guarantor, which that
  interface could not express.
- [x] **Deploy and wire the contracts** on testnet. `scripts/deploy.sh` does it
  in order and `scripts/smoke-testnet.sh` checks the result, including the
  multisig and the timelock.
- [x] **Wallet authentication.** Implemented as a SEP-53 signed message rather
  than SEP-10: the dashboard signs a challenge naming the service, the wallet, a
  one-time nonce and an expiry. The header stub is gone.
- [x] **Exchange rates.** Loans are priced at the off-ramp partner's rate, which
  is stored on the loan. The mock partner's rates are fixed; a real partner
  supplies live ones.
- [ ] **Repayments on chain.** Route partner attestations through the chain
  client as co-signed `attest_repayment` calls, with an API for the partner to
  sign its half.
- [ ] **Liquidation on chain.** Drive `flag_overdue` and `liquidate` from the
  lifecycle sweep, so the chain and the backend's records advance together.
- [ ] **Cancel an undisbursed loan.** A ledger call, co-signed by the partner and
  a verifier and allowed only before any repayment, that releases the collateral
  of a loan whose payout failed.
- [ ] **Persistent storage.** PostgreSQL behind the stores. The `pg` dependency
  is already present and the stores are structured for the port.
- [ ] **Reconcile chain and backend state.** Both track loan status and
  collateral. Vault figures are already read from chain; loans need the same,
  with the chain authoritative on disagreement and a check that detects drift.

---

## Phase 3 — first partner integration

Everything above can be done without leaving the building. This phase cannot: it
needs a licensed off-ramp partner willing to disburse and collect.

- [ ] **A real `OffRampAdapter`.** One implementation of the existing interface
  per partner. `MockOffRampAdapter` defines the shape.
- [ ] **Attestation checks.** On chain, the ledger now requires the partner's
  own signature. The backend's `verifyAttestation` still has to check each
  partner report properly before the verifier co-signs, because the verifier's
  signature is only as trustworthy as that check.
- [ ] **Partner onboarding and key management.** Registering a partner address
  on-chain, issuing and rotating API credentials, and exercising revocation.
- [ ] **Remittance history ingest** from the partner's records, which is what
  makes the cold-start reputation signal real rather than seeded.
- [ ] **Beneficiary notifications.** The beneficiary has no wallet and no
  dashboard, so SMS is the only channel through which they learn a schedule
  exists and that a payment is due.
- [ ] **Disbursement failure handling** against a real partner — timeouts,
  partial failures, and reconciling a disbursement whose outcome is unknown.
  The origination rollback assumes a clean throw; reality is messier.

---

## Phase 4 — production hardening

Nothing in this phase adds a feature. All of it is what separates a working
prototype from something that can hold a stranger's collateral.

- [ ] **A third-party security audit of the contracts.** Non-negotiable before
  mainnet. The [security policy](SECURITY.md) states plainly that the contracts
  are unaudited.
- [x] **TTL management.** Every call that uses an entry extends it to about 120
  days, and the liquidation cranks double as a keep-alive for idle loans.
  Anything untouched for longer is archived, not lost, and can be restored.
- [ ] **Multi-instance operation.** The lifecycle sweep runs in-process and
  assumes a single backend. Running several would sweep several times per tick,
  so this needs an external scheduler or a distributed lock.
- [x] **Admin key management.** The admin is a 2-of-3 multisig, and upgrades and
  settlement changes wait behind a 48-hour timelock.
- [ ] **Oracle and verifier key management.** Both are still single keys held by
  the backend. They need hardware-backed signing and a documented rotation
  procedure.
- [ ] **Contract events** for attestations, scheduled admin actions and defaults,
  so the timelock can be monitored rather than polled.
- [ ] **Monitoring and alerting.** Sweep failures, gateway errors, and
  disbursement failures are currently visible only in the audit log. A default
  that goes unrecorded is a silent loss.
- [ ] **Load and failure testing**, particularly around the sweep with a large
  book of open loans.

---

## Phase 5 — protocol evolution

These change what the protocol *is*, not just how well it runs. Each reduces a
trust assumption that v1 accepts deliberately. They are listed roughly by how
much trust they remove.

**Independent corroboration.** Today the loan's partner and one verifier
co-sign each repayment. The verifier is run by the platform, so a partner and the
platform acting together could still credit a repayment that never happened.
Holding released collateral for a dispute window before it becomes withdrawable,
or requiring a second independent attestation, removes the largest remaining
point of trust.

**On-chain reputation derivation.** The chain currently trusts an oracle's
published score, and that score sets the LTV. Moving part of the derivation
on-chain makes the LTV reproducible without trusting the oracle — the most
valuable of these items, and the hardest, since the scoring inputs are
themselves off-chain.

**DEX-based liquidation.** Forfeited collateral currently transfers to a
platform-controlled settlement address. Settling through a swap on the Stellar
DEX removes the platform as a custodian of seized funds.

**Per-loan grace configuration.** The grace period is one global value. Letting
it vary with loan size or beneficiary reputation would let a trusted borrower
absorb a late payment without the same consequence as a first-time one.

**Guarantor-side risk controls.** Exposure limits per beneficiary, and a way for
a guarantor to decline further origination against their vault without
withdrawing.

---

## Explicitly not planned

Stating these saves the argument later.

- **A beneficiary wallet or app.** The beneficiary never touching crypto is the
  premise of the protocol, not a limitation of the current version.
- **Pooled collateral.** Vault isolation is a core invariant. A shared pool would
  socialise losses across unrelated guarantors and is not on the table.
- **Accepting repayment claims from the beneficiary or guarantor.** Repayments
  come from the loan's partner, co-signed by a verifier, or they do not count.
- **Putting beneficiary identity on-chain.** The 32-byte handle exists precisely
  so personal data never reaches a public ledger.

---

## Contributing to the roadmap

If you want to work on something here, open an issue in
`remitcollateral-docs` first so the design can be agreed across the layers it
touches — most of these items span more than one repository. See
[cross-repository changes](CONTRIBUTING.md#cross-repository-changes) for how to
sequence the work so `main` is never broken in either place.
