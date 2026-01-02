# Architecture Overview

NonceArt implements NFT-like ownership of canvas tiles **without deploying any custom smart contract**.

## Component Architecture

```
                    WRITE PATH (transactions)
                    ─────────────────────────
┌──────────────┐      ┌───────────────────┐      ┌─────────────────────┐
│   Frontend   │─────▶│  x402 Facilitator │─────▶│   Base Blockchain   │
└──────────────┘      │   (gas sponsor)   │      │   (USDC contract)   │
       ▲              └───────────────────┘      └──────────┬──────────┘
       │                                                    │
       │                                                    │ events
       │                                                    ▼
       │                                         ┌─────────────────────┐
       │                                         │   The Graph         │
       │                                         │   Subgraph          │
       │                                         └──────────┬──────────┘
       │                                                    │
       │                                              polls │
       │                                                    ▼
       │              READ PATH (data)            ┌─────────────────────┐
       │              ───────────────             │      Backend        │
       │                                          │  (renders canvas,   │
       │◀──── fetches canvas/metadata ────────────│   serves metadata)  │
       │                                          └──────────┬──────────┘
       │                                                     │
       │                                              notify │
       │                                                     ▼
       │                                          ┌─────────────────────┐
       └─────────────── WebSocket ────────────────│     Realtime        │
                                                  │  (Cloudflare DO)    │
                                                  └─────────────────────┘
```

**Data Flow**:
1. Frontend submits signed authorizations to x402 Facilitator
2. Facilitator submits transactions to USDC contract (pays gas)
3. Subgraph indexes `AuthorizationUsed` events, decodes nonces, updates state
4. Backend polls Subgraph, renders canvas PNG, caches metadata
5. Backend notifies Realtime service when canvas updates
6. Realtime broadcasts to Frontend via WebSocket ("new data available")
7. Frontend fetches fresh canvas/metadata from Backend

## Component Responsibilities

| Component | Role |
|-----------|------|
| **USDC Contract** | Existing EIP-3009 implementation; stores nonces permanently |
| **x402 Facilitator** | Sponsors gas fees; submits `transferWithAuthorization` |
| **Subgraph** | Decodes nonces; validates operations; maintains canvas state |
| **Backend** | Polls Subgraph; renders canvas PNG; serves metadata; notifies Realtime |
| **Realtime** | Cloudflare Worker with Durable Object; broadcasts updates via WebSocket |
| **SDK** | Encodes operations into nonces; manages sessions |
| **Frontend** | UI; fetches from Backend; receives real-time updates via WebSocket |

The platform operates on a "store-on-transaction, read-from-backend" model:

- **Storage Layer**: Utilizes the nonce field (32 bytes) in USDC's `transferWithAuthorization` function (EIP-3009). Operations (`RESERVE`, `SET_PIXELS`, `BUY`, etc.) are encoded directly into these nonces.
- **State Machine**: The Graph subgraph acts as the consensus layer. It indexes USDC transactions, decodes the nonces, validates operations against the current state, and deterministically updates the canvas state.
- **Execution**: Free, gasless transfers facilitated by the x402 protocol, which relays these signed EIP-3009 authorizations to the blockchain.

---

## Stateless Smart Contract Design

### NFTs as Custom Contracts

Traditional NFT platforms deploy custom ERC-721 contracts that:
- Store ownership mappings in contract state
- Require contract deployment and auditing
- Must handle upgrades carefully (proxy patterns, migrations)
- Incur gas costs for state changes

### NonceArt's Approach

NonceArt leverages **x402** protocol and **existing USDC infrastructure** as its "contract":

1. **No custom contract**: Uses USDC's existing EIP-3009 `transferWithAuthorization`
2. **Nonce as data**: The 32-byte nonce field encodes all operations
3. **Subgraph as state machine**: The Graph interprets nonces and maintains derived state
4. **USDC as payment rail**: All operations require USDC payment (no zero-value txs)

### Why EIP-3009?

[EIP-3009](https://eips.ethereum.org/EIPS/eip-3009) defines `transferWithAuthorization`:

```javascript
function transferWithAuthorization(
    address from,
    address to,
    uint256 value,
    uint256 validAfter,
    uint256 validBefore,
    bytes32 nonce,           // ← NonceArt encodes operations here
    uint8 v, bytes32 r, bytes32 s
) external;
```

Key properties:
- **Arbitrary 32-byte nonce**: Per-authorizer uniqueness (not global)
- **Gasless execution**: x402 facilitators can submit on behalf of users
- **Permanent storage**: Nonces are stored in USDC contract forever
- **AuthorizationUsed event**: Exposes nonce for indexing

```javascript
event AuthorizationUsed(address indexed authorizer, bytes32 indexed nonce);
```

### Data Permanence

When a user executes a NonceArt operation:

1. User signs EIP-3009 authorization with encoded nonce
2. x402 facilitator submits transaction (pays gas)
3. USDC contract emits `AuthorizationUsed(authorizer, nonce)`
4. Nonce is permanently stored in USDC contract's `_authorizationStates` mapping
5. Blockchain stores the event log forever
6. Subgraph indexes and decodes the nonce

**Result**: Content data (images, metadata) persists as long as the Base blockchain exists.

---

## Operation Lifecycle

**End-to-end flow for a single operation**:

1. User creates operation via SDK/Frontend
2. Operation encoded into 32-byte nonce
3. User signs EIP-3009 authorization (off-chain)
4. Frontend sends signed authorization to x402 facilitator
5. Facilitator submits transaction to USDC contract (pays gas)
6. USDC contract validates signature and emits `AuthorizationUsed` event
7. The Graph subgraph indexes event
8. Subgraph decodes nonce and validates operation
9. State updated if valid (canvas, ownership, etc.)
10. Backend detects change on next poll, re-renders canvas PNG
11. Backend notifies Realtime service
12. Realtime broadcasts update to all connected Frontend clients
13. Frontend fetches fresh canvas/metadata from Backend

**Timing**:
- User signing: Instant (local wallet)
- Facilitator submission: ~20ms
- Blockchain preconfirmation: 200ms (Base)
- Subgraph indexing: ~1-3 seconds
- Backend poll + render: ~1-2 seconds
- WebSocket broadcast: Instant
- Frontend fetch: ~100-500ms

**Total latency**: ~2.5-10 seconds from signature to UI update.

---

## Payment Flows

### New Tile Purchase (Two-Phase)

**Phase 1: Reserve (1% to treasury)**
```
User → Facilitator → USDC (1% payment) → Treasury
                          ↓
                     AuthorizationUsed event
                          ↓
                      Subgraph creates Reservation entity
                          ↓
                   Backend polls, updates metadata
                          ↓
                   Realtime notifies Frontend
                          ↓
                   Frontend shows blinking overlay (15 min timer)
```

**Phase 2: Complete (98% to treasury)**
```
User → Facilitator → USDC (98% payment) → Treasury
                          ↓
                     AuthorizationUsed event
                          ↓
                      Subgraph converts Reservation → Lot
                          ↓
                   Backend polls, re-renders canvas PNG
                          ↓
                   Realtime notifies Frontend
                          ↓
                   Frontend shows owned tile
```

**Total**: 0.99 USDC per pixel for ownership. Additional 0.01 USDC per pixel for painting (total 1 USDC per pixel).

### Marketplace Purchase (Two-Phase)

**Phase 1: Buy Reserve (1% to treasury)**
```
Buyer → Facilitator → USDC (1% payment) → Treasury
                           ↓
                      AuthorizationUsed event
                           ↓
                   Subgraph creates BuyReservation (5 min lock)
                           ↓
                   Backend polls, updates metadata
                           ↓
                   Realtime notifies Frontend
                           ↓
                    Listing locked, countdown starts
```

**Phase 2: Complete (99% to seller)**
```
Buyer → Facilitator → USDC (99% payment) → Seller
                           ↓
                      AuthorizationUsed event
                           ↓
                   Subgraph transfers ownership
                           ↓
                   Backend polls, updates metadata
                           ↓
                   Realtime notifies Frontend
                           ↓
                    Buyer owns tile, listing removed
```

**Total**: 100% of listing price (1% treasury, 99% seller).

### Single-Phase Operations

**Painting, URL, Delegation, Transfer, etc.**:
```
User → Facilitator → USDC → Treasury (or Owner for tips)
                        ↓
                   AuthorizationUsed event
                        ↓
                    Subgraph updates state
                        ↓
                 Backend polls, re-renders if needed
                        ↓
                 Realtime notifies Frontend
                        ↓
                 Frontend reflects change
```

---

## State Machine

### Entities

**Canvas** (singleton):
- Total pixels painted
- Total tiles sold
- Revenue counters

**Lot** (tile ownership):
- Owner address
- Pixel data (RGB bytes)
- URL and description
- Delegates
- Listing (if for sale)

**Reservation** (pending tile purchase):
- User address
- Rectangle coordinates
- Amount paid
- Expiration timestamp

**Listing** (marketplace):
- Seller address
- Price per pixel
- Active buy reservation (if any)

**BuyReservation** (pending marketplace purchase):
- Buyer address
- Remaining payment
- Expiration timestamp (5 minutes)

**Delegate** (temporary painting permission):
- Lot reference
- Delegate address
- Validity window

**Transaction** (operation record):
- Blockchain transaction hash
- Opcode and arguments
- Success/failure
- Timestamp

### State Transitions

**Reservation lifecycle**:
```
[No Reservation]
     ↓ RESERVE Phase 1 (1%)
[ACTIVE Reservation] (15 min timer)
     ↓ RESERVE Phase 2 (98%)
[Ownership Complete] → Lot created
```

**Marketplace lifecycle**:
```
[Owned Tile]
     ↓ LIST (1 USDC)
[ACTIVE Listing]
     ↓ BUY Phase 1 (1%)
[RESERVED Listing] (5 min timer)
     ↓ BUY Phase 2 (99%)
[SOLD] → Ownership transfers
```

**Delegation lifecycle**:
```
[No Delegation]
     ↓ DELEGATE (flag=1, 1 USDC)
[Active Delegation] (time-limited)
     ↓ Expiry or DELEGATE (flag=0, 1 USDC)
[Revoked] → Delegate removed
```

---

## Coordinate System

**Canvas dimensions**: 2,400 × 1,675 pixels

**Origin**: Top-left corner is (0, 0)

**Coordinate bounds**:
- x: 0 to 2,400 (inclusive 0, exclusive 2,400)
- y: 0 to 1,675 (inclusive 0, exclusive 1,675)

**Tile rectangles use [inclusive, exclusive) bounds**:
- **Top-left (x1, y1)**: Inclusive (pixel is part of tile)
- **Bottom-right (x2, y2)**: Exclusive (pixel is NOT part of tile)

**Example**:
- Tile at (100, 200) with size 50×50
- Occupies pixels: x = 100-149, y = 200-249 (inclusive)
- Does NOT include pixels at x=150 or y=250
- Rectangle encoded as (100, 200, 150, 250)

**Why exclusive end coordinates?**
- Simplifies adjacency: Tile ending at x=150 (exclusive) can abut tile starting at x=150
- Eliminates edge-case bugs (no overlap ambiguity)
- Standard convention in graphics and array indexing

---

## Comparison with Traditional NFT Contracts

| Feature | Traditional ERC-721 | NonceArt |
|---------|---------------------|----------|
| **Custom contract** | Required | None |
| **Deployment cost** | Gas fees required | 0 |
| **Gas fees** | User pays ETH | x402 sponsors |
| **Upgradability** | Complex (proxies) | Update subgraph only |
| **State storage** | Contract storage | USDC nonces (permanent) |
| **Data permanence** | Until contract exists | Forever (blockchain events) |
| **Ownership proof** | Contract state | Subgraph derived from events |
| **Operation cost** | Gas + protocol | Protocol only (USDC) |
| **Censorship resistance** | High (contract immutable) | Very high (USDC + subgraph) |

**Advantages**:
- ✅ No contract deployment
- ✅ No gas fees for users
- ✅ Easier upgrades (subgraph + backend only)
- ✅ Permanent data storage
- ✅ Frontend never queries subgraph directly (reduced load, better caching)

**Tradeoffs**:
- Requires functional subgraph for state indexing
- Requires backend service for canvas rendering and metadata
- Rebuilding state requires re-indexing blockchain events
- USDC contract must remain operational

---

**Next**: Learn about data encoding → [Data Encoding](11-technical/data-encoding.md)
