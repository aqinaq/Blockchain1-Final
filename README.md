# Blockchain Technologies 1
## SE-2426 Akbope Bakytkeldy, Erasyk Tortkara, Aron Davlyudov
## Blockchain based Ticketing System

## 1. Scope & Goals
This project implements **blockchain-based ticketing system** running **only on an Ethereum test network** (Sepolia/Holesky) or local chain. Payments use **test ETH only** and have **no real-world monetary value**. Users are identified by their **wallet address via MetaMask**.

Primary goals:
- Issue tickets on-chain so ownership can be verified without a central database
- Allow organizers to create events and sell tickets
- Allow door staff to verify (check-in) tickets reliably
- Keep rules simple, auditable, and testnet-safe

Out of scope:
- Fiat payments, credit cards, KYC, real-world pricing guarantees
- Complex seating maps, dynamic pricing, marketplace integrations

## 2. Definitions

### Ticket
**ticket** is an on-chain token representing admission to **exactly one event**, optionally tied to a **zone/seat**, and owned by a wallet address.

ticket MUST include:
- **eventId**: identifies the event it belongs to
- **zoneId** (or seat/section label): identifies seating/entry zone (e.g., “GA”, “VIP”, “Section A”)
- **tokenId**: unique identifier of the ticket token
- **owner**: current wallet address that holds the ticket
- **checkedIn**: boolean status (initially false)
- **valid**: true unless canceled/refunded/voided

Implementation note: tickets are best represented as an **ERC-1155** token where each `(eventId, zoneId)` can be a token type, or as **ERC-721** where each ticket is unique. Either is acceptable, but uniqueness must be preserved by `(tokenId)` and linkable to `(eventId, zoneId)`.

### Event
An **event** is a record created by an organizer that defines:
- eventId
- organizer wallet address
- name/title
- start time (timestamp)
- optional sales end time
- zones with supply (capacity)
- ticket price per zone (in test ETH)
- refund policy flags (optional)

## 3. Roles
- **Organizer**: creates events, configures zones/supply/prices, can cancel event (if enabled)
- **Buyer / Attendee**: buys tickets, holds tickets, optionally transfers tickets, presents ticket for check-in
- **Verifier (Door Staff)**: checks ticket validity and marks it checked-in. This is a wallet address authorized by the organizer

Wallet-based identity: all roles are represented by wallet addresses (MetaMask).


## 4. Actions & Rules
### A) Create Event
**Who:** Organizer  
**Inputs:** event metadata, zone list, supply per zone, price per zone, (optional) sales end time, verifier list  
**Rules:**
- Only the creating wallet becomes the organizer of that event.
- Each zone has a fixed **maxSupply** and **remainingSupply**.
- Event configuration is immutable after creation, except:
  - organizer may add/remove verifiers (recommended)
  - organizer may cancel event (optional feature)

### B) Buy Ticket
**Who:** Any wallet  
**Inputs:** eventId, zoneId, quantity, payment in test ETH  
**Rules:**
- Purchase only allowed if:
  - event exists
  - current time is before sales end time (if defined)
  - remainingSupply in the chosen zone is sufficient
  - exact payment >= `price(zoneId) * quantity` (overpayment allowed or refunded per implementation; default: reject if not exact)
- On success:
  - mint `quantity` tickets to buyer wallet
  - decrement remainingSupply
  - record sale totals for the event (for reporting)

### C) Transfer Ticket (Optional)
**Who:** Current ticket owner  
**Inputs:** ticket tokenId (or token type + amount), recipient address  
**Rules:**
- If enabled, transfers are allowed only when:
  - ticket is valid
  - ticket is NOT checked-in
- Transfers do not change eventId/zoneId.
- If disabled, tickets are non-transferable (recommended if anti-scalping is desired).

### D) Check-in / Verify Ticket at the Door
**Who:** Authorized verifier (door staff)  
**Inputs:** ticket tokenId (or token type) + owner address presented  
**Rules:**
- A ticket is valid for check-in if:
  - it belongs to the intended event
  - it is owned by the presented wallet (on-chain owner check)
  - it is valid (not canceled/refunded/void)
  - it has not already been checked-in
- On successful verification:
  - set `checkedIn = true` (or burn the ticket; default: mark checkedIn)
- After check-in:
  - ticket cannot be checked in again
  - if transfers are enabled, checked-in tickets cannot be transferred

Verification UX note: door staff uses a verifier wallet connected to the dApp. The attendee presents a QR code containing `(eventId, tokenId)` or `(eventId, zoneId, tokenId)` and optionally owner address.

### E) Refund / Cancel (Optional)
Two optional modes (choose one for implementation):

**Mode 1: Organizer Cancel Event**
- Organizer can cancel before event start.
- All tickets become invalid.
- Buyers can claim refund of paid test ETH.

**Mode 2: Buyer Refund Window**
- Buyers can request refund before a cutoff time.
- Refunded tickets become invalid (burned or flagged invalid).
- remainingSupply increases accordingly (optional; default: increase supply).

Default if not implemented: no refunds; tickets remain valid until event start


## 5. Constraints
- **Network:** Sepolia/Holesky or local only. No mainnet deployment.
- **Value:** test ETH only; tokens/tickets have no real monetary value.
- **Identity:** wallet address via MetaMask; no usernames/passwords.
- **Security basics:**
  - prevent double check-in
  - prevent buying beyond supply
  - prevent unauthorized check-in (verifier authorization)
  - handle re-entrancy for payable/refund paths (implementation detail)

## 6. Minimal Acceptance Criteria
- Create event with at least 2 zones and fixed supplies.
- Buy ticket(s) using MetaMask and test ETH.
- View ownership of ticket(s) by wallet.
- Verify/check-in ticket using authorized verifier wallet.
- Checked-in ticket cannot be checked in twice.
- Works end-to-end on a testnet or local chain.
