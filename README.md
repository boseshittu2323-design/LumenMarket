# LumenMarket

A permissionless token launchpad on Stellar with bonding curve price discovery that automatically migrates liquidity to the native DEX when a market cap target is reached.

---

## Table of Contents

1. [Problem It Solves](#problem-it-solves)
2. [Architecture](#architecture)
3. [Project Structure](#project-structure)
4. [Getting Started](#getting-started)
5. [Contributing](#contributing)

---

## Problem It Solves

| Gap in the Ecosystem | How LumenMarket Addresses It |
|---|---|
| Token launches require upfront liquidity and trusted intermediaries | Bonding curve provides instant, trustless price discovery from token 0 — no liquidity bootstrapping needed |
| Projects rug-pull or lock liquidity indefinitely | When `xlm_raised >= target_xlm`, anyone can call `migrate()` — the contract autonomously creates a DEX offer and burns unsold tokens |
| No transparent mechanism to track fundraising progress | Every buy/sell emits on-chain events; the UI shows live progress bars and a price curve chart; the Telegram bot broadcasts milestone alerts |

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Users / Creators                       │
└────────────┬────────────────────────────────┬────────────────┘
             │ browser                        │ Telegram
             ▼                                ▼
┌────────────────────────┐      ┌─────────────────────────────┐
│   Next.js 14 Frontend  │      │   Python aiogram Bot        │
│  /web                  │      │  /bot                       │
│  • Token card grid     │      │  • /launches, /price <id>   │
│  • Bonding curve chart │      │  • /subscribe for alerts    │
│  • 3-step create wizard│      │  • Polls events every 30s   │
└────────────┬───────────┘      └──────────────┬──────────────┘
             │ Horizon / Soroban RPC            │ Soroban RPC getEvents
             ▼                                  ▼
┌──────────────────────────────────────────────────────────────┐
│              Soroban Smart Contract  /contract               │
│                                                              │
│  create_launch()  ──►  mint token, init BondingCurve         │
│  buy()            ──►  Δprice = virtual_xlm/virtual_tokens   │
│  sell()           ──►  inverse bonding curve math            │
│  migrate()        ──►  fee → admin, DEX offer, burn unsold   │
│  current_price()  ──►  view: virtual_xlm×1e7/virtual_tokens  │
│                                                              │
│  Events: LaunchCreated | TokenBought | TokenSold | Migrated  │
└──────────────────────────────┬───────────────────────────────┘
                               │ on migrate()
                               ▼
                  ┌────────────────────────┐
                  │   Stellar Native DEX   │
                  │  manage_sell_offer     │
                  │  (XLM ↔ Token pair)   │
                  └────────────────────────┘
```

**Key design decisions:**

- **Constant-product bonding curve** — `tokens_out = virtual_tokens × xlm_in / (virtual_xlm + xlm_in)`. Virtual reserves prevent zero-price edge cases at launch.
- **Permissionless migration** — no admin key required to trigger migration; the on-chain condition (`xlm_raised >= target_xlm`) is the sole gate.
- **Fee model** — flat `creation_fee` in XLM on launch; `migration_fee_bps` (basis points) taken from raised XLM on migrate. Both flow to `admin`.
- **No oracle dependency** — price is entirely determined by curve state; external price feeds are not needed.

---

## Project Structure

```
LumenMarket/
│
├── contract/                   # Soroban smart contract (Rust)
│   ├── Cargo.toml              # soroban-sdk 20, cdylib, release profile
│   └── src/
│       ├── lib.rs              # All contract logic: structs, impl, events
│       └── test.rs             # testutils suite (init, buy/sell, migration, slippage)
│
├── web/                        # Next.js 14 frontend
│   ├── app/
│   │   ├── layout.tsx          # Root layout, dark theme
│   │   ├── page.tsx            # Token card grid + "Create Launch" CTA
│   │   ├── create/page.tsx     # Wraps CreateWizard
│   │   └── launch/[id]/page.tsx # Launch detail: chart + buy/sell form
│   ├── components/
│   │   ├── TokenCard.tsx       # Name, progress bar, price, migrated badge
│   │   ├── BondingCurveChart.tsx # Chart.js line: price vs tokens sold
│   │   └── CreateWizard.tsx    # 3-step form (details → curve → deploy)
│   ├── lib/
│   │   ├── types.ts            # Launch interface
│   │   └── contract.ts         # Soroban interaction helpers (mock + TODOs)
│   ├── package.json
│   └── tailwind.config.js
│
└── bot/                        # Telegram alert bot (Python)
    ├── bot.py                  # aiogram 3 handlers, background polling loop
    ├── watcher.py              # Soroban RPC event fetcher (httpx async)
    └── requirements.txt        # aiogram, httpx, python-dotenv
```

---

## Getting Started

### Prerequisites

- Rust + `wasm32-unknown-unknown` target
- [Stellar CLI](https://developers.stellar.org/docs/tools/stellar-cli) ≥ 0.9
- Node.js ≥ 18
- Python 3.11+

---

### 1 — Build & Deploy the Contract

```bash
cd contract

# Add wasm target (once)
rustup target add wasm32-unknown-unknown

# Build optimised WASM
cargo build --release --target wasm32-unknown-unknown

# Deploy to Testnet (requires funded identity)
stellar contract deploy \
  --wasm target/wasm32-unknown-unknown/release/lumen_market.wasm \
  --network testnet \
  --source <YOUR_IDENTITY>

# Initialise (replace placeholders)
stellar contract invoke \
  --id <CONTRACT_ID> \
  --network testnet \
  --source <YOUR_IDENTITY> \
  -- init \
  --admin <ADMIN_ADDRESS> \
  --creation_fee 10000000 \
  --migration_fee_bps 100
```

### 2 — Run Contract Tests

```bash
cd contract
cargo test
```

### 3 — Run the Frontend

```bash
cd web
npm install
cp .env.example .env.local   # set NEXT_PUBLIC_CONTRACT_ID and NEXT_PUBLIC_RPC_URL
npm run dev                  # http://localhost:3000
```

To build for production:

```bash
npm run build && npm start
```

### 4 — Run the Telegram Bot

```bash
cd bot
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Required env vars
export BOT_TOKEN=<your_telegram_bot_token>
export SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
export CONTRACT_ID=<deployed_contract_id>

python bot.py
```

---

## Contributing

### Workflow

1. Fork the repo and create a feature branch: `git checkout -b feat/<short-description>`
2. Make your changes, following the code standards below.
3. Run relevant tests (`cargo test` / `npm run build`) before opening a PR.
4. Open a pull request against `main` with a clear title (≤ 70 chars) and description covering: what changed, how it was tested, and any known limitations.

### PR Guide

- Keep PRs focused — one logical change per PR.
- Link the relevant GitHub issue with `Closes #<number>`.
- PRs touching the contract require at least one new or updated test in `test.rs`.
- PRs touching the frontend require no TypeScript errors (`npm run build` passes).

### Areas to Contribute

| Area | Examples |
|---|---|
| Contract | Real SEP-41 token minting, actual DEX offer creation on migrate, price oracle integration |
| Frontend | Wallet connection (Freighter/WalletConnect), real Soroban RPC calls in `contract.ts`, mobile responsiveness |
| Bot | `/alert <id> <price>` price-target alerts, chart image generation, multi-language support |
| Tests | Fuzz tests for bonding curve math, integration tests against a local Stellar node |
| DevOps | GitHub Actions CI for contract build + test, Vercel/Netlify deploy workflow |

### Code Standards

- **Rust**: `cargo fmt` + `cargo clippy -- -D warnings` must pass. No `unwrap()` in contract code — use `expect()` with a message or propagate errors.
- **TypeScript**: strict mode enabled. No `any` types. Components must be typed with explicit props interfaces.
- **Python**: `black` formatting. Type hints on all public functions. No bare `except:` clauses.
- Commit messages follow Conventional Commits: `feat:`, `fix:`, `docs:`, `test:`, `chore:`.

### Reporting Issues

Open a GitHub issue with the **Bug** or **Feature Request** template. For bugs include: steps to reproduce, expected vs actual behaviour, and relevant contract/transaction IDs if on-chain.
