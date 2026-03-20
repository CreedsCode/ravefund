# RaveFund — Architecture & Trust Model

## System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Contributor Flow                      │
│                                                         │
│   Telegram/X ──→ Blink ──→ Contribute USDC ──→ Vault   │
│       or         Link       (SPL Transfer)    (PDA)     │
│   Web App                                               │
└───────────────────────┬─────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   Solana                         │
│                                                         │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  Vault   │  │  Governance  │  │  Ticket Minter   │  │
│  │  Escrow  │  │   Module     │  │  (cNFTs)         │  │
│  │  (PDA)   │  │  (Quadratic) │  │                  │  │
│  └────┬─────┘  └──────┬───────┘  └────────┬─────────┘  │
│       │               │                    │            │
│       ▼               ▼                    ▼            │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Fund State Account                  │   │
│  │  - target_amount: u64                            │   │
│  │  - current_amount: u64                           │   │
│  │  - deadline: i64                                 │   │
│  │  - organizer: Pubkey                             │   │
│  │  - status: enum { Funding, Active, Complete,     │   │
│  │                    Refunding, Cancelled }         │   │
│  │  - spending_log: Vec<SpendEntry>                 │   │
│  │  - contributor_count: u32                        │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   Frontend / Web App                    │
│                                                         │
│  - Fund creation wizard                                 │
│  - Live funding progress                                │
│  - Voting interface                                     │
│  - Spending log viewer (real-time)                      │
│  - Ticket claim / wallet connect                        │
└─────────────────────────────────────────────────────────┘
```

---

## Permission Framework

RaveFund has three distinct actor roles with different permission levels.

### Roles

| Role | Who | Permissions |
|------|-----|-------------|
| **Creator** | Person who initializes the fund | Set target amount, deadline, description, perk tiers. Cannot withdraw funds. Can cancel fund (triggers auto-refund) before target is met. |
| **Organizer** | Designated executor (can be same as creator) | Withdraw funds post-target with mandatory memo. Execute vendor payments. Update event details. Cannot change target or deadline after creation. |
| **Contributor** | Anyone who deposits USDC | Vote on proposals (weight = quadratic of contribution). Claim NFT ticket. Trigger refund claim if deadline passes without target. View all spending. |

### Permission Matrix

```
Action                          Creator  Organizer  Contributor  Smart Contract
─────────────────────────────────────────────────────────────────────────────────
Create fund                       ✅        —           —            —
Set target & deadline             ✅        —           —            —
Cancel fund (pre-target)          ✅        —           —            —
Contribute USDC                   ✅        ✅          ✅            —
Withdraw from vault               —        ✅           —            —
  (requires memo)
Log spending entry                —        ✅           —            —
Create governance proposal        ✅        ✅          ✅            —
Vote on proposal                  ✅        ✅          ✅            —
Claim NFT ticket                  ✅        ✅          ✅            —
Trigger auto-refund               —         —           —           ✅
  (deadline passed, target not met)
Distribute tickets                —         —           —           ✅
  (target met)
```

---

## Trust Model — The Spectrum

RaveFund sits deliberately in the middle of the trust spectrum. Not everything should be trustless. Here's the breakdown:

### Fully Trustless (Enforced by Smart Contract)

These guarantees hold regardless of organizer behavior:

- **Escrow** — Contributions go to a program-derived address (PDA). No single private key controls the vault. The program defines withdrawal conditions.
- **Auto-refund** — If `current_amount < target_amount` when `deadline` passes, any contributor can invoke `claim_refund()`. The program calculates pro-rata return and transfers directly. No organizer approval needed.
- **Ticket minting** — When the fund transitions to `Active` status, cNFT tickets are minted to contributor wallets automatically based on contribution tier. No manual distribution.
- **Voting** — Governance votes are recorded on-chain. Vote weight is derived from contribution amount using quadratic formula (`weight = sqrt(contribution)`). Results are computed deterministically by the program.
- **Spending transparency** — Every withdrawal from the vault includes a mandatory `memo` field (Solana Memo Program). These are permanently recorded and publicly visible.

### Trusted to Organizer (Intentionally Permissioned)

These actions require trusting the organizer. This is a deliberate design choice:

- **Spending execution** — The organizer can withdraw funds from the vault after target is met, with a memo describing the expense. There is **no per-spend governance vote**. The organizer is trusted to spend responsibly.
- **Vendor payments** — The organizer pays vendors directly. RaveFund does not enforce that payments go to "verified vendors" because real-world event logistics don't work that way (cash tips, last-minute swaps, emergency purchases).
- **Event delivery** — There is no oracle that verifies "the rave actually happened." The organizer is trusted to deliver the event.

### Why We Choose Trust Here

Putting every spending decision into a governance vote would kill the product:

1. **Speed** — Event logistics move in hours, not governance cycles. A venue wants a deposit by 6pm today. A DJ cancels and you need to book a replacement in 2 hours. You can't wait for 47 people to vote.
2. **Operational reality** — Many event costs are cash, informal, or negotiated in real-time. Forcing on-chain vendor verification would exclude 90% of real-world suppliers.
3. **Governance fatigue** — Communities that vote on everything, vote on nothing. Contributor engagement drops to zero after the third proposal about speaker rental pricing.
4. **The actual problem we solve** — The problem isn't that organizers make bad spending decisions. It's that contributors can't *see* what decisions were made, and can't get their money back if the event never happens. RaveFund solves both of those.

### The Accountability Loop

Trust doesn't mean blind trust. The accountability mechanism is:

```
Organizer withdraws funds
        │
        ▼
Memo logged on-chain (immutable, public)
        │
        ▼
Contributors see real-time spending feed
        │
        ▼
Post-event: on-chain spending history becomes
the organizer's permanent reputation record
        │
        ▼
Next fund: contributors check organizer's
historical spending behavior before backing
```

The organizer's on-chain spending history across all their funds acts as a **reputation system**. Over time, this is more powerful than per-transaction governance — it creates market-driven accountability.

---

## Governance Module

### What Gets Voted On

Governance is reserved for **high-level creative decisions** that shape the event identity:

- Headline DJ / artist selection (from organizer-curated shortlist)
- Venue style (warehouse vs. outdoor vs. club)
- Theme / aesthetic direction
- Date selection (from organizer-proposed options)
- Perk tier structure

### What Does NOT Get Voted On

Operational spending and logistics are organizer-managed:

- Vendor selection and pricing
- Sound system rental
- Security and door staff
- Marketing spend
- Emergency expenses
- Cash outlays

### Voting Mechanics

- **Weight**: Quadratic — `vote_weight = floor(sqrt(contribution_in_usdc))`
- **Rationale**: Prevents plutocratic capture. Someone contributing 1,000 USDC gets ~31x voting power, not 100x, compared to someone contributing 10 USDC.
- **Quorum**: Configurable per fund (default: 30% of contributors must vote)
- **Duration**: Configurable per proposal (default: 48 hours)
- **Implementation**: Lightweight on-chain voting — not full Realms DAO. Proposals are created by organizer or contributors, votes are SPL token-weighted.

---

## Ticketing (Compressed NFTs)

### Why cNFTs

- Mint 10,000 tickets for < $1 total (compressed Merkle tree)
- Each ticket is a unique on-chain asset with metadata
- Composable — integrates with existing Solana NFT infrastructure
- Transferable on secondary markets (optional, configurable)

### Ticket Tiers (Example)

| Contribution | Tier | Perks |
|-------------|------|-------|
| 10–49 USDC | 🎵 Supporter | General admission ticket |
| 50–149 USDC | 🎧 Backer | GA + priority entry |
| 150–499 USDC | 🔊 Champion | GA + priority entry + backstage |
| 500+ USDC | 👑 Patron | All above + organizer shoutout + lifetime fund discount |

Tiers are configurable by the fund creator. Perks are social commitments, not smart contract enforced (because "backstage access" can't be verified on-chain).

---

## Fund Lifecycle

```
┌──────────┐     target met      ┌──────────┐     event done     ┌───────────┐
│ FUNDING  │ ──────────────────→ │  ACTIVE  │ ──────────────────→│ COMPLETE  │
└──────────┘                     └──────────┘                    └───────────┘
     │                                │
     │ deadline passed                │ organizer cancels
     │ target NOT met                 │ (remaining funds refunded)
     ▼                                ▼
┌──────────────┐              ┌──────────────┐
│  REFUNDING   │              │  CANCELLED   │
│  (auto)      │              │  (partial    │
│              │              │   refund)    │
└──────────────┘              └──────────────┘
```

### State Transitions

| From | To | Trigger | Who |
|------|----|---------|-----|
| Funding | Active | `current_amount >= target_amount` | Automatic |
| Funding | Refunding | `now > deadline && current_amount < target_amount` | Any contributor calls `claim_refund()` |
| Active | Complete | Organizer calls `complete_fund()` | Organizer |
| Active | Cancelled | Organizer calls `cancel_fund()` | Organizer (remaining balance refunded pro-rata) |

---

## Future Considerations

- **Reputation NFTs** — Soulbound tokens for organizers showing fund history, completion rate, contributor satisfaction
- **Streaming contributions** — Drip-fund using Streamflow or similar for recurring community event funds
- **Cross-fund composability** — Frequent contributors get perks across multiple RaveFunds (loyalty layer)
- **Fiat on-ramp** — Stripe/MoonPay integration so non-crypto contributors can participate via card payment
- **Multi-sig organizer** — Optional 2-of-3 multisig for larger funds where single-organizer trust isn't sufficient
