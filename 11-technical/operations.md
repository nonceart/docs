# Operation Specifications

Complete technical details for all NonceArt operations.

## `0x00`: `RESERVE`

**Purpose**: Create a time-limited reservation to purchase a rectangular area of the canvas.

**Nonce Encoding**:
```
Byte 0:      0x00 (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-8:   Rectangle (6 bytes)
Byte 9:      0x00 (ToS separator)
Bytes 10-31: "I accept nonce.art/tos" (22 ASCII bytes)
```

**Transfer Parameters**:
- `to`: TREASURY_ADDRESS
- `value`: See cost calculation below

**Cost Calculation**:
```
area = (x2 - x1) × (y2 - y1)
minDeposit = max(1000000, area × 10000)  // max(1 USDC, area × 0.01 USDC) in USDC units

Phase 1 (first call): value >= minDeposit
Phase 2 (additional calls): value > 0 (any amount)
```

**Validation (Phase 1)**:
1. Validate ToS acceptance at byte 9 (separator 0x00) + bytes 10-31 (exact string match)
2. Decode rectangle from bytes 3-8
3. Check `isValidLot(rect)`:
   - `0 <= x1 < x2 <= 2400`
   - `0 <= y1 < y2 <= 1675`
   - `(x2 - x1) × (y2 - y1) >= 100`
4. Check `value >= minDeposit`
5. Check `!hasOverlap(rect, authorizer, now)` against all lots and active reservations
6. No existing active reservation for this (user, rect) combination

**State Changes (Phase 1 Success)**:
```
resId = "{authorizer}-{x1}-{y1}-{x2}-{y2}"
res = new Reservation(resId)
res.globalIndex = "global"
res.user = authorizer
res.x1, res.y1, res.x2, res.y2 = rect coordinates
res.cost = area × 990000  // 0.99 USDC per pixel (total ownership cost)
res.paid = value
res.expiresAt = now + 900  // `TIME_WINDOW` = 15 minutes
res.status = "ACTIVE"

canvas.totalReservations++
tx.reservation = resId
tx.success = true
```

**Validation (Phase 2 - Adding Payment)**:
1. Validate ToS acceptance (same as Phase 1)
2. Decode rectangle
3. Load existing reservation
4. Check reservation exists and `expiresAt > now`
5. Check `value > 0`

**State Changes (Phase 2)**:
```
res.paid += value

If res.paid >= res.cost:
  // Safety check: verify no overlaps before creating lot
  If hasOverlap(rect, authorizer, now):
    res.status = "FORFEITED"
    res.globalIndex = null
    tx.failureReason = "Overlap detected at completion time"
    return

  // Create lot
  lotId = "{x1}-{y1}"
  lot = new Lot(lotId)
  lot.globalIndex = "global"
  lot.owner = res.user
  lot.x1, lot.y1, lot.x2, lot.y2 = rect coordinates
  lot.pixelData = null  // Initialized to default gray (0xE8E8E8) on first `SET_PIXELS`
  lot.url = null
  lot.urlFragmentsMask = null
  lot.urlFragmentCount = 0
  lot.description = null
  lot.descriptionFragmentsMask = null
  lot.descriptionFragmentCount = 0
  lot.salesCount = 0
  lot.tipCount = 0
  lot.totalTips = 0

  res.status = "COMPLETED"
  res.globalIndex = null  // Removes from activeReservations

  canvas.totalLotsPurchased++
  canvas.totalRevenue += res.paid

  tx.lot = lotId
  tx.success = true
```

**Failure Conditions**:
- ToS not accepted: Funds forfeited
- Invalid rectangle: Funds forfeited
- Insufficient payment (Phase 1): Funds forfeited
- Overlap detected (Phase 1): Funds forfeited
- Overlap detected at completion (Phase 2): Reservation forfeited
- Expired reservation (Phase 2): Funds forfeited (new reservation not created)

---

## `0x01`: `SET_PIXELS`

**Purpose**: Set 1-8 consecutive pixels in an owned lot, starting from a point.

**Nonce Encoding**:
```
Byte 0:        0x01 (opcode)
Bytes 1-2:     LocalNonce (big-endian uint16)
Bytes 3-5:     Point (3 bytes)
Byte 6:        N (number of pixels, 1-8)
Bytes 7-9:     Color 1 (RGB)
Bytes 10-12:   Color 2 (RGB)
...
Bytes 7+3N-1:  Color N (RGB)
Remainder:     Padding (ignored)
```

**Transfer Parameters**:
- `to`: TREASURY_ADDRESS
- `value`: N × 10000 (N × 0.01 USDC in USDC units)

**Validation**:
1. Decode point from bytes 3-5
2. Decode N from byte 6
3. Check `1 <= N <= 8`
4. Check `value >= N × 10000`
5. Find lot containing point:
   - Try O(1) lookup: `Lot.load("{x}-{y}")`
   - If not found, iterate through user's lots to find containing lot
6. Verify `lot.owner == authorizer` OR `isActiveDelegate(lot, authorizer, now)`

**State Changes**:
```
Initialize lot.pixelData if null:
  width = lot.x2 - lot.x1
  height = lot.y2 - lot.y1
  lot.pixelData = new bytes[width × height × 3]
  Fill with `DEFAULT_COLOR` (0xE8)

x = point.x
y = point.y

For i = 0 to N-1:
  If y >= lot.y2:
    break  // Out of bounds, stop

  localX = x - lot.x1
  localY = y - lot.y1
  pixelIndex = localY × (lot.x2 - lot.x1) + localX
  byteOffset = pixelIndex × 3

  lot.pixelData[byteOffset] = colors[i].r
  lot.pixelData[byteOffset+1] = colors[i].g
  lot.pixelData[byteOffset+2] = colors[i].b

  x++
  If x >= lot.x2:
    x = lot.x1  // Wrap to start of row
    y++         // Next row

tx.lot = lot.id
tx.success = true
```

**Failure Conditions**:
- Invalid N (not 1-8): Transaction fails
- Insufficient payment: Transaction fails
- Point not in any owned lot: Transaction fails
- Not owner or active delegate: Transaction fails

**Pixel Wrapping**:
- Pixels wrap at lot boundaries (x wraps, then y increments)
- If y exceeds lot bounds, remaining pixels are discarded
- User still pays for all N pixels

---

## `0x02`: `SET_PIXELS_RL`

**Purpose**: Set multiple pixels using run-length encoding (up to 6 runs).

**Nonce Encoding**:
```
Byte 0:      0x02 (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-5:   Point (3 bytes)
Bytes 6-8:   Color 1 (RGB)
Byte 9:      Count 1 (uint8)
Bytes 10-12: Color 2 (RGB)
Byte 13:     Count 2 (uint8)
...
Bytes 26-28: Color 6 (RGB)
Byte 29:     Count 6 (uint8)
Bytes 30-31: Padding
```

**Transfer Parameters**:
- `to`: TREASURY_ADDRESS
- `value`: totalPixels × 10000
  - `totalPixels = sum(Count1..Count6)`

**Validation**:
1. Decode point from bytes 3-5
2. Decode 6 runs: each has Color (3 bytes) + Count (1 byte)
3. Calculate `totalPixels = sum of all counts`
4. Check `value >= totalPixels × 10000`
5. Find lot containing point (same as `SET_PIXELS`)
6. Verify `lot.owner == authorizer` OR `isActiveDelegate(lot, authorizer, now)`

**State Changes**:
```
Initialize lot.pixelData if null (same as `SET_PIXELS`)

x = point.x
y = point.y

For each run (i = 0 to 5):
  color = runs[i].color
  count = runs[i].count

  For j = 0 to count-1:
    If y >= lot.y2:
      break  // Out of bounds

    Set pixel at (x, y) to color (same logic as `SET_PIXELS`)

    x++
    If x >= lot.x2:
      x = lot.x1
      y++

tx.lot = lot.id
tx.success = true
```

**Failure Conditions**:
- Insufficient payment: Transaction fails
- Point not in owned lot: Transaction fails
- Not owner or active delegate: Transaction fails

---

## `0x03`: `TRANSFER`

**Purpose**: Transfer lot ownership to another address.

**Nonce Encoding**:
```
Byte 0:      0x03 (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-8:   Rectangle (6 bytes) - must match lot exactly
Bytes 9-28:  New owner address (20 bytes)
Bytes 29-31: Padding
```

**Transfer Parameters**:
- `to`: TREASURY_ADDRESS
- `value`: 100000 (0.10 USDC in USDC units = `TRANSFER_COST`)

**Validation**:
1. Decode rectangle from bytes 3-8
2. Decode new owner address from bytes 9-28
3. Check `value >= 100000`
4. Load lot: `lotId = "{x1}-{y1}"`
5. Verify lot exists
6. Verify lot bounds match rectangle exactly:
   - `lot.x1 == rect.x1 && lot.y1 == rect.y1 && lot.x2 == rect.x2 && lot.y2 == rect.y2`
7. Verify `lot.owner == authorizer` (only owner can transfer, not delegates)
8. Verify lot is not listed (`ACTIVE` or `RESERVED` status):
   - Load `listing = Listing.load(lotId)`
   - Check `!listing || (listing.status != "ACTIVE" && listing.status != "RESERVED")`

**State Changes**:
```
lot.owner = newOwner

tx.lot = lot.id
tx.success = true
```

**Content Preservation**:
- `lot.pixelData`: Unchanged
- `lot.url`, `lot.urlFragmentsMask`: Unchanged
- `lot.description`, `lot.descriptionFragmentsMask`: Unchanged
- Delegates: Unchanged (delegates persist across ownership transfer)

**Failure Conditions**:
- Insufficient payment: Transaction fails
- Lot not found: Transaction fails
- Rectangle mismatch: Transaction fails
- Not lot owner: Transaction fails
- Lot is listed: Transaction fails with "Cannot transfer listed lot - delist first"

---

## `0x04`: `SET_URL_FRAGMENT`

**Purpose**: Set one of up to 4 URL fragments (24 bytes each = 96 bytes total).

**Nonce Encoding**:
```
Byte 0:      0x04 (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-5:   Point (3 bytes)
Byte 6:      LocalId (uint8, 0-3)
Bytes 7-30:  String content (24 bytes)
Byte 31:     Padding
```

**Transfer Parameters**:
- `to`: TREASURY_ADDRESS
- `value`: 10000 (0.01 USDC in USDC units)

**Validation**:
1. Decode point from bytes 3-5
2. Decode localId from byte 6
3. Check `0 <= localId <= 3` (MAX_URL_FRAGMENTS = 4)
4. Check `value >= 10000`
5. Extract string from bytes 7-30 (null-terminated)
6. Find lot containing point
7. Verify `lot.owner == authorizer` OR `isActiveDelegate(lot, authorizer, now)`

**State Changes**:
```
fragmentId = "{lotId}-URL-{localId}"
fragment = StringFragment.load(fragmentId) or new StringFragment(fragmentId)
fragment.lot = lot.id
fragment.type = "URL"
fragment.localId = localId
fragment.content = extractString(bytes 7-30)  // Find null terminator, extract up to null
fragment.transaction = tx.id
fragment.save()

// Update lot's fragment mask
If NOT isBitSet(lot.urlFragmentsMask, localId):
  lot.urlFragmentCount++

lot.urlFragmentsMask = setBit(lot.urlFragmentsMask, localId)

// Rebuild concatenated URL
lot.url = ""
hasGap = false
For i = 0 to 3:
  frag = StringFragment.load("{lotId}-URL-{i}")
  If frag exists:
    If hasGap:
      lot.url += "[...]"
      hasGap = false
    lot.url += frag.content
    If frag.content contains '\0':
      break  // End of string
  Else:
    If lot.url.length > 0:
      hasGap = true  // Mark gap but continue

tx.lot = lot.id
tx.success = true
```

**String Extraction**:
- Scan bytes until null terminator (0x00) or end of 24-byte field
- Convert to ASCII string
- Ignore bytes after null terminator

**Fragment Reassembly**:
- Fragments can arrive out of order
- Missing fragments show as `[...]` if there are fragments after the gap
- Example: Fragments 0,2,3 present → "part0[...]part2part3"

**Failure Conditions**:
- LocalId >= 4: Transaction fails
- Insufficient payment: Transaction fails
- Lot not found: Transaction fails
- Not owner or delegate: Transaction fails

---

## `0x05`: `CLEAR_URL`

**Purpose**: Delete all URL fragments and reset URL to null.

**Nonce Encoding**:
```
Byte 0:      0x05 (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-5:   Point (3 bytes)
Bytes 6-31:  Padding
```

**Transfer Parameters**:
- `to`: TREASURY_ADDRESS
- `value`: 10000 (0.01 USDC in USDC units)

**Validation**:
1. Decode point from bytes 3-5
2. Check `value >= 10000`
3. Find lot containing point
4. Verify `lot.owner == authorizer` OR `isActiveDelegate(lot, authorizer, now)`

**State Changes**:
```
lot.url = null
lot.urlFragmentsMask = null
lot.urlFragmentCount = 0

// Delete all URL fragments
For i = 0 to 3:
  fragmentId = "{lotId}-URL-{i}"
  If StringFragment.load(fragmentId) exists:
    delete StringFragment(fragmentId)

tx.lot = lot.id
tx.success = true
```

**Failure Conditions**:
- Insufficient payment: Transaction fails
- Lot not found: Transaction fails
- Not owner or delegate: Transaction fails

---

## `0x06`: `DELEGATE`

**Purpose**: Grant or revoke delegation rights for painting/metadata operations.

**Nonce Encoding**:
```
Byte 0:      0x06 (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-5:   Point (3 bytes)
Byte 6:      Flag (0 = revoke, non-zero = grant)
Bytes 7-26:  Delegate address (20 bytes)
Bytes 27-31: Padding
```

**Transfer Parameters**:
- `to`: TREASURY_ADDRESS
- `value`: 1000000 (1 USDC in USDC units = `DELEGATE_COST`)

**Validation**:
1. Decode point from bytes 3-5
2. Decode flag from byte 6
3. Decode delegate address from bytes 7-26
4. Check `value >= 1000000`
5. Find lot containing point
6. Verify `lot.owner == authorizer` (only owner can delegate, not sub-delegates)

**State Changes (Flag != 0, Grant/Update)**:
```
delegateId = "{lotId}-{delegateAddress}"
delegate = Delegate.load(delegateId) or new Delegate(delegateId)
delegate.lot = lot.id
delegate.delegate = delegateAddress

// Extract validAfter and validBefore from transaction input
// Decode transferWithAuthorization parameters (EIP-3009):
// input[0:4] = function selector (0xcf092995)
// input[100:132] = validAfter (uint256, bytes 96-127 in calldata)
// input[132:164] = validBefore (uint256, bytes 128-159 in calldata)

delegate.validAfter = decode_uint256(txInput[100:132])
delegate.validBefore = decode_uint256(txInput[132:164])
delegate.save()

tx.delegate = delegateId
tx.lot = lot.id
tx.success = true
```

**State Changes (Flag == 0, Revoke)**:
```
delegateId = "{lotId}-{delegateAddress}"
If Delegate.load(delegateId) exists:
  delete Delegate(delegateId)

tx.delegate = delegateId
tx.lot = lot.id
tx.success = true
```

**Delegate Permissions**:
- ✅ Can: `SET_PIXELS`, `SET_PIXELS_RL`
- ✅ Can: `SET_URL_FRAGMENT`, `CLEAR_URL`
- ✅ Can: `SET_DESCRIPTION_FRAGMENT`, `CLEAR_DESCRIPTION`
- ❌ Cannot: `TRANSFER` (owner only)
- ❌ Cannot: `DELEGATE` (no sub-delegation)
- ❌ Cannot: `LIST`, `DELIST` (owner only)

**Permission Check Algorithm**:
```
function isOwnerOrActiveDelegate(lot, user, currentTime):
  If lot.owner == user:
    return true

  For each delegate in lot.delegates:
    If delegate.delegate != user:
      continue

    If currentTime >= delegate.validAfter AND currentTime < delegate.validBefore:
      return true

    // Lazy cleanup
    If currentTime >= delegate.validBefore:
      delete Delegate(delegate.id)

  return false
```

**Failure Conditions**:
- Insufficient payment: Transaction fails
- Lot not found: Transaction fails
- Not lot owner: Transaction fails

---

## `0x07`: `SET_DESCRIPTION_FRAGMENT`

**Purpose**: Set one of up to 10 description fragments (24 bytes each = 240 bytes total).

**Implementation**: Identical to `SET_URL_FRAGMENT` except:
- `type = "DESCRIPTION"`
- `localId` range: 0-9 (MAX_DESCRIPTION_FRAGMENTS = 10)
- Fragment ID format: `"{lotId}-DESCRIPTION-{localId}"`

**Nonce Encoding**: Same as `0x04` but opcode = `0x07`

**Validation**: Same as `0x04` but check `0 <= localId <= 9`

**State Changes**: Same as `0x04` but updates `lot.description`, `lot.descriptionFragmentsMask`, `lot.descriptionFragmentCount`

---

## `0x08`: `CLEAR_DESCRIPTION`

**Purpose**: Delete all description fragments and reset description to null.

**Implementation**: Identical to `CLEAR_URL` except operates on description fields.

**Nonce Encoding**: Same as `0x05` but opcode = `0x08`

**State Changes**: Same as `0x05` but clears `lot.description`, `lot.descriptionFragmentsMask`, `lot.descriptionFragmentCount`

---

## `0x09`: `LIST`

**Purpose**: List a lot for sale on the marketplace.

**Nonce Encoding**:
```
Byte 0:      0x09 (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-5:   Point (3 bytes)
Bytes 6-13:  PixelPrice (8 bytes, big-endian uint64)
Bytes 14-31: Padding
```

**Transfer Parameters**:
- `to`: TREASURY_ADDRESS
- `value`: 1000000 (1 USDC in USDC units = `LISTING_FEE`)

**Validation**:
1. Check `value >= 1000000`
2. Decode point from bytes 3-5
3. Decode price per pixel from bytes 6-13 (big-endian uint64)
4. Check `pricePerPixel >= 10000` (minimum 0.01 USDC per pixel)
5. Find lot containing point
6. Verify `lot.owner == authorizer` (only owner can list)
7. If listing exists with status `RESERVED`, check reservation expired:
   - Load `listing.activeBuyReservation`
   - If not expired, fail with "Cannot modify listing while reserved"

**State Changes**:
```
listing = Listing.load(lot.id) or new Listing(lot.id)
listing.lot = lot.id
listing.seller = authorizer
listing.pricePerPixel = pricePerPixel
listing.status = "ACTIVE"
listing.listedAt = now
listing.activeBuyReservation = null
listing.globalIndex = "global"  // Links to GlobalIndex.activeListings

canvas.totalListingFees += value

tx.listing = listing.id
tx.lot = lot.id
tx.success = true
```

**Updating Existing Listing**:
- If listing already exists with status `ACTIVE`, `SOLD`, or `DELISTED`, it's updated with new price
- User pays listing fee again (no refund for previous listing)

**Failure Conditions**:
- Insufficient payment: Funds forfeited
- Price below minimum: Funds forfeited
- Lot not found: Funds forfeited
- Not lot owner: Funds forfeited
- Active reservation exists: Funds forfeited

---

## `0x0A`: `DELIST`

**Purpose**: Remove listing from marketplace.

**Nonce Encoding**:
```
Byte 0:      0x0A (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-5:   Point (3 bytes)
Bytes 6-31:  Padding
```

**Transfer Parameters**:
- `to`: TREASURY_ADDRESS
- `value`: 10000 (0.01 USDC in USDC units = `DELIST_FEE`)

**Validation**:
1. Check `value >= 10000`
2. Decode point from bytes 3-5
3. Find lot containing point
4. Verify `lot.owner == authorizer`
5. Load `listing = Listing.load(lot.id)`
6. Check listing exists
7. Check `listing.status == "ACTIVE" || listing.status == "RESERVED"`
8. If `RESERVED`, check reservation expired:
   - Load `reservation = BuyReservation.load(listing.activeBuyReservation)`
   - If `reservation.expiresAt > now`, fail
   - If expired, mark `reservation.status = "EXPIRED"` (lazy cleanup)

**State Changes**:
```
listing.status = "DELISTED"
listing.globalIndex = null  // Removes from GlobalIndex.activeListings
listing.activeBuyReservation = null

canvas.totalListingFees += value

tx.listing = listing.id
tx.lot = lot.id
tx.success = true
```

**Failure Conditions**:
- Insufficient payment: Funds forfeited
- Lot not found: Funds forfeited
- Not lot owner: Funds forfeited
- Listing not found: Funds forfeited
- Status is `SOLD` or `DELISTED`: Funds forfeited
- Active (non-expired) reservation exists: Funds forfeited

---

## `0x0B`: `BUY`

**Purpose**: Purchase a listed lot (two-phase: reserve then complete).

**Nonce Encoding**:
```
Byte 0:      0x0B (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-8:   Rectangle (6 bytes) - must match listing exactly
Byte 9:      0x00 (ToS separator)
Bytes 10-31: "I accept nonce.art/tos" (22 ASCII bytes)
```

**Transfer Parameters**:
- Phase 1 (reserve): `to` = TREASURY_ADDRESS
- Phase 2 (complete): `to` = Listing.seller

**Routing Logic**:
```
If to == TREASURY_ADDRESS:
  handleBuyReserve()
Else:
  handleBuyComplete()
```

### `BUY` Phase 1: Reserve (to = Treasury)

**Transfer Parameters**:
- `value`: 1% of total price, minimum 0.01 USDC

**Cost Calculation**:
```
area = (rect.x2 - rect.x1) × (rect.y2 - rect.y1)
totalPrice = listing.pricePerPixel × area
reserveFee = max(totalPrice × 1 / 100, 10000)  // 1%, min 0.01 USDC

Required: value >= reserveFee
```

**Validation**:
1. Validate ToS acceptance at bytes 9-31
2. Decode rectangle from bytes 3-8
3. Check `lotId = "{x1}-{y1}"` exists
4. Verify rectangle matches lot bounds exactly
5. Load `listing = Listing.load(lotId)`
6. Check listing exists
7. Check `listing.status == "ACTIVE"` (handle expired reservations lazily)
8. Check `lot.owner == listing.seller` (defensive check)
9. Check `authorizer != listing.seller` (cannot buy own listing)
10. Check `value >= reserveFee`

**State Changes**:
```
area = (lot.x2 - lot.x1) × (lot.y2 - lot.y1)
totalPrice = listing.pricePerPixel × area
remainingPayment = totalPrice × 99 / 100  // 99% to seller

reservation = new BuyReservation(txHash + "-" + logIndex)
reservation.listing = listing.id
reservation.buyer = authorizer
reservation.reservationFee = value
reservation.remainingPayment = max(remainingPayment, 10000)  // Min 0.01 USDC
reservation.createdAt = now
reservation.expiresAt = now + 300  // `BUY_RESERVATION_WINDOW` = 5 minutes
reservation.status = "ACTIVE"

listing.status = "RESERVED"
listing.activeBuyReservation = reservation.id

canvas.totalMarketplaceRevenue += value

tx.listing = listing.id
tx.buyReservation = reservation.id
tx.lot = lot.id
tx.success = true
```

**Failure Conditions**:
- ToS not accepted: Funds forfeited
- Lot not found: Funds forfeited
- Rectangle mismatch: Funds forfeited
- Listing not found: Funds forfeited
- Listing not ACTIVE: Funds forfeited
- Active reservation exists: Funds forfeited
- Seller no longer owns lot: Funds forfeited, listing delisted
- Buyer is seller: Funds forfeited
- Insufficient payment: Funds forfeited

### `BUY` Phase 2: Complete (to = Seller)

**Transfer Parameters**:
- `value`: Any amount towards remaining payment

**Validation**:
1. Validate ToS acceptance at bytes 9-31
2. Decode rectangle from bytes 3-8
3. Check `lotId = "{x1}-{y1}"` exists
4. Verify rectangle matches lot bounds exactly
5. Load `listing = Listing.load(lotId)`
6. Check `listing.status == "RESERVED"`
7. Check `listing.activeBuyReservation` exists
8. Load `reservation = BuyReservation.load(listing.activeBuyReservation)`
9. Check `reservation.status == "ACTIVE"`
10. Check `reservation.expiresAt > now`
11. Check `to == listing.seller` (payment goes to seller)
12. Check `authorizer == reservation.buyer` (must be reservation holder)
13. Check `lot.owner == listing.seller` (defensive check)

**State Changes (Partial Payment)**:
```
reservation.remainingPayment -= value

canvas.totalSalesVolume += value

tx.listing = listing.id
tx.buyReservation = reservation.id
tx.lot = lot.id
tx.success = true
```

**State Changes (Full Payment: remainingPayment <= 0)**:
```
// Clear all delegations
For each delegate in lot.delegates:
  delete Delegate(delegate.id)

// Transfer ownership
lot.owner = authorizer
lot.salesCount++

reservation.status = "COMPLETED"

listing.status = "SOLD"
listing.globalIndex = null  // Removes from activeListings
listing.activeBuyReservation = null

canvas.totalSalesVolume += value

tx.listing = listing.id
tx.buyReservation = reservation.id
tx.lot = lot.id
tx.success = true
```

**Content Preservation**:
- `lot.pixelData`, `lot.url`, `lot.description`: Unchanged
- Delegates: Cleared on ownership transfer

**Failure Conditions**:
- ToS not accepted: Transaction fails (no forfeit, goes to seller)
- Lot not found: Transaction fails
- Rectangle mismatch: Transaction fails
- Listing not `RESERVED`: Transaction fails
- No active reservation: Transaction fails
- Reservation expired: Transaction fails
- Payment to wrong address: Transaction fails
- Wrong buyer: Transaction fails
- Seller no longer owns lot: Transaction fails, reservation expired, listing delisted

---

## `0x0C`: `TIP`

**Purpose**: Send tip to lot owner.

**Nonce Encoding**:
```
Byte 0:      0x0C (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-5:   Point (3 bytes) - lot origin (x1, y1)
Byte 6:      0x00 (ToS separator)
Bytes 7-28:  "I accept nonce.art/tos" (22 ASCII bytes)
Bytes 29-31: Padding
```

**Transfer Parameters**:
- `to`: Lot owner address
- `value`: Any amount >= 10000 (0.01 USDC minimum)

**Validation**:
1. Validate ToS acceptance at bytes 6-28 (note: offset 6 for `TIP`, not 9)
2. Decode point from bytes 3-5
3. Check `value >= 10000`
4. Load `lot = Lot.load("{x}-{y}")` (O(1) lookup, point must be lot origin)
5. Check lot exists
6. Check `to == lot.owner` (payment goes to owner)
7. Check `authorizer != lot.owner` (cannot tip own lot)

**State Changes**:
```
tip = new Tip(tx.id)  // Uses same {txHash}-{logIndex} format as Transaction
tip.lot = lot.id
tip.tipper = authorizer
tip.amount = value
tip.transaction = tx.id

lot.tipCount++
lot.totalTips += value

canvas.totalTips += value

tx.tip = tip.id
tx.lot = lot.id
tx.success = true
```

**Failure Conditions**:
- ToS not accepted: Transaction fails
- Lot not found: Transaction fails
- Payment to wrong address: Transaction fails
- Self-tipping: Transaction fails

**Notes**:
- Point must be lot origin (top-left corner) for O(1) lookup
- No arbitrary point lookup like other operations
- Tips are publicly visible on-chain

---

## `0x0D`: `FUND`

**Purpose**: Fund an ephemeral account for a lot you own.

**Nonce Encoding**:
```
Byte 0:      0x0D (opcode)
Bytes 1-2:   LocalNonce (big-endian uint16)
Bytes 3-5:   Point (3 bytes) - lot origin (x1, y1)
Byte 6:      0x00 (ToS separator)
Bytes 7-28:  "I accept nonce.art/tos" (22 ASCII bytes)
Bytes 29-31: Padding
```

**Transfer Parameters**:
- `to`: Ephemeral account address (recipient of funds)
- `value`: Any amount >= 10000 (0.01 USDC minimum)

**Validation**:
1. Validate ToS acceptance at bytes 6-28 (note: offset 6 for `FUND`, not 9)
2. Decode point from bytes 3-5
3. Check `value >= 10000`
4. Load `lot = Lot.load("{x}-{y}")` (O(1) lookup, point must be lot origin)
5. Check lot exists
6. Check `authorizer == lot.owner` (only lot owner can fund)
7. Check `authorizer != to` (cannot self-fund)

**State Changes**:
```
fund = new Fund(tx.id)  // Uses same {txHash}-{logIndex} format as Transaction
fund.lot = lot.id
fund.funder = authorizer
fund.recipient = to
fund.amount = value
fund.transaction = tx.id

lot.fundCount++
lot.totalFunding += value

canvas.totalFunding += value

tx.fund = fund.id
tx.lot = lot.id
tx.success = true
```

**Failure Conditions**:
- ToS not accepted: Transaction fails
- Lot not found: Transaction fails
- Funder is not lot owner: Transaction fails
- Self-funding: Transaction fails

**Notes**:
- Point must be lot origin (top-left corner) for O(1) lookup
- Funder must be the lot owner (used for funding ephemeral accounts to paint)
- Self-funding is disallowed (it's a no-op)
- Funds go directly to recipient (no treasury fee)

---

**Next**: [Operations UI →](11-technical/operations-ui.md)
