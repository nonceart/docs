# Subgraph State Machine

Complete state machine logic for The Graph subgraph.

## Event Processing Flow

```
1. Listen for AuthorizationUsed events from USDC contract

2. Find corresponding Transfer event in same transaction

3. Extract transfer data:
   from = Transfer.topics[1] (authorizer)
   to = Transfer.topics[2] (recipient)
   value = Transfer.data (USDC amount)

4. Validate:
   - from == AuthorizationUsed.authorizer
   - For most ops: to == TREASURY_ADDRESS
   - For `BUY` complete: to == seller
   - For `TIP`: to == lot owner
   - value >= threshold (spam filter)

5. Decode nonce:
   opcode = nonce[0]
   localNonce = nonce[1] << 8 | nonce[2]
   args = nonce[3..31]

6. Create Transaction entity (always, even if operation fails)

7. Dispatch to operation handler based on opcode

8. Update canvas stats (lastUpdatedBlock, lastUpdatedTimestamp)
```

## GlobalIndex Singleton

**Purpose**: Provides O(1) access to all lots, active reservations, and active listings for overlap detection.

**Entity**:
```graphql
type GlobalIndex @entity {
  id: ID!  # "global"
  lots: [Lot!]! @derivedFrom(field: "globalIndex")
  activeReservations: [Reservation!]! @derivedFrom(field: "globalIndex")
  activeListings: [Listing!]! @derivedFrom(field: "globalIndex")
  lastCleanupTimestamp: BigInt!
}
```

**Linking Entities**:
```
// Active entities
lot.globalIndex = "global"           // Lot appears in GlobalIndex.lots
reservation.globalIndex = "global"   // Appears in activeReservations
listing.globalIndex = "global"       // Appears in activeListings

// Removing from GlobalIndex
reservation.globalIndex = null       // When `COMPLETED`/`EXPIRED`
listing.globalIndex = null           // When `SOLD`/`DELISTED`
```

**Usage**:
```
globalIndex = getGlobalIndex()

// Check overlaps
allLots = globalIndex.lots.load()
For each lot in allLots:
  If rectanglesOverlap(newRect, lot):
    return "overlap detected"

activeReservations = globalIndex.activeReservations.load()
For each res in activeReservations:
  If res.user == excludeUser: continue
  If res.expiresAt <= now: continue
  If rectanglesOverlap(newRect, res):
    return "overlap detected"
```

## String Fragment Assembly

### Fragment Storage

```graphql
type StringFragment @entity {
  id: ID!  # "{lotId}-{URL|DESCRIPTION}-{localId}"
  lot: Lot!
  type: StringType!  # URL or DESCRIPTION
  localId: Int!  # 0-3 for URL, 0-9 for DESCRIPTION
  content: String!  # Up to 24 bytes
  transaction: Transaction!  # Creation transaction
}
```

**Example of String Building**:
```
Fragments: 0="http://ex", 2="le.com", 3="" (null)
Result: "http://ex[...]le.com"

Fragments: 0="http://example.com\0", 1="ignored"
Result: "http://example.com"
```

## Reservation Lifecycle

### States

```
enum ReservationStatus {
  ACTIVE      // Within `TIME_WINDOW`, not fully paid
  EXPIRED     // Past `TIME_WINDOW`, not fully paid
  COMPLETED   // Converted to Lot
  FORFEITED   // Rejected (overlap, invalid)
}
```

### Transitions

```
ACTIVE ──(paid >= cost)──▶ COMPLETED ──▶ Lot created
       │
       └──(expiresAt <= now)──▶ EXPIRED
       │
       └──(overlap at completion)──▶ FORFEITED
```

## Listing Lifecycle

### States

```
enum ListingStatus {
  ACTIVE    // Available for purchase
  RESERVED  // Buyer has active reservation (5-min lock)
  SOLD      // Successfully sold
  DELISTED  // Seller cancelled
}
```

### Transitions

```
LIST ──▶ ACTIVE ──(`BUY` reserve)──▶ RESERVED ──(`BUY` complete)──▶ SOLD
                │                      │
                └──(`DELIST`)──▶ DELISTED
                                       │
                                       └──(expires)──▶ ACTIVE
```

### Buy Reservation Lifecycle

```
enum BuyReservationStatus {
  ACTIVE    // Awaiting completion
  COMPLETED // Buyer paid, ownership transferred
  EXPIRED   // Past expiry, buyer lost 1%
}
```

## Race Condition Mitigation

NonceArt employs a multi-layered approach to handle blockchain race conditions.

### Layer 1: Two-Phase Purchase Protocol

**The Problem**: If two users try to buy overlapping areas simultaneously, one transaction will succeed and one will fail. With single-phase purchases, the losing user could lose significant funds.

**The Solution**: Split purchase into two phases:

```
Phase 1: `RESERVE` (1% stake)
├── User pays max(1 USDC, area × 0.01 USDC)
├── Creates 15-minute reservation lock
├── Non-refundable (prevents griefing)
└── Only non-overlapping reservation with earliest block timestamp wins

Phase 2: COMPLETE (98% remaining, 99% total ownership)
├── User has exclusive lock on the area
├── Can pay in multiple transactions
└── When paid ≥ cost (99% of $1/pixel), lot is created
```

The operation `BUY` also uses two-phase (1% to treasury, 99% to seller = 100% of listing price).

**Risk Reduction**: User risks only 1% in race condition vs. 100%.

**Example**: 400-pixel lot
- Single-phase risk: 400 USDC (full price)
- Two-phase risk: 4 USDC (1% reserve)

### Layer 2: Real-Time WebSocket Subscription

The frontend monitors blockchain events via direct WebSocket connection to detect conflicts **before subgraph confirmation**. The frontend establishes a WebSocket connection (via viem) to the RPC provider. It subscribes to pending `Transfer` events targeting the `TREASURY_ADDRESS`. Logic:
1. It intercepts incoming transactions in the mempool or identifying confirmed blocks before the subgraph indexes them.
2. It decodes the nonces of these transactions locally.
3. If it detects a `RESERVE` or `BUY` operation that conflicts with the user's intent, it warns the user immediately or disables the purchase button.

This drastically reduces the window of vulnerability where a user might submit a transaction that is destined to fail due to a race condition

**Timeline**:
```
T+0ms:    User A submits `RESERVE`
T+500ms:  WebSocket receives Transfer event
T+600ms:  Frontend shows "pending" overlay (dashed border)
T+3000ms: Subgraph confirms reservation
T+3100ms: Overlay switches to confirmed state (solid blinking)
```

**Visual Indicators**:
- **Pending**: Dashed blue border with dashed border (marching ants animation)
- **Confirmed**: Blinking overlay with time-based colors
  - Green: 10-15 minutes remaining
  - Yellow: 5-10 minutes remaining
  - Red: 0-5 minutes remaining

**Auto-Cleanup**: Pending operations removed after 30 seconds or when backend confirms.

### Layer 3: Subgraph Block Ordering

Within a single block, transactions are processed in log index order. If valid and non-overlapping, subgraph creates a Reservation entity with an expiry. This "locks" the area for the user. If multiple users attempt to reserve the same area in the same block, the first transaction (earliest logIndex) wins. Losers only lose the 1% deposit fee, not the full purchase price.

**Implementation**:
```
Transaction.logIndex is used for ordering within a block

For each `RESERVE` in block (ordered by logIndex):
  If hasOverlap():
    Reject (funds forfeited)
  Else:
    Create Reservation (lock acquired)
```

### SDK Nonce Management

Each operation requires a unique nonce. The SDK uses random 16-bit local nonces, which should be sufficient given that each operation takes other arguments.

**Implementation**:
```typescript
function generateRandomNonce(): number {
  return Math.floor(Math.random() * 65536);  // 0-65535
}
```

**Collision Probability** (Birthday Paradox):
- 100 operations: ~0.076% chance
- 200 operations: ~0.3% chance
- Acceptable for typical usage

On collision error, the SDK/frontend prompts retry with new random nonce.

---

**Next**: [Security Model →](11-technical/security.md)
