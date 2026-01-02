# Marketplace

The NonceArt marketplace enables peer-to-peer tile sales with trustless escrow via the subgraph.

## Listing Your Tile for Sale

**Requirements**:
- Must own the tile
- Tile cannot have active buy reservation (wait for expiry or completion)

**How to list**:
1. Click your tile on canvas
2. Pinned tooltip appears
3. Click "List for Sale" button
4. List dialog opens

**List dialog**:
- **Price per pixel**: Enter amount in USDC (minimum 0.01 USDC per pixel)
- **Total price**: Calculated automatically (area × price per pixel)
- **Listing fee**: 1 USDC flat fee (paid upfront to treasury)
- **Terms of Service**: Must accept ToS
- **Preview**: Shows tile info and total price

**Pricing examples**:
- 100-pixel tile at 2 USDC/pixel = 200 USDC total
- 1,000-pixel tile at 1.5 USDC/pixel = 1,500 USDC total
- 500-pixel tile at 0.01 USDC/pixel = 5 USDC total (minimum)

**Minimum price**: 0.01 USDC per pixel
- Prevents spam listings
- Ensures buyers pay meaningful amount

**Listing fee**: 1 USDC
- Paid to NonceArt treasury
- Non-refundable (paid even if tile doesn't sell)
- Covers protocol fees

**Confirmation**:
1. Enter price per pixel
2. Review total price and fee
3. Accept Terms of Service
4. Click "List for Sale" button
5. Sign transaction
6. Wait for confirmation (10-30 seconds)

**After listing**:
- Tile tooltip shows listing status and price when clicked
- Other users can view listing details by clicking the tile
- You remain owner until someone buys

**Restrictions while listed**:
- ❌ Cannot edit tile (paint, set URL, etc.)
- ❌ Cannot transfer tile
- ❌ Cannot delegate tile
- ✅ Can delist anytime (if no active buy reservation)

**Updating price**: Must delist, then relist with new price (costs another 1 USDC listing fee).

## Buying Listed Tiles

### Finding Tiles for Sale

**On canvas**: Click tiles to check if they're listed for sale

**In tooltip**: Shows listing status and price if tile is listed

**Activity feed**: Watch for "SELL" transactions (new listings)

### Viewing Listing Details

Click listed tile to see:
- Current price (total and per pixel)
- Tile size and position
- Current owner (seller)
- Painted content preview
- URL and description (if set)
- "Buy" button

### Two-Phase Purchase

Like reserving new tiles, buying listed tiles uses two-phase payment to prevent race conditions.

#### Phase 1: Reserve (1% to treasury)

**Cost**:
- 1% of listing price
- Paid to NonceArt treasury (not seller)

**Examples**:
- 200 USDC tile: 2 USDC reserve
- 1,500 USDC tile: 15 USDC reserve
- 10,000 USDC tile: 100 USDC reserve

**What happens**:
1. Payment settles on blockchain
2. Listing locked for 5 minutes (not 15 like new reservations)
3. No one else can buy during this window
4. Seller cannot delist during this window
5. Timer starts counting down

**Terms of Service**: Must accept ToS before reserving.

**Rectangle requirement**: Buy operation encodes exact listing rectangle. Verifies you're buying the correct tile (provides explicit buyer intent).

**Visual feedback**: Tile tooltip shows "Reserved by You" with countdown timer.

#### Phase 2: Complete Purchase (99% to seller)

**Cost**:
- Remaining 99% of listing price
- Paid directly to seller

**Examples**:
- 200 USDC tile: 198 USDC complete (200 total)
- 1,500 USDC tile: 1,485 USDC complete (1,500 total)
- 10,000 USDC tile: 9,900 USDC complete (10,000 total)

**Payment breakdown**:
- 1% to treasury (reserve)
- 99% to seller (complete)

**How to complete**:
1. Click reserved tile (within 5-minute window)
2. Tooltip shows "Complete Purchase" section
3. Review remaining payment amount
4. Click "Complete Purchase" button
5. Sign transaction
6. Wait for confirmation

**No partial payments**: Must pay full 99% in single transaction (unlike reservations which allow partial payments).

**Completion**:
- Ownership transfers to you
- Seller receives 99% of price
- Listing removed
- Tile is yours (can edit, transfer, relist)

**What transfers**: Same as regular transfers—ownership, pixels, URL, and description are preserved; delegations are cleared. See [Transferring Ownership](06-tile-management.md#transferring-ownership) for details.

### Buy Reservation Expiry

**Time limit**: 5 minutes (shorter than new tile reservations)

**If expired before completion**:
- Reserve payment (1%) is forfeited (no refund)
- Listing unlocked (others can buy)
- You can reserve again if desired

**Visual feedback**:
- Countdown timer in tooltip
- Tooltip shows "EXPIRED" if time runs out
- "Complete Purchase" button disabled after expiry

### Restrictions

**Seller cannot**:
- Delist while buy reservation is active
- Edit tile while listed
- Transfer tile while listed

**Buyer cannot**:
- Buy own listing (seller address excluded)
- Buy if insufficient USDC balance
- Make partial payments on phase 2

**Platform cannot**:
- Change listing price
- Cancel listings
- Interfere with sales

## Delisting

**Purpose**: Remove listing and make tile editable again.

**Requirements**:
- Must own the tile
- No active buy reservation (wait for expiry or completion)

**How to delist**:
1. Click your listed tile
2. Tooltip shows "Delist" button
3. Click "Delist"
4. Delist dialog opens

**Delist dialog**:
- Shows tile info
- Delist fee: 0.01 USDC (paid to treasury)
- Terms of Service acceptance

**Cost**: 0.01 USDC flat fee
- Much cheaper than listing (1 USDC)
- Covers protocol fees

**Confirmation**:
1. Review delist fee
2. Accept Terms of Service
3. Click "Delist" button
4. Sign transaction
5. Wait for confirmation

**After delisting**:
- Listing status removed from tooltip
- Tile returns to normal
- You can edit, transfer, or delegate
- Can relist later (costs 1 USDC listing fee again)

**Why delist?**
- Want to edit tile (paint, change URL)
- Changed mind about selling
- Want to transfer to specific address (off-platform sale)
- Listing price no longer appropriate

## Marketplace Statistics

### Activity Feed

**Location**: Sidebar menu → Activity section

**Marketplace transaction types**: SELL, BID, SOLD, CANCEL. See [Activity Feed](03-navigating-canvas.md#activity-feed) for the complete list of transaction types and color coding.

## Best Practices

### For Sellers

1. **Price competitively**: Check similar tiles' prices
2. **Budget listing fee**: 1 USDC per listing (non-refundable)
3. **Finalize content first**: Cannot edit while listed
4. **High-quality art**: Better content attracts higher prices
5. **Set URL and description**: Help buyers understand your tile
6. **Monitor buy reservations**: Check Activity feed for interest
7. **Be patient**: Tiles may take time to sell

### For Buyers

1. **Inspect carefully**: Click tile to see painted content
2. **Verify price**: Check USDC per pixel and total cost
3. **Budget correctly**: Reserve (1%) + Complete (99%) = Total
4. **Complete promptly**: Only 5 minutes to complete purchase
5. **Check URL/description**: See what you're buying
6. **Watch Activity feed**: Find newly listed tiles
7. **Be decisive**: Others can buy while you hesitate

## Marketplace Lifecycle

**State transitions**:

```
Owned tile
    ↓ LIST (1 USDC) ↓
ACTIVE listing
    ↓ BUY reserve (1%) ↓
RESERVED listing (5 min)
    ↓                    ↓
    ↓ BUY complete (99%) ↓ (expires)
    ↓                    ↓
SOLD                  ACTIVE
    ↓
Buyer owns
```

**Delist path**:
```
ACTIVE listing
    ↓ DELIST (0.01 USDC) ↓
Owned tile (not listed)
```

**Cannot delist during RESERVED state** (must wait for buy completion or expiry).

---

**Next**: Learn about tipping and community features → [Community Features](08-community-features.md)
