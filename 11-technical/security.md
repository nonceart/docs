# Security Model

Security considerations, trust assumptions, and attack mitigations for NonceArt.

## Economic Security

### Attack Vectors and Mitigations

| Attack | Mitigation |
|--------|-----------|
| **Reservation spam** | Minimum 1 USDC deposit, non-refundable |
| **Front-running** | 1% stake limits loss; WebSocket detection |
| **Griefing (fake reservations)** | 15-minute expiry; lost deposit |
| **Nonce collision** | Per-user uniqueness; random 16-bit local nonce |
| **Delegation abuse** | 1 USDC cost; time-limited validity |

### Cost Structure Defense

All operations require payment to prevent spam and abuse:

| Operation | Cost (USDC) | Purpose |
|-----------|-------------|---------|
| `RESERVE` | max(1, area × 0.01) | Prevent reservation spam |
| `SET_PIXELS` | N × 0.01 | Prevent pixel spam |
| `SET_PIXELS_RL` | pixels × 0.01 | Prevent pixel spam |
| `TRANSFER` | 0.10 | Discourage rapid ownership changes |
| `SET_URL_FRAGMENT` | 0.01 | Prevent metadata spam |
| `CLEAR_URL` | 0.01 | Prevent metadata spam |
| `DELEGATE` | 1 | Prevent delegation abuse |
| `SET_DESCRIPTION_FRAGMENT` | 0.01 | Prevent metadata spam |
| `CLEAR_DESCRIPTION` | 0.01 | Prevent metadata spam |
| `LIST` | 1 | Prevent listing spam |
| `DELIST` | 0.01 | Discourage rapid list/delist cycles |
| `BUY` (reserve) | 1% of price | Prevent marketplace spam |
| `BUY` (complete) | 99% of price | Actual purchase |
| `TIP` | ≥ 0.01 | Prevent tip spam |

**Key Principle**: No zero-value operations allowed. Every action has economic weight.

## Trust Assumptions

### Component Trust Model

| Component | Trust Assumption | Risk Level |
|-----------|------------------|------------|
| **USDC Contract** | Audited, battle-tested; stores nonces correctly | Low |
| **Base Blockchain** | Standard L2 security (derived from Ethereum) | Low |
| **x402 Facilitator** | Can censor but not forge; trustless verification possible | Low |
| **Subgraph** | Open source; anyone can run; data derived from chain | Low |
| **Frontend** | Can be rebuilt from chain data; subgraph is source of truth | Low |

### USDC Contract

- **Trust Level**: High
- **Why**: Audited implementation of EIP-3009, widely used in DeFi
- **Storage Guarantee**: Nonces stored in `_authorizationStates` mapping forever
- **Event Guarantee**: `AuthorizationUsed` events emitted on-chain

### Base Blockchain

- **Trust Level**: High
- **Why**: L2 derived from Ethereum security
- **Data Availability**: All transaction data available on L1
- **Permanence**: Blocks cannot be reversed after finality

### x402 Facilitator

- **Trust Level**: Low (custodial element)
- **Powers**:
  - ✅ Can submit transactions on behalf of users
  - ✅ Can censor (refuse to relay transactions)
  - ❌ Cannot forge signatures (user signs EIP-712 message)
  - ❌ Cannot spend user funds (authorization is time-limited and specific)
- **Mitigation**: Users can verify facilitator behavior; alternative facilitators possible

### The Graph Subgraph

- **Trust Level**: High (verifiable)
- **Why**: Open source; deterministic; anyone can run their own
- **Verification**: Subgraph code is public; results can be independently verified
- **Decentralization**: Multiple hosted service providers available

### Frontend

- **Trust Level**: Medium (convenience layer)
- **Why**: Can be malicious or compromised
- **Mitigation**: All data can be reconstructed from blockchain
- **Independence**: Users can interact with protocol via SDK directly

## Ownership Enforcement

### Lot Ownership

**ID Format**: `"{x1}-{y1}"` (top-left corner)

**Ownership Proof**:
```
Lot.owner field in subgraph (derived from RESERVE operation)
```

**Transfer Validation**:
```
TRANSFER operation:
  1. Check lot exists
  2. Verify lot.owner == authorizer
  3. Verify rectangle matches lot bounds exactly
  4. Verify lot is not listed (ACTIVE or RESERVED status)
```

**Delegate Validation**:
```
Delegate permissions:
  1. Only owner can `DELEGATE` (no sub-delegation)
  2. Delegates time-limited via validAfter/validBefore
  3. Lazy cleanup removes expired delegations
  4. Delegates cannot `TRANSFER` or `LIST`
```

### Reservation Ownership

**ID Format**: `"{authorizer}-{x1}-{y1}-{x2}-{y2}"`

**Overlap Prevention**:
```
On RESERVE Phase 1:
  1. Check all existing lots (no overlap allowed)
  2. Check all active reservations (exclude same user)
  3. User can overlap own expired reservations
  4. First transaction in block (by logIndex) wins
```

**Expiry Enforcement**:
```
Reservation expires after `TIME_WINDOW` (15 minutes)
Additional payments rejected if expired
Lazy cleanup removes expired reservations periodically
```

### Listing Ownership

**ID Format**: `"{lotId}"` (same as lot)

**Validation**:
```
On LIST:
  1. Check lot.owner == authorizer
  2. Check no active BuyReservation exists

On DELIST:
  1. Check lot.owner == authorizer
  2. Check BuyReservation expired (if RESERVED)

On BUY:
  1. Check listing.status == ACTIVE or RESERVED
  2. Check seller still owns lot
  3. Check buyer != seller
```

## Payment Validation

### Payment Enforcement

**All Operations**:
```
1. Extract value from Transfer event
2. Validate value >= required amount
3. Reject if insufficient (funds forfeited)
```

**Routing Validation**:
```
Most operations: to == TREASURY_ADDRESS
BUY complete: to == listing.seller
TIP: to == lot.owner
```

**Defensive Checks**:
```
// Verify payment recipient matches expectation
if (to != expectedRecipient) {
  tx.failureReason = "Invalid recipient"
  return
}

// Verify payment amount
if (value < requiredAmount) {
  tx.failureReason = "Insufficient payment"
  return
}
```

### Funds Forfeiture

**Non-Refundable Operations**: All operations are non-refundable by design.

**Why**:
- Prevents griefing (reserve then cancel)
- Simplifies protocol (no refund logic)
- Discourages spam (economic cost to fail)

**When Funds Forfeited**:
- Invalid operation encoding
- Overlap detected (`RESERVE`)
- Insufficient payment
- Unauthorized operation (not owner/delegate)
- ToS not accepted

**Treasury Accumulation**:
```
canvas.totalForfeitedFunds += value
```

## Nonce Collision Handling

### Per-User Uniqueness

**EIP-3009 Guarantee**: Nonces are unique per `authorizer` address, not globally.

**Format**:
```
| Opcode (1) | LocalNonce (2) | Arguments (29) |
```

**LocalNonce Space**: 0-65535 (16-bit)

### Collision Probability

**Random Nonce Strategy**:
```typescript
function generateRandomNonce(): number {
  return Math.floor(Math.random() * 65536);
}
```

**Birthday Paradox**:
- 100 operations: ~0.076% collision chance
- 200 operations: ~0.3% collision chance
- 1000 operations: ~7.6% collision chance

**Acceptable**: For typical user (< 100 operations), risk is negligible.

### Collision Detection

**Symptoms**:
- USDC contract reverts with `"FiatTokenV2: authorization is used or canceled"`
- x402 facilitator returns error

**Recovery**:
1. SDK detects collision error
2. Generate new random nonce
3. Re-sign and submit
4. User informed of retry

**Frontend UX**:
```
Error: "Transaction failed due to nonce collision. Retrying automatically..."
[Automatic retry with new nonce]
```

## Delegate Permissions and Limits

### Permission Matrix

| Operation | Owner | Delegate | Notes |
|-----------|-------|----------|-------|
| `SET_PIXELS` | ✅ | ✅ | Painting operations |
| `SET_PIXELS_RL` | ✅ | ✅ | Painting operations |
| `SET_URL_FRAGMENT` | ✅ | ✅ | Metadata operations |
| `CLEAR_URL` | ✅ | ✅ | Metadata operations |
| `SET_DESCRIPTION_FRAGMENT` | ✅ | ✅ | Metadata operations |
| `CLEAR_DESCRIPTION` | ✅ | ✅ | Metadata operations |
| `TRANSFER` | ✅ | ❌ | Owner only |
| `DELEGATE` | ✅ | ❌ | Owner only (no sub-delegation) |
| `LIST` | ✅ | ❌ | Owner only |
| `DELIST` | ✅ | ❌ | Owner only |

### Time-Limited Validity

**Validation Window**:
```
delegate.validAfter <= currentTime < delegate.validBefore
```

**Extracted From**:
- EIP-3009 `transferWithAuthorization` parameters
- SDK sets appropriate window when creating delegation
- Typical: 1 hour to 30 days

**Lazy Cleanup**:
```
On delegate permission check:
  If currentTime >= delegate.validBefore:
    delete Delegate(delegate.id)
    return false
```

### No Sub-Delegation

**Enforcement**:
```
On DELEGATE:
  Verify lot.owner == authorizer
  If authorizer is delegate (not owner):
    tx.failureReason = "Delegates cannot delegate"
    return
```

**Why**: Prevents permission escalation and complexity.

## Attack Scenarios and Mitigations

### Scenario 1: Reservation Front-Running

**Attack**:
1. Attacker monitors mempool for `RESERVE` transactions
2. Submits own `RESERVE` with higher gas price
3. Attacker's transaction mined first
4. Victim loses 1% deposit

**Mitigations**:
1. **Low Risk**: Only 1% at stake (vs 100% in single-phase)
2. **WebSocket Detection**: Frontend warns of conflicting pending transactions
3. **Block Ordering**: Within same block, first logIndex wins (not gas price)
4. **Economic Disincentive**: Attacker pays 1% for each griefing attempt

### Scenario 2: Nonce Collision Griefing

**Attack**:
1. Attacker generates nonce collision for victim's operation
2. Submits invalid operation using collision nonce
3. Victim's operation fails (nonce already used)

**Mitigations**:
1. **Large Nonce Space**: 65536 possible values per user
2. **Random Generation**: Collision unlikely in normal usage
3. **Automatic Retry**: SDK regenerates and retries on collision
4. **Economic Cost**: Attacker wastes money on failed operations

### Scenario 3: Delegate Abuse

**Attack**:
1. Malicious delegate paints offensive content
2. Owner unable to immediately revoke (time-limited)

**Mitigations**:
1. **High Cost**: 1 USDC per delegation (owner pays, considers trust)
2. **Owner Can Revoke**: Owner can `DELEGATE` with flag=0 anytime
3. **Repaint**: Owner or new delegate can `SET_PIXELS` to overwrite

### Scenario 4: Expired Reservation Overlap

**Attack**:
1. User A reserves area, expires without completing
2. User B reserves same area
3. User A completes payment after expiry
4. Both think they own the area

**Mitigations**:
1. **Completion Check**: Phase 2 `RESERVE` re-checks overlaps before creating lot
2. **Status Tracking**: Expired reservations marked `EXPIRED`, not eligible for completion
3. **Forfeiture**: If overlap detected at completion, funds forfeited

### Scenario 5: Marketplace Race Condition

**Attack**:
1. Seller lists lot
2. Buyer reserves with `BUY`
3. Seller transfers lot before buyer completes
4. Buyer pays 99% to non-owner

**Mitigations**:
1. **Transfer Block**: Cannot `TRANSFER` while listing is `ACTIVE` or `RESERVED`
2. **Ownership Verification**: `BUY` complete checks seller still owns lot
3. **Expiry**: BuyReservation expires in 5 minutes (shorter than lot reservation)
4. **Defensive Checks**: Multiple ownership validations before transfer

## Content Permanence

### On-Chain Data (Immutable)

All content is permanently stored on-chain:
- Pixel data (`lot.pixelData`)
- URLs (`lot.url`)
- Descriptions (`lot.description`)
- Ownership history (transactions)

**Key Principle**: Censorship-resistant storage. Anyone can build alternative frontends with different display policies.

---

