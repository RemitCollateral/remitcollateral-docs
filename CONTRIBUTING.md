# Contributing to StellarHomes

Thank you for your interest in contributing to **StellarHomes**! We welcome contributions of all kinds, including bug reports, feature requests, documentation improvements, frontend enhancements, and smart contract optimizations.

This guide outlines our development workflow, coding standards, and step-by-step instructions to ensure a smooth contribution process.
---

## 🗺️ Table of Contents

- [Code of Conduct](#-code-of-conduct)
- [Getting Started](#-getting-started)
- [Development Workflows](#-development-workflows)
  - [Smart Contract Development (Rust/Soroban)](#smart-contract-development-rustsoroban)
  - [Frontend Web Application (Next.js/React/TypeScript)](#frontend-web-application-nextjsreacttypescript)
- [Coding Standards & Best Practices](#-coding-standards--best-practices)
  - [Soroban Smart Contracts](#soroban-smart-contracts)
  - [Frontend client (React & TypeScript)](#frontend-client-react--typescript)
- [Git Branching & Commit Guidelines](#-git-branching--commit-guidelines)
  - [Branch Naming Convention](#branch-naming-convention)
  - [Commit Message Format](#commit-message-format)
- [Submitting a Pull Request](#-submitting-a-pull-request)

---

## 🤝 Code of Conduct

We are committed to fostering a welcoming, collaborative, and inclusive environment. By participating in this project, you agree to:
- Be respectful, constructive, and empathetic to other contributors.
- Focus on what is best for the community and the project.
- Accept constructive criticism gracefully.

---

## 🚀 Getting Started

To get started, follow these steps:

1. **Fork the Repository**: Create a personal copy of the repository on GitHub.
2. **Clone the Fork**: Clone your fork to your local machine:
   ```bash
   git clone https://github.com/<your-username>/StellarHomes.git
   cd StellarHomes
   ```
3. **Set Up Upstream Remote**: Track the original repository to fetch the latest updates:
   ```bash
   git remote add upstream https://github.com/NeonsLabs/Stellar-Homes.git
   ```
4. **Create a Feature Branch**: Never work directly on `main`. Create a descriptive branch (see [Git Branching Guidelines](#-git-branching--commit-guidelines)):
   ```bash
   git checkout -b feature/your-feature-name
   ```

---

## 🛠️ Development Workflows

StellarHomes is split into two primary environments: the **Soroban smart contracts** and the **Next.js web application**.

### Smart Contract Development (Rust/Soroban)

All smart contract source code is located in the `contracts/` directory.

> [!IMPORTANT]
> To compile and test the contracts, you must have Rust and the Soroban CLI installed on your machine. See the [Stellar Developer Docs](https://developers.stellar.org) for installation guides.

#### 1. Compile Contracts
To build the contracts into WASM bytecode, run the following command from the root directory:
```bash
cargo build --manifest-path contracts/Cargo.toml --target wasm32-unknown-unknown --release
```
Alternatively, you can navigate into the `contracts` directory and run:
```bash
cd contracts
cargo build --target wasm32-unknown-unknown --release
```

#### 2. Run Tests
StellarHomes uses Rust's built-in testing framework for unit and integration testing. Run tests with:
```bash
# From the root directory:
cargo test --manifest-path contracts/Cargo.toml

# Or from the contracts directory:
cd contracts
cargo test
```

#### 3. Linting and Formatting
Before committing, ensure your code complies with formatting rules and passes the linter:
```bash
# Format code
cargo fmt --manifest-path contracts/Cargo.toml

# Run Clippy (linter)
cargo clippy --manifest-path contracts/Cargo.toml -- -D warnings
```

---

### Frontend Web Application (Next.js/React/TypeScript)

The frontend is located in the `frontend/` directory and is built using Next.js (Pages Router), React, TypeScript, and Tailwind CSS.

#### 1. Install Dependencies
Navigate to the `frontend/` directory and install the required packages:
```bash
cd frontend
npm install
```

#### 2. Run Development Server
Run the local development server at `http://localhost:3000`:
```bash
npm run dev
```

#### 3. Production Build
Verify that the Next.js production build succeeds without errors:
```bash
npm run build
```

#### 4. Linting
Verify TypeScript types and run ESLint:
```bash
# Run ESLint check
npm run lint

# Run type check
npx tsc --noEmit
```

---

## 📐 Coding Standards & Best Practices

### Soroban Smart Contracts

To maintain the security, upgradability, and readability of the smart contracts:
- **Authorization Checks**: Always verify that the caller is authorized. Use `require_auth()` for functions changing status or balances.
- **Reentrancy Mitigation**: Modify all internal state (e.g. user balances, transaction status) *before* invoking external contract calls or token transfers.
- **Event Logging**: Emit a descriptive event for every state-changing action. Events must follow the patterns laid out in [ARCHITECTURE.md](ARCHITECTURE.md) (e.g., `ProposalCreated`, `ConfirmationRecorded`, `TokenDeposited`).
- **Defensive Quorums**: Ensure that administrative changes to quorum satisfy `0 < quorum <= total_signers`.
- **Bounded execution delays**: Limit execution delays to sensible defaults and respect the maximum `MAX_DELAY` constraint of 30 days.

### Frontend Client (React & TypeScript)

To keep the interface premium, bug-free, and high-performance:
- **Strict Typing**: Avoid using `any`. Define clear TypeScript interfaces and types for props, states, and contract payloads.
- **Consistent File Layout**:
  - Components belong in `frontend/src/components/`.
  - Pages belong in `frontend/src/pages/`.
  - Global styles belong in `frontend/src/styles/`.
- **Aesthetics & UI**:
  - Keep styling consistent with Tailwind CSS. Maintain responsiveness across mobile, tablet, and desktop breakpoints.
  - Follow the established dark mode design system. Avoid hardcoded raw colors and prefer Tailwind theme utility classes and CSS variables.
  - Use subtle hover states and micro-animations to enhance interactive components (e.g. buttons, proposals, form inputs).
- **SEO & Accessibility**:
  - Every page should include proper meta titles and descriptions.
  - Use semantic HTML tags (`<header>`, `<main>`, `<section>`, `<footer>`, etc.) instead of nesting generic `<div>` tags exclusively.
  - All form controls and interactive buttons must have unique, descriptive `id` attributes.

---

## 🔀 Git Branching & Commit Guidelines

We use structured branching and descriptive commits to maintain a clean project history.

### Branch Naming Convention

Name your branches based on the nature of your changes:
- `feature/` - New features or capabilities (e.g., `feature/dynamic-quorum`)
- `fix/` - Bug fixes (e.g., `fix/proposal-fee-refund`)
- `docs/` - Documentation updates (e.g., `docs/contributing-guidelines`)
- `refactor/` - Code restructuring without behavioral changes (e.g., `refactor/roles-validation`)

### Commit Message Format

Commits should have a clear category and description. Use the following prefix convention:
- `feat(<scope>):` for new features
- `fix(<scope>):` for bug fixes
- `docs:` for documentation modifications
- `style:` for formatting, white-space adjustments, or visual styling changes
- `refactor(<scope>):` for code refactoring

*Examples:*
- `feat(contracts): add dynamic quorum thresholds`
- `fix(frontend): resolve double submission on proposal creation`
- `docs: update deployment guidelines in readme`

---

## 📤 Submitting a Pull Request

Ready to submit your changes? Follow this checklist to ensure a quick merge:

1. **Keep Branch Synchronized**: Rebase or merge the latest `main` into your feature branch before submitting:
   ```bash
   git fetch upstream
   git merge upstream/main
   ```
2. **Verify All Checks Pass**:
   - Ensure `cargo test` passes successfully.
   - Run `cargo fmt` and `cargo clippy`.
   - Run `npm run build` and `npm run lint` in the `frontend` folder.
3. **Submit the PR**: Go to the GitHub repository and click "Compare & pull request".
4. **Fill Out the PR Template**:
   - **Summary**: Describe the changes, the rationale behind them, and what problem they solve.
   - **Testing**: Explain how you verified your changes (e.g. unit tests, browser manual testing, contract deployment).
   - **Related Issues**: Reference any open issues resolved by the PR (e.g., `Closes #12`).
5. **Address Feedback**: Be prepared to make modifications based on reviewer suggestions. Once approved, your PR will be merged into `main`.
