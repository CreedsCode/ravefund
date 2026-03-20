# RaveFund

**Pool. Vote. Rave.**
Community-funded events with on-chain transparency and trustless ticketing on Solana.

---

## The Problem

Underground rave and music communities run on trust and group chats. Someone collects money via PayPal or Revolut, books a venue, maybe a DJ — and everyone just hopes it works out.

- No transparency on how funds are spent
- No real say in creative decisions
- If the organizer ghosts, your money's gone
- Bigger promoters gatekeep lineups and pricing

The culture that's supposed to be community-driven... isn't.

## The Solution

RaveFund lets any community spin up an on-chain event fund on Solana.

### How It Works

```
[ Contributors ] → [ Fund Vault ] → [ Organizer Executes ] → [ Event Happens ]
       ↓                 ↓                    ↓                      ↓
   Governance        On-chain            Transparent            NFT Tickets
    Weight           Escrow              Spending Log           to backers
```

1. **Fund** — A community creates a RaveFund vault with a target amount (e.g. 5,000 USDC). Anyone can contribute. Funds are held in a program-controlled escrow.
2. **Vote** — Contributors vote on high-level creative decisions: venue style, headline DJ, theme, date. Quadratic voting prevents whale takeover.
3. **Execute** — Organizers manage day-to-day spending against the fund. Every withdrawal is logged on-chain with a memo.
4. **Rave** — Contributors receive compressed NFT tickets proportional to their contribution, with optional perks (backstage, early entry) set by the community.
5. **Refund** — If the target isn't met by the deadline, all contributions auto-refund via smart contract.

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the full technical design, trust model, and permission framework.

## Trust Model (TL;DR)

RaveFund is **not fully trustless**. This is intentional.

| Layer | Trustless? | Why |
|-------|-----------|-----|
| Fund collection & escrow | ✅ Yes | Program-controlled vault, no single keyholder |
| Auto-refund on failed target | ✅ Yes | Smart contract enforced, no human intervention |
| NFT ticket distribution | ✅ Yes | Automatic on funding milestone |
| High-level creative votes | ✅ Yes | On-chain governance, quadratic weighted |
| Day-to-day spending | ❌ No | Organizer is trusted — see [why](#why-not-fully-trustless) |
| Vendor payments | ❌ No | Organizer executes, community audits after |

### Why Not Fully Trustless?

Putting every €50 speaker rental into a governance vote kills the event. Real event organizing requires fast, ungoverned decisions — negotiating last-minute DJ fees, swapping a broken PA system at 11pm, tipping the door guy in cash.

RaveFund's position: **trust the organizer with execution, but make every spend visible.** You're not removing trust — you're making it auditable. If an organizer misuses funds, the on-chain spending log is permanent evidence, and their reputation score reflects it.

This is how the real world works. You trust a festival organizer to put on a good show. RaveFund just makes sure you can see exactly where your money went — and get it back if the show never happens.

## Categories

- 🎧 **Consumer Apps** — onboarding music communities to crypto without them needing to care about crypto
- 💰 **DeFi** — community pooling, escrow, programmatic refunds
- 🪙 **Stablecoins/Payments** — USDC-denominated contributions and vendor payouts

## Tech Stack (Proposed)

- **Solana** — program runtime (Anchor framework)
- **USDC (SPL)** — contribution denomination
- **Compressed NFTs (cNFTs)** — ticketing at scale (sub-cent mint cost)
- **Realms / SPL Governance** — voting infrastructure (or custom lightweight governance)
- **Helius** — RPC, webhooks, cNFT indexing
- **Next.js** — contributor-facing web app
- **Dialect / Blinks** — Solana Actions for in-chat contributions ("fund this rave" links)

## Why Solana?

- **Sub-cent transactions** — micro-contributions (5 USDC) are viable, not eaten by gas
- **Fast finality** — voting rounds resolve in seconds
- **Compressed NFTs** — mint 10,000 tickets for < $1
- **Blinks/Actions** — contribute directly from Twitter/Telegram links
- **Ecosystem** — Realms governance, SPL tokens, composable DeFi rails already exist

## User Story

> *"My crew wants to throw a warehouse party in Cologne. I create a RaveFund, share the Blink in our Telegram group. 47 people chip in between 10–200 USDC. We vote on the headliner — community picks a local techno DJ over the expensive Berlin import. The organizer books the venue, rents sound, handles the door. Every payment shows up on the fund page. I get my cNFT ticket + early entry perk for being a top-10 backer. If only 12 people had funded and we didn't hit target? Auto-refund, no drama."*

## License

MIT

---

**Built for the Superteam Germany Supertour Aachen Ideathon 🇩🇪**