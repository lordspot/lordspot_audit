# LordsPot Protocol — Audit & Security Review Guide

Welcome to the security review repository for **LordsPot**. This repository contains the smart contracts that power LordsPot across both **Solana** and **Base (EVM)**.

This guide is written in plain, simple English without confusing jargon.

---

## 🎯 Contracts in Scope to Audit (nSLOC & Deployments)

| Smart Contract / Program | Target Platform | Source File(s) | nSLOC | Devnet Deployed Address |
| :--- | :--- | :--- | :---: | :--- |
| **`LordsPotBaseVault`** | Base (EVM / Solidity) | [`base_smart_contracts/src/LordsPotBaseVault.sol`](base_smart_contracts/src/LordsPotBaseVault.sol) | 165 | [`0xE11b2B80fD954eA7e6a9B6e4E9A5Af406a2De4f5`](https://basescan.org/address/0xE11b2B80fD954eA7e6a9B6e4E9A5Af406a2De4f5) |
| **`solana_smart_contracts`** (LordsPot Program) | Solana (SVM / Anchor Rust) | [`solana_smart_contracts/programs/solana_smart_contracts/src/lib.rs`](solana_smart_contracts/programs/solana_smart_contracts/src/lib.rs)<br>[`solana_smart_contracts/programs/solana_smart_contracts/src/constants.rs`](solana_smart_contracts/programs/solana_smart_contracts/src/constants.rs) | 673<br>*(648 + 25)* | [`5M2BS7XuZgFtKWBBGdyNy4g3UkgdMvd7gvaFVvabcGWo`](https://explorer.solana.com/address/5M2BS7XuZgFtKWBBGdyNy4g3UkgdMvd7gvaFVvabcGWo?cluster=devnet) |
| **Total Scope** | | | **838** | |

---

## 📌 Quick Reference & Protocol PDAs

| Component | Network | Address / Identifier | Description |
| :--- | :--- | :--- | :--- |
| **LordsPot Program** | **Solana Devnet** | [`5M2BS7XuZgFtKWBBGdyNy4g3UkgdMvd7gvaFVvabcGWo`](https://explorer.solana.com/address/5M2BS7XuZgFtKWBBGdyNy4g3UkgdMvd7gvaFVvabcGWo?cluster=devnet) | Anchor program handling purchases, state, and claim payouts |
| **State PDA** | **Solana Devnet** | `13fJA2pmD837DpSMFGGzcpGw4jS1P1cbncEUGnstvEYz` | Holds protocol configuration, admin key, treasury key, epoch & pricing |
| **Vault Authority PDA** | **Solana Devnet** | Derived: `[b"vault_authority"]` | Controls the Solana USDC vault token account |
| **Vault USDC Token Account (ATA)** | **Solana Devnet** | [`A59FKpMApFfsKwEqrd5YWoEUyGefDWvJSd4CzS4X8Grb`](https://explorer.solana.com/address/A59FKpMApFfsKwEqrd5YWoEUyGefDWvJSd4CzS4X8Grb?cluster=devnet) | Canonical USDC ATA owned by `vault_authority` PDA |
| **USDC Mint** | **Solana Devnet** | `4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU` *(Mainnet: `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`)* | SPL Token mint used for all ticket purchases and prize payouts |
| **LordsPot Base Vault** | **Base Devnet** | [`0xE11b2B80fD954eA7e6a9B6e4E9A5Af406a2De4f5`](https://basescan.org/address/0xE11b2B80fD954eA7e6a9B6e4E9A5Af406a2De4f5) | Solidity contract forwarding orders to Megapot & holding ticket NFTs |
| **Megapot Jackpot** | **Base (EVM)** | [`0x3bAe643002069dBCbcd62B1A4eb4C4A397d042a2`](https://basescan.org/address/0x3bAe643002069dBCbcd62B1A4eb4C4A397d042a2) | Target upstream jackpot lottery contract on Base |


---

## 🧭 Auditor Navigation — Choose Your Audit Track

To make your audit as efficient as possible, the technical details, known issues, test suites, and security invariants are split into two standalone sections:

- 🦀 **[Part 1: Solana Program Audit Track (Anchor / Rust)](#part-1-solana-program-audit-track-anchor--rust)**  
  *For Solana auditors: Covers Solana program instructions, account constraints, PDAs, token transfers, hot/cold key separation, Solana-specific known issues, and Anchor tests.*

- 🔷 **[Part 2: Base EVM Contract Audit Track (Solidity / Foundry)](#part-2-base-evm-contract-audit-track-solidity--foundry)**  
  *For EVM auditors: Covers Base vault router logic, Megapot interface, NFT handling, order deduplication, Base-specific known issues, and Foundry tests.*

---

## 📖 System Overview & Cross-Chain Lifecycle

**LordsPot** is a cross-chain lottery router that allows users on **Solana** to play in **Megapot** (an established jackpot lottery operating on **Base**), using **Solana USDC**.

Users stay entirely on Solana. They do not need an EVM wallet, Base ETH for gas, or a manual bridge. LordsPot handles the ticket validation, money collection on Solana, relayed ticket purchasing on Base, winning claim collection, and prize payouts back to Solana users in USDC.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. SOLANA: User buys ticket                                                 │
│    User calls buy_ticket() on Solana with selected numbers & USDC.          │
│    USDC moves into Solana Vault. TicketPurchaseEvent is emitted.            │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. RELAYER: Off-Chain Detection & Validation                                │
│    Backend detects the event, re-verifies the tx directly from Solana RPC,  │
│    and generates a unique orderId.                                          │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. BASE: Vault purchases tickets on Megapot                                 │
│    Relayer calls LordsPotBaseVault.buyTickets() on Base.                    │
│    Base Vault executes Megapot.buyTickets() using its own USDC balance.     │
│    Megapot mints ticket NFTs to the Base Vault. Order marked fulfilled.     │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. DRAWING & HARVEST: Megapot draws winning numbers                         │
│    Backend reads winning numbers from Megapot and grades user tickets.      │
│    Relayer calls LordsPotBaseVault.claimWinnings() on Base.                 │
│    Megapot burns ticket NFTs and transfers prize USDC into Base Vault.      │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. SOLANA PAYOUT: Winner claims USDC                                        │
│    Backend issues a half-signed claim voucher (signed by Admin hot key).    │
│    User counter-signs the transaction.                                      │
│    Solana program verifies both signatures and sends USDC to user's ATA.    │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 🎟️ Megapot Lifecycle: How Rounds, Drawings & Claims Work

Megapot operates on a continuous, daily cycle. Here is the lifecycle in plain English:

```
    [ Day Starts: Epoch N Opens ]
                 │
                 ▼
    ┌───────────────────────────┐
    │ 1. Active Buying Window   │ ◄── Users buy tickets anytime during the 24h day
    │    (Tickets Minted)       │     (Base Vault holds NFTs for Solana buyers)
    └────────────┬──────────────┘
                 │ (24 hours elapse / end of day)
                 ▼
    ┌───────────────────────────┐
    │ 2. Draw & Epoch Rollover  │ ◄── Ticket sales close for Epoch N
    │    (Winning Balls Picked) │     Winning balls drawn via on-chain randomness
    └────────────┬──────────────┘     Megapot rolls over: Epoch N+1 opens immediately
                 │
                 ▼
    ┌───────────────────────────┐
    │ 3. Claiming Winnings      │ ◄── Users can claim anytime for past epochs
    │    (Past Epochs Only)     │     NFT is burned, prize USDC paid out
    └───────────────────────────┘
```

1. **Epochs = Daily Rounds**:
   - Megapot organizes lottery draws into **epochs** (round numbers, e.g. Epoch 156, Epoch 157).
   - Each epoch lasts approximately **one day (24 hours)**.

2. **Ticket Purchasing (During the Day)**:
   - Throughout the day, the current active epoch is open for ticket purchases.
   - Users pick their numbers (5 regular balls + 1 bonus ball) and buy tickets.
   - For every purchase, Megapot mints unique ticket IDs as NFTs. In LordsPot, these NFTs are assigned to our Base Vault contract on behalf of the Solana user.

3. **Draw & Epoch Rollover (At the End of the Day)**:
   - At the end of the 24-hour cycle, ticket sales for that epoch close.
   - Megapot draws the official winning balls (5 regular numbers + 1 bonus ball) using on-chain randomness.
   - Megapot immediately rolls over to the next epoch (e.g. Epoch 156 concludes → Epoch 157 opens). Ticket sales for the new day's round begin immediately.

4. **Claiming Winnings (Anytime in the Future — for Past Epochs Only)**:
   - Once an epoch finishes and its winning balls are set, it becomes a **past epoch**.
   - Anyone holding winning tickets for a past epoch can claim their prize **anytime in the future** — there is no rush, countdown, or tight expiration window.
   - **Crucial Rule**: You can only claim winnings for an epoch that is **already in the past**. You cannot claim for an ongoing/active epoch before its winning numbers have been drawn.
   - When claimed via `claimWinnings`, Megapot checks the ticket numbers against the drawn balls, burns the ticket NFTs so they cannot be claimed again, and transfers the USDC prize money into the claimant's account (into the LordsPot Base Vault, which allows the winner to be paid out on Solana).

---
---

# Part 1: Solana Program Audit Track (Anchor / Rust)

> **Auditor Focus**: Anchor accounts & constraints, CPI token transfers, signature authorization, arithmetic safety, PDA derivations, and privilege separation.

### 📂 Solana Scope
- Directory: [`solana_smart_contracts/`](file:///Users/hail_the_lord/code/project/lordspot_audit/solana_smart_contracts/)
- Primary Program File: [`solana_smart_contracts/programs/solana_smart_contracts/src/lib.rs`](solana_smart_contracts/programs/solana_smart_contracts/src/lib.rs) (648 nSLOC)
- Constants File: [`solana_smart_contracts/programs/solana_smart_contracts/src/constants.rs`](solana_smart_contracts/programs/solana_smart_contracts/src/constants.rs) (25 nSLOC)
- **Total Solana nSLOC**: **673**
- Deployed Devnet Program ID: [`5M2BS7XuZgFtKWBBGdyNy4g3UkgdMvd7gvaFVvabcGWo`](https://explorer.solana.com/address/5M2BS7XuZgFtKWBBGdyNy4g3UkgdMvd7gvaFVvabcGWo?cluster=devnet)
- State PDA: `13fJA2pmD837DpSMFGGzcpGw4jS1P1cbncEUGnstvEYz`
- Vault Authority PDA: Derived from `[b"vault_authority"]`
- Vault USDC ATA: [`A59FKpMApFfsKwEqrd5YWoEUyGefDWvJSd4CzS4X8Grb`](https://explorer.solana.com/address/A59FKpMApFfsKwEqrd5YWoEUyGefDWvJSd4CzS4X8Grb?cluster=devnet)
- USDC Mint: Devnet `4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU` *(will update to `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` on Mainnet)*

---

### 🏛️ Solana Account Architecture & PDAs

1. **State Account (`LordsPotState`)**:
   - Seed: `[b"lords_pot_state"]`
   - Holds protocol global parameters:
     - `admin`: Hot key used for daily operations (pausing, resuming, updating ball ranges, co-signing claim vouchers).
     - `treasury_authority`: Cold key with sole authority to withdraw funds from the vault.
     - `normal_max`, `bonus_max`: Permitted ball ranges for the active round.
     - `ticket_price`: Cost per ticket (in USDC, 6 decimals).
     - `ongoing_epoch`: Current lottery round number.
     - `is_lords_pot_paused`: Emergency/rollover pause flag.
     - `relay_fee_base`, `relay_fee_per_ticket`: Fee parameters.
     - `max_tickets_per_purchase`: Maximum tickets allowed in a single purchase instruction (capped at 65).
     - `max_claim_amount`: Maximum allowed payout for a single claim.

2. **Vault Authority PDA**:
   - Seed: `[b"vault_authority"]`
   - Pure signer PDA (holds no data). Acts as the owner of the Solana Vault USDC token account.

3. **Vault USDC Token Account**:
   - Canonical Associated Token Account (ATA) owned by `vault_authority` PDA for the USDC mint.

---

### ⚙️ Solana Instructions Breakdown

- **`initialize`**: One-time initialization. Sets admin, treasury authority, fee recipient, ball ranges, ticket price, and creates the vault ATA.
- **`buy_ticket`**:
  - Validates that tickets are not empty and do not exceed `max_tickets_per_purchase`.
  - Validates that each ticket has exactly 5 normal balls, strictly sorted in ascending order with no duplicates, all `<= normal_max`.
  - Validates that bonus ball is `> 0` and `<= bonus_max`.
  - Calculates total ticket cost (`ticket_count * ticket_price`) and relay fee, transferring USDC from the buyer to the vault and fee recipient via SPL Token CPI.
  - Emits `TicketPurchaseEvent`.
- **`claim_winnings`**:
  - Gated by dual signatures: requires **both** the user and the admin to sign.
  - Verifies that the protocol is not paused.
  - Verifies that `amount <= max_claim_amount` and the vault has enough USDC.
  - Transfers USDC from the vault to the user's canonical ATA using `vault_authority` PDA seeds.
  - Emits `WinningsClaimedEvent`.
- **`pause_protocol` / `resume_protocol`**: Admin instructions to freeze purchases during epoch rollovers or emergencies.
- **`update_epoch`**: Admin instruction to update `normal_max` and `bonus_max` when Megapot changes ball ranges for a new round.
- **`set_admin`**: Rotates the admin hot key (requires current admin and new admin to sign).
- **`set_treasury_authority`**: Rotates the cold treasury key (requires current authority and new authority to sign).
- **`set_relay_config`**: Admin instruction to adjust fee rates and ticket bounds.
- **`withdraw_vault_funds`**: Treasury-only instruction to withdraw USDC from the vault for liquidity rebalancing or emergency evacuation.
- **`migrate_state`**: Reallocates and migrates the state PDA for zero-downtime upgrades.

---

### 🛡️ Solana Known Issues, Accepted Risks & Out-of-Scope Items

Please review these known architectural decisions before logging them as audit findings:

#### 1. `init_if_needed` on User USDC Account in `claim_winnings`
- **What auditors might flag**: `init_if_needed` in Anchor can sometimes be risky if an attacker can reinitialize or substitute an account.
- **Why it is safe & intentional**:
  In `claim_winnings`, the account initialized is strictly the canonical Associated Token Account (`user_usdc_account`), derived deterministically from `user` + `usdc_mint` via the SPL Associated Token Program. The `payer` is `user`. It is mathematically impossible to hijack ownership or redirect tokens. If the account exists, it is reused; if not, it is created so the user receives their winnings without pre-creating an ATA.

#### 2. Hot Admin Key vs. Cold Treasury Key Separation
- **What auditors might flag**: The `admin` key is centralized.
- **Why it is safe & intentional**:
  The protocol enforces a hard privilege split:
  - **Admin (Hot Key)**: Stored on the backend server to co-sign claim vouchers and manage pause/resume. **The admin key has NO authority to withdraw vault funds.**
  - **Treasury (Cold Key / Squads Multisig)**: `withdraw_vault_funds` strictly requires `treasury.key() == lords_pot_state.treasury_authority`.
  - Even a complete compromise of the backend server cannot drain the vault through withdrawals. Single claim payouts are capped by `max_claim_amount`.

#### 3. Dual Signatures on Claim Payouts
- **What auditors might flag**: Users cannot claim winnings trustlessly on Solana without an admin signature.
- **Why it is designed this way**:
  Megapot drawings happen on Base. Solana cannot read Base EVM state directly without an off-chain oracle or relayer. The backend grades winning tickets and provides an admin signature on the voucher. On-chain, the user counter-signs and funds can **only** land in the signer's own canonical USDC ATA.

#### 4. Requirement for `fee_recipient` ATA Pre-creation
- **What auditors might flag**: If a non-zero `relay_fee` is configured, `buy_ticket` will revert if the `fee_recipient` has not created their USDC ATA.
- **Why it is intentional**:
  The deployer/admin must ensure the `fee_recipient` ATA is initialized. Failing fast prevents fees from being sent to an uninitialized address or lost.

#### 5. 65 Tickets per Transaction Limit
- **What auditors might flag**: Why is `max_tickets_per_purchase` limited to 65?
- **Why it is intentional**:
  Solana has a strict 1232-byte MTU transaction size limit. 65 tickets take ~1,130 bytes, leaving just enough buffer for transaction headers and signatures.

#### 6. Code Checklist Markers (`// !`)
- **What auditors might flag**: Comments starting with `// !` in the program.
- **Why it is intentional**:
  These are operator checklist markers for deployment verification, not active code bugs.

---

### 🧪 Solana Build and Test Instructions

```bash
cd solana_smart_contracts

# Build the Anchor program
anchor build

# Run TypeScript integration tests
anchor test

# Verify feature-gated mint addresses (devnet vs mainnet USDC)
cargo test
cargo test --features mainnet-beta
```

---

### 🔒 Solana Security Invariants to Verify

1. **Treasury Protection**: Can any signer other than `treasury_authority` call `withdraw_vault_funds`?
2. **Ticket Validation**: Can an attacker submit out-of-order, duplicate, zero, or out-of-bounds lottery ball numbers?
3. **Claim Safety**: Can a user claim USDC without a valid admin signature, or claim more than `max_claim_amount`?
4. **Destination Binding**: Can an admin voucher be used to direct funds to any address other than the user who counter-signs?
5. **Reentrancy & Math**: Are all token transfers protected against math overflow and CPI reentrancy?

---
---

# Part 2: Base EVM Contract Audit Track (Solidity / Foundry)

> **Auditor Focus**: EVM router logic, OpenZeppelin inheritance, ERC721 receiver handling, order fulfillment idempotency, safe ERC20 transfers, and upstream Megapot interface integration.

### 📂 Base Scope
- Directory: [`base_smart_contracts/`](file:///Users/hail_the_lord/code/project/lordspot_audit/base_smart_contracts/)
- Primary Contract: [`base_smart_contracts/src/LordsPotBaseVault.sol`](base_smart_contracts/src/LordsPotBaseVault.sol)
- **Total Base nSLOC**: **165**
- Deployed Devnet Base Vault: [`0xE11b2B80fD954eA7e6a9B6e4E9A5Af406a2De4f5`](https://basescan.org/address/0xE11b2B80fD954eA7e6a9B6e4E9A5Af406a2De4f5)
- Upstream Megapot Contract on Base: [`0x3bAe643002069dBCbcd62B1A4eb4C4A397d042a2`](https://basescan.org/address/0x3bAe643002069dBCbcd62B1A4eb4C4A397d042a2)

---

### 🎯 Upstream Megapot Integration Details

The Base Vault interacts with Megapot's `IJackpot` interface:

```solidity
interface IJackpot {
    struct Ticket {
        uint8[] normals;
        uint8 bonusball;
    }

    function buyTickets(
        Ticket[] calldata _tickets,
        address _recipient,
        address[] calldata _referrers,
        uint256[] calldata _referralSplitBps,
        bytes32 _source
    ) external returns (uint256[] memory ticketIds);

    function claimWinnings(uint256[] calldata _userTicketIds) external;
}
```

1. **`buyTickets`**:
   - Purchases lottery tickets for the current drawing epoch.
   - Assigns minted ticket NFTs to the recipient address (`address(this)` / Base Vault).
2. **`claimWinnings`**:
   - Called after an epoch concludes to claim prize money for winning ticket NFT IDs.
   - Burns the ticket NFTs and pays out USDC directly to `msg.sender` (the Base Vault).

---

### ⚙️ Base Vault Architecture & Functions

`LordsPotBaseVault.sol` inherits OpenZeppelin's `Ownable`, `Pausable`, and `IERC721Receiver`.

- **`buyTickets`**:
  - Gated by `onlyRelayer` and `whenNotPaused`.
  - Checks `isOrderFulfilled[_orderId]`. If already processed, reverts with `OrderAlreadyProcessed()`.
  - Sets `isOrderFulfilled[_orderId] = true` (Checks-Effects-Interactions pattern).
  - Calls Megapot's `buyTickets`, specifying `address(this)` as the NFT recipient.
  - Emits `TicketsRouted`.
- **`claimWinnings`**:
  - Gated by `onlyRelayer`.
  - Takes an array of winning ticket NFT IDs and calls Megapot's `claimWinnings`.
  - Snapshots USDC balance before and after to emit `WinningsHarvested` with the net harvested amount.
- **`withdrawUsdc`**:
  - Gated by `onlyOwner`. Allows the cold owner to withdraw USDC for treasury rebalancing.
- **`pause` / `unPause`**:
  - Gated by `onlyOwner`. Pauses ticket purchases during emergencies.
- **`onERC721Received`**:
  - Standard ERC-721 handshake allowing the vault to receive Megapot ticket NFTs.
- **Configuration Setters**:
  - `setVaultRelayer`, `setVaultMegapotAddress`, `setVaultUsdcAddress`: Gated by `onlyOwner`.
  - `setVaultReferrerAddress`: Gated by `onlyReferrer`.

---

### 🛡️ Base Known Issues, Accepted Risks & Out-of-Scope Items

Please review these known architectural decisions before logging them as audit findings:

#### 1. Megapot Ticket NFTs Custodied in Base Vault
- **What auditors might flag**: Individual Solana players do not own their ticket NFTs on Base.
- **Why it is designed this way**:
  Solana users do not have EVM addresses or Base ETH for gas. The `LordsPotBaseVault` contract acts as an automated custodian that purchases the tickets, holds the NFTs, and claims prizes on behalf of the protocol.

#### 2. `claimWinnings` is Not Gated by `whenNotPaused`
- **What auditors might flag**: `claimWinnings` can be called even when the vault contract is paused.
- **Why it is intentional**:
  Pausing stops new ticket purchases (`buyTickets`). However, harvesting winning tickets from Megapot must **never be blocked**, even during an emergency pause, so prize claims do not expire. `claimWinnings` only brings USDC *into* the vault from Megapot; it can never extract funds out.

#### 3. Quadratic Gas & 15-Ticket Relay Chunking
- **What auditors might flag**: Why does the relayer submit tickets in batches of 15?
- **Why it is intentional**:
  Megapot's on-chain ticket processing has quadratic gas characteristics. Relaying in chunks of ~15 tickets provides the optimal gas efficiency per ticket.

#### 4. Idempotency via `isOrderFulfilled`
- **What auditors might flag**: Can a relayer replay an order and buy double tickets?
- **Why it is safe & intentional**:
  Every order has a unique `orderId` derived from the Solana purchase transaction. `isOrderFulfilled[_orderId]` guarantees that an order cannot be executed twice.

---

### 🧪 Base Build and Test Instructions

```bash
cd base_smart_contracts

# Build Solidity contracts
forge build

# Run Forge unit tests
forge test

# Run tests with execution trace
forge test -vvvv
```

---

### 🔒 Base Security Invariants to Verify

1. **Access Control**: Are `buyTickets` and `claimWinnings` strictly restricted to `relayer`?
2. **Withdrawal Safety**: Can any address other than `owner` withdraw USDC from the vault via `withdrawUsdc`?
3. **Replay Protection**: Does attempting to execute the same `_orderId` twice revert with `OrderAlreadyProcessed`?
4. **Token Safety**: Are all token approvals and transfers utilizing OpenZeppelin's `SafeERC20`?
5. **NFT Handling**: Does the contract accept ERC-721 ticket NFTs properly and prevent them from being locked or misdirected?
