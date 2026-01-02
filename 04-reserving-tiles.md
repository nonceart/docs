# Reserving Tiles

## How to Reserve

### Entering Selection Mode

**Method 1**: Click "Reserve" button in the left sidebar menu

**Method 2**: Press `Enter` key

**Effect**: Selection UI appears at bottom-center of screen.

**Note**: Most UI elements are hidden during selection mode. Zoom controls remain visible.

### Selecting Your Area

**Mouse selection**:
1. Click and drag on canvas to draw rectangle
2. Release to complete selection
3. Selection updates input fields automatically

**Keyboard input**:
1. Click into coordinate/size input fields
2. Enter values directly:
   - **Top-left**: x, y coordinates
   - **Size**: width × height in pixels
3. Selection rectangle updates on canvas in real-time

**Both methods work simultaneously** (mouse and keyboard stay in sync).

### Selection Interface

**Input fields** (left to right):
- **Top-left (x, y)**: Starting position on canvas
- **Size (w × h)**: Width and height in pixels
- **Price**: Total cost (read-only, calculated automatically)
- **OK**: Confirm selection (opens payment modal)
- **Cancel**: Exit selection mode

**Keyboard navigation**:
- `Tab`: Move forward through fields (x → y → width → height → OK → Cancel → x...)
- `Shift+Tab`: Move backward
- `Enter` on input field: Focus OK button (if selection valid)
- `Enter` on button: Activate button
- `Escape`: Cancel selection

**Focus indicators**: Blue border on inputs, blue ring on buttons.

### Coordinate System

**Origin**: Top-left corner of canvas is (0, 0)

**Bounds**:
- x: 0 to 2,400
- y: 0 to 1,675

**Inclusive/Exclusive**:
- **Top-left coordinates (x1, y1)**: Inclusive (pixel is part of your tile)
- **Bottom-right coordinates (x2, y2)**: Exclusive (pixel is NOT part of your tile)

**Example**:
- Selection: Top-left (100, 200), Size 50×50
- Your tile occupies: x = 100 to 149, y = 200 to 249 (inclusive)
- Pixels at x=150 or y=250 are NOT yours
- Bottom-right coordinates would be (150, 250) but those pixels are excluded

**Why this matters**: Tiles placed edge-to-edge don't overlap. If your tile ends at x=150 (exclusive), another tile can start at x=150 without conflict.

### Validation

**Minimum size**: 100 pixels total
- Example valid sizes: 10×10, 20×5, 100×1, 50×2

**Messages**:
- Blue hint (area = 0): "Select area with mouse or enter coordinates"
- Red error (invalid): Specific validation error
- No message (valid): Selection ready

**Common errors**:
- Coordinates out of bounds (< 0 or beyond canvas edges)
- Width or height is zero or negative
- Area < 100 pixels
- Overlaps existing tile or active reservation

**OK button** enabled only when selection is valid.

## Pricing

**Pricing Model**:
- **Ownership Cost**: 0.99 USDC per pixel (reserve 0.01 + complete 0.98)
- **Painting Cost**: 0.01 USDC per pixel (paid separately when painting)
- **Total**: 1.00 USDC per pixel (to own and paint)

**Examples (Ownership Cost)**:
- 100 pixels (10×10): 99 USDC
- 500 pixels (25×20): 495 USDC
- 1,000 pixels (50×20): 990 USDC
- 10,000 pixels (100×100): 9,900 USDC

**No discounts**: Linear pricing regardless of tile size.

## Two-Phase Payment

NonceArt uses a reservation system to prevent race conditions (two users claiming the same area simultaneously).

### Phase 1: Reserve (1% upfront)

**Cost**:
- 1% of ownership cost (area × 0.01 USDC)
- **Minimum**: 1 USDC (even if 1% < 1 USDC)

**Examples**:
- 100-pixel tile ($100): 1 USDC reserve (1% of 100 = 1)
- 500-pixel tile ($500): 5 USDC reserve (1% of 500)
- 1,000-pixel tile ($1,000): 10 USDC reserve (1% of 1,000)
- 10,000-pixel tile ($10,000): 100 USDC reserve (1% of 10,000)

**What happens**:
1. Payment settles on blockchain (via x402)
2. Backend creates reservation entity
3. Reservation appears on canvas as blinking rectangle
4. 15-minute timer starts
5. Area is locked (others cannot reserve)

**Terms of Service**: Must accept ToS before reserving. Click checkbox and confirm.

**Reservation overlay colors**:
- Green: 10-15 minutes remaining
- Yellow: 5-10 minutes remaining
- Red: 0-5 minutes remaining

### Phase 2: Complete Payment (98% remaining)

**Cost**:
- Remaining 98% of ownership cost (area × 0.98 USDC)
- **Note**: Does NOT include painting cost (0.01 USDC/px)

**Examples**:
- 100-pixel tile: 98 USDC remaining (1 USDC reserve + 98 USDC = 99 USDC total ownership)
- 500-pixel tile: 490 USDC remaining (5 USDC reserve + 490 USDC = 495 USDC total ownership)
- 1,000-pixel tile: 980 USDC remaining (10 USDC reserve + 980 USDC = 990 USDC total ownership)

**How to complete**:
1. Click your reservation on canvas (blinking rectangle)
2. Tooltip shows remaining payment amount
3. Enter amount to pay (or click "Pay Remaining")
4. Confirm payment

**Partial payments allowed**:
- Can pay in multiple installments
- Minimum: 1 USDC per payment
- Example: 1,000-pixel tile (990 USDC remaining):
  - Pay 500 USDC → 490 USDC remaining
  - Pay 400 USDC → 90 USDC remaining
  - Pay 90 USDC → Ownership complete

**Dust payments** (< 1 USDC remaining):
- Input field disabled
- Must pay exact remaining amount
- No tolerance (exact BigInt equality required)

**Completion**:
- When total paid = ownership cost (reserve + complete = 99% of $1/pixel)
- Reservation converts to owned tile
- Blinking stops, tile becomes solid
- You can now edit and paint

**Visual feedback**:
- During payment: Button disabled, "Processing..." message
- After payment settles: Reservation overlay turns solid blue
- After backend confirms: Reservation disappears, you own the tile

### Payment Status Tracking

**Three states**:

1. **Active reservation** (blinking overlay):
   - Payment incomplete
   - Click to pay more
   - Shows time remaining

2. **Payment settling** (solid blue overlay):
   - Payment transaction submitted
   - Waiting for blockchain confirmation
   - Cannot make additional payments (prevents overpayment)

3. **Completed** (no overlay):
   - Backend confirmed ownership
   - Tile is yours
   - Ready to edit and paint

### Expiration

**Time limit**: 15 minutes from initial reserve payment

**If expired before full payment**:
- Reservation is forfeited
- **No refund** on reserve payment (1%)
- Area becomes available for others to reserve
- You can reserve the same area again if desired

**Preventing expiration**:
- Watch countdown timer on reservation overlay
- Complete payment before timer expires
- Make partial payments to secure your claim

## Managing Pending Reservations

### Viewing Your Reservations

**On canvas**: Blinking rectangles (colored by time remaining)

**Pending Reservations button** (top-right corner, below wallet button):
- Shows count of your active reservations
- Click to view list
- Each reservation shows:
  - Position and size
  - Remaining payment
  - Time left
  - "Complete Payment" button

**Identifying your reservations**:
- Use "Show Only Mine" mode (`M` key)
- Your reservations remain visible, others' are faded

### Completing Payments

**From canvas**:
1. Click your reservation (blinking rectangle)
2. Tooltip opens with payment section
3. Review remaining amount
4. Enter payment amount or click "Pay Remaining"
5. Confirm transaction

**From Pending Reservations panel**:
1. Click "Complete Payment" on any reservation
2. Navigates to reservation on canvas
3. Opens payment tooltip
4. Follow payment flow as above

### Multiple Reservations

**Allowed**: You can have multiple active reservations simultaneously

**Strategy**:
- Reserve multiple areas during exploration
- Complete payments within 15-minute windows
- Prioritize based on time remaining

**Warning**: Each reservation requires 1% upfront. Budget accordingly.

## Troubleshooting

**Common issues**:
- "Authorization already used" error (nonce collision)
- Reservation not appearing on canvas
- Payment completed but tile not owned
- Reservation expired before completion

**Solutions**: See the dedicated [Troubleshooting](10-troubleshooting.md) guide for detailed solutions to these and other common issues.

## Best Practices

1. **Start small**: Reserve 100-500 pixels for your first tile
2. **Check availability**: Use zoom to verify area is empty before selecting
3. **Complete promptly**: Don't wait until final minutes to pay remaining amount
4. **Save transaction hashes**: Keep record of reserve and payment transactions
5. **Budget correctly**:
   - $0.99/pixel to own the tile (1% reserve + 98% complete)
   - Additional $0.01/pixel to paint it
   - Total = $1 USDC per pixel (owned + painted)

---

**Next**: Learn how to paint and customize your tile → [Editing Your Tiles](05-editing-tiles.md)
