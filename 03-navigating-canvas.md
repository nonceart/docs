# Navigating the Canvas

## Canvas Controls

### Zoom

**Mouse wheel**: Zoom towards cursor position
**Keyboard**:
- `+` or `=`: Zoom in
- `-`: Zoom out
- `0`: Fit to screen
- `1`: 100% zoom (1:1)

**Buttons** (bottom-right):
- `+` / `-`: Zoom in/out
- `Fit`: Reset to fit entire canvas in viewport
- `1:1`: Zoom to 100% (native resolution)

**Zoom range**: Fit mode (minimum) to 10× (1000%)

**Note**: Cannot zoom out beyond fit mode (prevents excessive empty space).

### Pan

**Mouse**: Click and drag anywhere on canvas

**Keyboard**: Arrow keys (when not in selection mode)

**Bounds**: Cannot pan beyond canvas edges. When at fit zoom, panning is limited to prevent showing opposite edges simultaneously.

## Viewing Tiles

### Hover Preview

Hover over any tile to see tooltip with:
- Canvas preview (painted pixels)
- Size and position
- Owner address

**Tooltip behavior**:
- Position locks when you first hover over a tile
- Tooltip only moves when you hover over a different tile
- Disappears when mouse leaves canvas area

### Pinned Details

Click any tile to pin the tooltip:

**Pinned tooltip shows** (blue border indicates pinned):
- Full tile information
- URL (clickable if set)
- Description (if set)
- Owner address (truncated, click to copy)
- Actions (if you own the tile or it's for sale)

**To close**: Click anywhere outside the tooltip or press `Escape`

**Links**: URLs are only clickable when tooltip is pinned (not during hover).

### Tile Information

**For painted tiles**:
- Width × Height (area in pixels)
- Position: Top-left corner coordinates (x, y)
- Owner: Ethereum address
- URL: Clickable link (if set)
- Description: Text (if set)
- Listing: Price and status (if for sale)

**For unpainted tiles** (owned but not painted):
- Same info as painted tiles
- Background remains gray until painted

**For reservations**:
- Blinking colored rectangle
- Time remaining (15 minutes total)
- Can click if you own it to complete payment

**Coordinate system**:
- Top-left of canvas is (0, 0)
- x increases rightward, y increases downward
- Coordinates specify **inclusive start, exclusive end**
  - Example: Tile at (100, 200) with size 50×50 occupies pixels (100, 200) through (149, 249)
  - Pixels at x=150 or y=250 are NOT included

## View Modes

### Clean View

**Activate**: Press `Space` or click eye icon (bottom-right)

**Hidden elements**:
- Header (logo, wallet button)
- Sidebar menu
- Footer
- Zoom controls (except clean view toggle)
- Tile tooltips

**Visible elements**:
- Canvas art
- Reservation overlays (blinking)
- Unpainted tile overlays (pastel colors)
- Clean view toggle button (semi-transparent)

**Use cases**:
- Taking screenshots
- Viewing art without UI distractions
- Presentations

**To exit**: Press `Space` again or click eye icon

### Show Only Mine

**Activate**: Press `M` or click user icon (bottom-right)

**Requirements**: Wallet must be connected

**Effect**:
- Your tiles: Full opacity (normal)
- Others' tiles: Faded with semi-transparent gray overlay
- Reservations: All visible (yours and others')

**Use cases**:
- Quickly find your tiles
- Check your portfolio
- Identify which tiles you own

**To exit**: Press `M` again or click user icon

**Note**: Overlay matches canvas background color for seamless fade effect.

## Activity Feed

**Location**: Sidebar menu → Activity section

**Shows**: Recent transactions on the canvas (real-time)

**Transaction types** (with color coding):
- **NEW** (violet): New tile reserved
- **PAID** (violet): Reservation completed
- **TRANSFER** (violet): Ownership transferred
- **SELL** (violet): Tile listed for sale
- **BID** (violet): Buy reservation placed
- **SOLD** (violet): Sale completed
- **TIP** (yellow): Owner tipped
- **FUND** (violet): Funded ephemeral account
- **CANCEL** (gray): Listing cancelled
- **DELEGATE** (violet): Delegation granted
- **REVOKE** (gray): Delegation revoked

**Note**: Painting and metadata operations (SET_PIXELS, SET_URL, etc.) are not shown in the activity feed to reduce noise.

**Each entry shows**:
- Operation type (label with icon)
- Location on canvas (x, y coordinates)
- Timestamp (relative: "2m ago", "1h ago")
- Transaction hash (click to view on block explorer)

**Real-time updates**: Activity feed updates via WebSocket connection (indicator shows connection status).

**New transactions flash briefly** (highlight animation) when they appear.

## Canvas Statistics

**Location**: Sidebar menu → Info section

**Displays**:
- **Painted pixels**: Total pixels that have been painted
- **Total pixels**: 4,020,000 (canvas capacity)
- **Percentage sold**: Progress bar showing canvas utilization
- **Refresh pulse**: Animated opacity indicator that pulses when data updates

## Keyboard Shortcuts Reference

**Zoom**:
- `+` / `=`: Zoom in
- `-`: Zoom out
- `0`: Fit to screen
- `1`: 100% zoom

**View**:
- `Space`: Toggle clean view mode
- `M`: Toggle "show only mine" filter

**Selection** (when in purchase mode):
- `Enter`: Enter/exit selection mode
- `Escape`: Cancel selection

**Tooltips**:
- `Escape`: Close pinned tooltip

**Navigation**:
- Arrow keys: Pan canvas (when not in selection mode)

**Field Navigation** (in dialogs):
- `Tab`: Next field
- `Shift+Tab`: Previous field

**Note**: Shortcuts disabled when typing in input fields.

---

**Next**: Learn how to reserve your first tile → [Reserving Tiles](04-reserving-tiles.md)
