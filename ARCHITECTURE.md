# Architecture: StellarHomes Protocol on Stellar

## Core Principle: Separate Compliance & Milestones from Funding

The architecture splits into two layers: a **Compliance & Verification Layer** that establishes land title validity, property valuation, and construction milestones off-chain, and a **Settlement Layer** (Stellar/Soroban) that handles liquidity pool deposits, mortgage issuance, milestone-gated escrow disbursements, and secondary market trading.
---

## 1. Compliance & Verification Layer (Off-Chain)

**Title verification**: The trustee submits property title documents and surveys. The oracle queries land registries (MLHUD in Nigeria or the Lands Commission in Ghana) to verify ownership and check for encumbrances.

**Property valuation**: A licensed surveyor provides an official property valuation report. Once verified by the oracle, this value is anchored on-chain to determine the maximum borrowing capacity (LTV).

**Identity verification (KYC)**: Diaspora members and mortgage investors must pass KYC/AML screening via an integrated compliance provider (complying with local regulations in their country of residence) before they are authorized to hold or trade tokens.

**Output**: A verified property record in the `PropertyRegistry` contract, enabling the issuance of property-specific equity tokens (`PROP`) to authorized wallets.

---

## 2. Settlement Layer (Stellar / Soroban)

### PropertyRegistry — Asset Tokenization

A Soroban smart contract that manages the property lifecycle from submission to tokenization. It records title hashes, assigns trustees, writes oracle-verified valuations, and mints divisible `PROP-[ID]` tokens representing fractional equity in the property.

### Token Model (LP & Property Tokens)

All investment instruments are issued as Stellar custom assets via the Stellar Asset Contract (SAC), enforcing compliance:
- **`PROP-[ID]`**: Fractional equity in a specific property. Issued with `AUTH_REQUIRED` (only KYC-cleared wallets can hold/transfer) and `CLAWBACK_ENABLED` (for regulatory recovery or court-ordered disputes).
- **`MORT-[ID]`**: Shares of a specific mortgage loan, earning pro-rata interest repayments.
- **`POOL-HC`**: Shares of the global mortgage liquidity pool.

### MortgagePool — Liquidity & Loan Governance

Manages the global mortgage liquidity pool (`POOL-HC`), loan issuance, repayments, and default liquidation:
- **Deposit**: Investors deposit USDC to receive `POOL-HC` shares.
- **Collateralization**: Diaspora members lock `PROP-[ID]` tokens as collateral (maximum 70% LTV).
- **Issuance**: Transfers USDC directly to the `BuildEscrow` contract (never to the borrower).
- **Repayment**: Splits payments into principal (returned to the pool) and interest (distributed to `MORT-[ID]` holders).
- **Liquidation**: If LTV exceeds 80% and a margin call is missed, `PROP` tokens are auctioned on the Stellar DEX.

### BuildEscrow — Milestone-Gated Escrow

Holds mortgage disbursements and releases funds in tranches directly to pre-vetted builders and material suppliers. Funds are never touched by the borrower:
1. **Foundation (20%)**: Released on trustee sign-off and photo attestation.
2. **Structural Walls (25%)**: Released on trustee sign-off and licensed inspector report.
3. **Roofing (20%)**: Released on trustee sign-off and photo attestation.
4. **Finishing (25%)**: Released on trustee sign-off and inspector report.
5. **Handover (10%)**: Released on final diaspora member sign-off.

### Governance / Multisig

The escrow and pool operations are protected by Stellar's native multi-signature capabilities. Critical state transitions and emergency halts (`halt()`) require co-signatures from the platform compliance officers and trustees.

### Secondary Market

Allows LPs and diaspora members to trade `PROP` and `MORT` tokens on the Stellar DEX. The Stellar Asset Contract's `AUTH_REQUIRED` flag ensures that the Stellar DEX only matches trades between wallets that have been explicitly KYC-cleared.

### Stablecoin Settlement

All deposits, mortgage loans, milestone payments, repayments, and secondary market trades are settled in USDC on Stellar, providing sub-cent fees and 3–5 second finality.

---

## 3. Evidence Storage

IPFS stores immutable copies of land titles, surveyor valuations, building inspections, and milestone photo attestations. The resulting IPFS content hashes are written to the `BuildEscrow` and `PropertyRegistry` contracts to provide a permanent, auditable ledger of the build progress.

---

## 4. Frontend & Wallet Integration

- **StellarHomes dApp (Web & Mobile)**: Diaspora members connect their Stellar wallets (e.g., Freighter) to deposit USDC, track construction progress via milestone photos, and manage their equity.
- **Trustee Portal**: Local agents submit title documents, upload milestone evidence, and sign off on completed stages.
- **Investor Dashboard**: Mortgage investors monitor pool performance, track yields, and manage their `POOL-HC` and `MORT` tokens.

---

## 5. Backend / Orchestration

A Node.js/TypeScript backend coordinates the off-chain workflows:
- **Stellar SDK** (js-stellar-sdk) for monitoring ledger events, verifying transactions, and invoking Soroban contracts.
- **PostgreSQL** for storing off-chain metadata, user profiles, and tracking construction milestones.
- **Smile ID / KYC Provider** for identity verification and AML compliance.
- **Stellar Anchors (Nigeria & Ghana)** for handling NGN and GHS on/off-ramps via local payment rails (e.g., Flutterwave, Paystack) to pay local builders and suppliers.
- **IPFS Pinning Service (Pinata)** for storing property documents and milestone media.

---

## End-to-End Flow Summary

1. **Property Submission**: A local trustee submits title deeds and surveys; the oracle verifies them with MLHUD/Lands Commission and records the valuation.
2. **Tokenization**: `PROP-[ID]` tokens are minted with `AUTH_REQUIRED` and `CLAWBACK` flags enabled.
3. **Mortgage Funding**: The diaspora member deposits `PROP-[ID]` tokens as collateral into the `MortgagePool`. The pool issues a USDC loan (up to 70% LTV) using investor liquidity, transferring the funds directly to the `BuildEscrow`.
4. **Milestone Construction**: As the builder completes each stage, the trustee uploads photo/inspector evidence to IPFS. The oracle verifies the evidence, and the escrow contract releases the next USDC tranche to the builder's wallet.
5. **Repayment**: The diaspora member makes monthly USDC repayments. The principal is returned to the liquidity pool, and the interest is distributed to `MORT-[ID]` holders.
6. **Completion**: Once the loan is fully repaid, the collateralized `PROP-[ID]` tokens are unlocked and returned to the diaspora member.
