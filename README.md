# Turbin3 Pre-Builder Q425

## Capstone: Cross-Chain Funding Rate Arbitrage System

### The Opportunity

Capture **funding rate differentials** between Drift (Solana) and Ambient (Fogo) through delta-neutral positions. Different user bases create persistent spreads: retail Solana vs institutional Fogo.

- Note: the aim is to demonstrate an scalable design that eventually can be taken to production after mvp.
---
# Components Diagram
```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                           ARBITRAGE SYSTEM ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                     │
│  ┌────────────────────────────────────────────────────────────────────────────────┐ │
│  │                           OFF-CHAIN LAYER (Rust Bot)                           │ │
│  ├────────────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                                │ │
│  │  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐             │ │
│  │  │ FUNDING MONITOR │    │ OPPORTUNITY     │    │ RISK ENGINE     │             │ │
│  │  │                 │───▶│ SCANNER         │───▶│                 │             │ │
│  │  │ • Drift API     │    │                 │    │ • Margin check  │             │ │
│  │  │ • Ambient API   │    │ • Spread calc   │    │ • Position size │             │ │
│  │  │ • WebSocket     │    │ • Threshold     │    │ • Stop-loss     │             │ │
│  │  └─────────────────┘    └─────────────────┘    └────────┬────────┘             │ │
│  │                                                         │                      │ │
│  │                                                         ▼                      │ │
│  │                              ┌─────────────────────────────────┐               │ │
│  │                              │     EXECUTION COORDINATOR       │               │ │
│  │                              │                                 │               │ │
│  │                              │  • Atomic cross-chain orders    │               │ │
│  │                              │  • Rollback on failure          │               │ │
│  │                              │  • Position tracking            │               │ │
│  │                              └──────────────┬──────────────────┘               │ │
│  │                                             │                                  │ │
│  └─────────────────────────────────────────────┼──────────────────────────────────┘ │
│                                                │                                    │
│                    ┌───────────────────────────┴───────────────────────────┐        │
│                    ▼                                                       ▼        │
│  ┌─────────────────────────────────────┐     ┌─────────────────────────────────────┐│
│  │       SOLANA ON-CHAIN LAYER         │     │         FOGO ON-CHAIN LAYER         ││
│  ├─────────────────────────────────────┤     ├─────────────────────────────────────┤│
│  │                                     │     │                                     ││
│  │  ┌───────────────────────────────┐  │     │  ┌───────────────────────────────┐  ││
│  │  │         VAULT PROGRAM         │  │     │  │         VAULT PROGRAM         │  ││
│  │  │                               │  │     │  │                               │  ││
│  │  │  • Deposit/Withdraw           │  │     │  │  • Deposit/Withdraw           │  ││
│  │  │  • Delegate authority         │  │     │  │  • Delegate authority         │  ││
│  │  │  • Risk constraints           │  │     │  │  • Risk constraints           │  ││
│  │  │  • LP token mint              │  │     │  │  • LP token mint              │  ││
│  │  └──────────────┬────────────────┘  │     │  └──────────────┬────────────────┘  ││
│  │                 │                   │     │                 │                   ││
│  │                 ▼                   │     │                 ▼                   ││
│  │  ┌───────────────────────────────┐  │     │  ┌───────────────────────────────┐  ││
│  │  │      DRIFT ADAPTER (CPI)      │  │     │  │     AMBIENT ADAPTER (CPI)     │  ││
│  │  │                               │  │     │  │                               │  ││
│  │  │  • place_perp_order()         │  │     │  │  • open_position()            │  ││
│  │  │  • close_position()           │  │     │  │  • close_position()           │  ││
│  │  │  • get_funding_rate()         │  │     │  │  • get_funding_rate()         │  ││
│  │  └──────────────┬────────────────┘  │     │  └──────────────┬────────────────┘  ││
│  │                 │                   │     │                 │                   ││
│  │                 ▼                   │     │                 ▼                   ││
│  │        ┌───────────────┐            │     │        ┌───────────────┐            ││
│  │        │ DRIFT PROTOCOL│            │     │        │AMBIENT FINANCE│            ││
│  │        └───────────────┘            │     │        └───────────────┘            ││
│  │                                     │     │                                     ││
│  └─────────────────────────────────────┘     └─────────────────────────────────────┘│
│                    │                                           │                    │
│                    └─────────────────┬─────────────────────────┘                    │
│                                      │                                              │
│                            ┌─────────▼─────────┐                                    │
│                            │     WORMHOLE      │                                    │
│                            │                   │                                    │
│                            │ • State sync      │                                    │
│                            │ • Position verify │                                    │
│                            │ • Emergency relay │                                    │
│                            └───────────────────┘                                    │
│                                                                                     │
└─────────────────────────────────────────────────────────────────────────────────────┘
```


## Current State (Dec 2025)

| Component | Status | Details |
|-----------|--------|---------|
| Drift Protocol | **Live** | $1B+ TVL, $70B+ cumulative volume |
| Fogo Mainnet | **Jan 13, 2026** | Genesis Phase active, USDC bridging open |
| Ambient Finance | **Confirmed** | Native perps DEX on Fogo, uses DFBA model |
| Funding Spreads | **2-4 bps/hour** | speculative ~17-35% APY opportunity |

---

## Why This Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         FUNDING RATE ARBITRAGE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SOLANA (Drift)                    FOGO (Ambient)                           │
│  ─────────────────                 ─────────────────                        │
│  Retail traders                    Institutional traders                    │
│  Directional (long-biased)         Delta-neutral hedging                    │
│  Volatile funding rates            Stable funding rates                     │
│                                                                             │
│  When Solana longs pay funding → Fogo shorts receive funding                │
│  = PERSISTENT SPREAD OPPORTUNITY                                            │
│                                                                             │
│  STRATEGY:                                                                  │
│  1. SHORT on platform with positive funding (receive payments)              │
│  2. LONG on platform with negative funding (pay less)                       │
│  3. Net = funding spread captured, delta-neutral                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Architecture

### On-Chain Components
- Vault programs on Solana and Fogo (hold principal)
- Wormhole bridge integration (cross-chain capital)
- CPI to Drift and Ambient protocols

### Off-Chain Components
- Funding rate monitoring (Drift API + Ambient API)
- Opportunity detection & signal generation
- Cross-chain execution coordination
- Risk management & position tracking

---

## MVP Scope

| Component | MVP | Post-MVP |
|-----------|-----|----------|
| Chain Support | Drift only (Solana devnet) | Add Ambient (Fogo mainnet) |
| Funding Monitor | Historical + live Drift rates | Cross-chain comparison |
| Position Management | Single-chain vault | Cross-chain vaults |
| Execution | Manual demo | Automated |

### MVP Demo Flow

```
[1] Display live funding rates from Drift
         ↓
[2] Detect opportunity (rate > threshold)
         ↓
[3] Execute position on Drift devnet
         ↓
[4] Track P&L + funding accrual
         ↓
[5] Present scaling path to cross-chain
```

---

## User Stories (MVP)

### Personas

| Persona | Goal |
|---------|------|
| **Arbitrage Trader** | Capture funding rate spreads with minimal risk |
| **Vault Depositor** | Earn yield without active management |

---

### Epic 1: Funding Rate Monitoring

**1.1: View Live Funding Rates**
> As an **Arbitrage Trader**, I want to **view real-time funding rates from Drift** so that **I can identify profitable opportunities**.

- [ ] Dashboard displays rates for BTC-PERP, ETH-PERP, SOL-PERP
- [ ] Rates update every 60 seconds
- [ ] Historical chart shows last 24 hours

**1.2: Set Rate Alerts**
> As an **Arbitrage Trader**, I want to **set threshold alerts** so that **I am notified when opportunities arise**.

- [ ] Configurable threshold (e.g., >5 bps/hour)
- [ ] Alert triggers when exceeded
- [ ] Shows affected market and spread

---

### Epic 2: Position Management

**2.1: Deposit Capital**
> As a **Vault Depositor**, I want to **deposit USDC into the vault** so that **my capital is ready for execution**.

- [ ] Connect Solana wallet (Phantom/Backpack)
- [ ] Submit deposit transaction to devnet
- [ ] Balance reflected in dashboard

**2.2: Execute Trade**
> As an **Arbitrage Trader**, I want to **open a perpetual position** so that **I can collect funding payments**.

- [ ] Select market and position size
- [ ] Execute via Drift CPI
- [ ] Position appears in active list

**2.3: Track P&L**
> As an **Arbitrage Trader**, I want to **see unrealized P&L including funding** so that **I know when to exit**.

- [ ] Display entry price, current price, funding accrued
- [ ] Net P&L calculation
- [ ] Close position available

---

### Epic 3: Cross-Chain (Post-MVP)

**3.1: Bridge to Fogo**
> As an **Arbitrage Trader**, I want to **bridge USDC to Fogo via Wormhole** so that **I can deploy capital on both chains**.

- [ ] Wormhole bridge integration
- [ ] Bridge status tracking
- [ ] Fogo balance displayed

