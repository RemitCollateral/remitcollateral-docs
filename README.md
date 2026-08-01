# StellarHomes
A decentralized, compliant, and automated real estate tokenization and mortgage protocol built on Stellar.

The StellarHomes Protocol streamlines the lifecycle of real estate financing and development for diaspora communities—from property verification and tokenization to mortgage liquidity pools, milestone-based escrow disbursements, and compliant secondary market trading. By utilizing Soroban smart contracts, USDC escrows, oracle-verified land titles, and compliant secondary trading, it replaces slow, paper-based real estate transactions with a transparent, on-chain alternative.

It's the bridge between diaspora home financing and blockchain efficiency.

## Motivation
Diaspora communities face high operational overhead, lack of transparency, and risk of fraud when trying to finance and build homes in their home countries. Traditional mortgages are difficult to secure internationally, and funds sent to individuals are often mismanaged.

StellarHomes makes international property financing secure, transparent, and liquid:

*   **Automate Compliance:** Enforce investor and borrower whitelisting on-chain. Only KYC-cleared wallets can hold or trade property equity (`PROP`) tokens.
*   **Direct-to-Builder Funding:** Escrow mortgage disbursements securely and release funds directly to verified local builders and suppliers, ensuring funds never touch the borrower directly.
*   **Govern Disbursements:** Protect capital by requiring trustee sign-offs and licensed surveyor reports on construction milestones before releasing funds.
*   **Inject Secondary Liquidity:** Enable property owners and mortgage investors to trade their tokenized equity (`PROP` and `MORT` tokens) peer-to-peer via atomic swaps on the Stellar DEX.

## Features
*   **Property Registry Contract** — Manages property verification, oracle-verified valuations, and `PROP` token minting/clawbacks via the Stellar Asset Contract (SAC).
*   **Mortgage Pool Contract** — Manages the global mortgage liquidity pool (`POOL-HC`), collateral locking (maximum 70% LTV), interest repayment distribution, and default liquidations.
*   **Build Escrow Contract** — Milestone-based escrow holding construction funds and releasing them in tranches directly to pre-vetted builders and suppliers.
*   **Oracle Integration** — Off-chain data feeds that verify land titles with registries (MLHUD/Lands Commission) and write property valuations and milestone approvals on-chain.
*   **Secondary Market Integration** — Facilitates atomic, compliant trading of `PROP` and `MORT` tokens on the Stellar DEX, ensuring all participants are KYC-cleared.
*   **Stablecoin Settlement** — All deposits, mortgage loans, milestone payments, and secondary trades settle in USDC on Stellar with sub-cent fees and 3–5 second finality.

## Stack
*   **Frontend:** Next.js, TypeScript, TailwindCSS
*   **Wallet:** Freighter + `@stellar/freighter-api`
*   **Smart Contracts:** Rust, Soroban SDK v22
*   **Backend:** Node.js, Express, TypeScript, Stellar SDK (`@stellar/stellar-sdk` v13)
*   **Database:** PostgreSQL (for off-chain metadata, user profiles, and milestones)

## Running it locally

You can run and build the entire full-stack application (smart contracts, frontend, and backend) from the root directory of the project.

### Prerequisites
*   Node.js ≥ 20.0.0
*   Rust (latest stable) + `wasm32-unknown-unknown` target
*   Stellar CLI — `cargo install stellar-cli`
*   [Freighter Wallet](https://www.freighter.app/) browser extension

### 1. Setup Frontend & Backend
Install dependencies for both folders:
```bash
npm --prefix frontend install
npm --prefix backend install
```

Configure `backend/.env` (use `backend/.env.example` as a template):
```env
PORT=4000
STELLAR_NETWORK=testnet
STELLAR_RPC_URL=https://soroban-testnet.stellar.org
PROPERTY_REGISTRY_CONTRACT_ID=your_deployed_property_registry_contract_id
MORTGAGE_POOL_CONTRACT_ID=your_deployed_mortgage_pool_contract_id
BUILD_ESCROW_CONTRACT_ID=your_deployed_build_escrow_contract_id
```

### 2. Run the Application
Start the development servers from the root directory:
```bash
# Start the Next.js frontend (http://localhost:3000)
npm run frontend:dev

# Start the Express backend (http://localhost:4000)
npm run backend:dev
```

### 3. Build & Test Contracts
To build and test the Soroban smart contracts:
```bash
cd contracts
cargo build --target wasm32-unknown-unknown --release
cargo test
```

## How the StellarHomes Protocol Works
The protocol coordinates the entire financing and construction lifecycle on-chain through six phases:

1.  **Property Submission & Verification**: A trustee submits title documents; the oracle verifies them with local land registries and records the valuation.
2.  **Tokenization**: `PROP` tokens representing fractional equity are minted with compliance flags (`AUTH_REQUIRED` and `CLAWBACK_ENABLED`) enabled.
3.  **Collateral & Mortgage Funding**: The diaspora member locks `PROP` tokens as collateral in the `MortgagePool` to secure a USDC mortgage loan (up to 70% LTV).
4.  **Escrow Funding**: The mortgage loan is drawn down from investor liquidity and deposited directly into the `BuildEscrow` contract.
5.  **Milestone-Gated Disbursements**: As construction progresses, the trustee uploads milestone evidence to IPFS. The oracle verifies the progress, and the escrow contract releases the next USDC tranche directly to the builder.
6.  **Repayment & Exit**: The borrower makes monthly repayments in USDC, which are distributed to mortgage investors. Once fully repaid, the `PROP` collateral is unlocked.

## Roadmap
*   **Automated Land Registry Oracles**: Integrate APIs directly with West African land registries (MLHUD, Lands Commission) for instant title validation.
*   **Yield-Bearing Escrows**: Integrate idle escrow funds with Soroban-based lending protocols (e.g. Blend) to earn yield during construction phases.
*   **Decentralized Surveyor Network**: Transition from a single oracle to a decentralized network of licensed surveyors staking tokens to verify property valuations.
*   **Historical Build Indexer**: Set up a custom indexer to track and display construction progress milestones, photos, and historical payouts.

## Documentation
*   [Architecture](ARCHITECTURE.md): Core design principles, contract interactions, and system architecture.
*   [Contributing Guide](CONTRIBUTING.md): Code formatting, branching style, and pull request guidelines.

## License
MIT
