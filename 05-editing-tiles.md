# Editing Your Tiles

## Opening the Edit Wizard

**Requirement**: Must own the tile

**How to open**:
1. Click your tile on canvas
2. Pinned tooltip appears
3. Click "Edit Tile" button

**Edit Wizard opens** (full-screen modal with 3 phases).

**Note**: Tile cannot be listed for sale while editing. Delist first if needed.

## Wizard Overview

**Three phases**:
1. **Content**: Paint pixels, set URL, set description
2. **Settings**: Configure delegation
3. **Execute**: Review, sign, and submit operations

**Session management**:
- Progress auto-saved in browser (IndexedDB)
- Survives page refresh
- Can download session file as backup
- Can load session file to resume later

**Navigation**:
- Stepper at top shows current phase
- "Next" / "Back" buttons to navigate
- Can jump to any phase (if session loaded)
- "Start Fresh" to reset wizard completely

## Phase 1: Content

### Painting Pixels

**Upload image**:
1. Click "Upload Image" or drag-and-drop
2. Supported formats: PNG, JPG, WebP, GIF, etc.
3. Image automatically resized to fit your tile dimensions

**Position image**:
- Drag preview to position within tile bounds
- Default: Centered
- Preview updates in real-time

**Clear image**: Click "Clear Image" to remove and start over

**Preview**:
- Shows exact pixels that will be painted
- Pixelated rendering (matches canvas style)
- Size indicator shows tile dimensions

**Cost**: 0.01 USDC per pixel painted
- Example: 100-pixel tile = 1 USDC to paint all pixels
- Only pay for pixels that are different from current state
- Re-painting same pixels still costs 0.01 USDC each

### Setting URL

**Purpose**: Make your tile clickable (links to website, portfolio, etc.)

**Maximum length**: 96 bytes (approximately 96 ASCII characters)

**How it works**:
- URL stored in up to 4 fragments (24 bytes each)
- Backend concatenates fragments into full URL
- Displayed in tile tooltip

**Input**:
1. Enter URL in text field
2. Byte counter shows used/remaining
3. Invalid if exceeds 96 bytes

**Cost**: 0.01 USDC per fragment
- URL splits into 1-4 fragments automatically
- Examples:
  - "https://example.com" (20 chars) = 1 fragment = 0.01 USDC
  - "https://example.com/page" (25 chars) = 2 fragments = 0.02 USDC
  - 90-char URL = 4 fragments = 0.04 USDC

**Validation**: Must be valid URL format (checked by browser).

**Clear URL**: Click "Clear URL" button to remove (costs 0.01 USDC).

### Setting Description

**Purpose**: Add text description of your tile (displayed in tooltip)

**Maximum length**: 240 bytes (approximately 240 ASCII characters)

**How it works**:
- Description stored in up to 10 fragments (24 bytes each)
- Backend concatenates fragments into full description
- Displayed in tile tooltip

**Input**:
1. Enter description in text area
2. Byte counter shows used/remaining
3. Invalid if exceeds 240 bytes

**Cost**: 0.01 USDC per fragment
- Description splits into 1-10 fragments automatically
- Examples:
  - 20-char description = 1 fragment = 0.01 USDC
  - 100-char description = 5 fragments = 0.05 USDC
  - 240-char description = 10 fragments = 0.10 USDC

**Clear description**: Click "Clear Description" button to remove (costs 0.01 USDC).

### Byte Counting

**Why bytes, not characters?**
- On-chain storage is measured in bytes
- ASCII characters = 1 byte each
- Unicode characters (emoji, accents) = 2-4 bytes each

**Example**:
- "Hello World!" = 12 bytes (ASCII)
- "Hello 👋 World!" = 16 bytes (emoji is 4 bytes)

**Counter format**: "Used / Max bytes" (updates as you type)

**Warning**: Input turns red if exceeds maximum.

### Validation

**Phase 1 errors**:
- URL exceeds 96 bytes
- Description exceeds 240 bytes
- Self-delegation (if delegation enabled, see Phase 2)

**Cannot proceed to Phase 2** if validation errors exist.

## Phase 2: Settings

Phase 2 lets you choose how operations are signed.

### Signing Options

**Option 1: Sign each operation with your wallet** (default)
- Your wallet prompts for each operation
- Every action is explicitly approved
- No extra cost
- Best for: Small edits (few operations)

**Option 2: Use a temporary account** (delegation)
- Fund a temporary account, then all operations sign automatically
- No repeated wallet prompts
- Cost: +1-2 USDC overhead (delegation + optional revocation)
- Best for: Many operations (large paintings)

### Using Delegation

**Purpose**: Avoid signing dozens of transactions yourself by delegating to a temporary account.

**Important**: All signing happens on your machine. NonceArt never has access to the private key. Save your private key to recover unused funds if the session is interrupted.

**How it works**:
1. Select "Use a temporary account"
2. Generate or paste a private key
3. Wizard delegates your tile to this temporary account
4. Temporary account signs operations automatically
5. Your wallet only signs: delegate, fund, and optionally revoke

**Cost breakdown**:
- Delegate permission: 1 USDC
- Fund temporary account: (cost of your operations)
- Revoke permission (optional): 1 USDC

**Permissions granted to delegate**: See [Delegate Permissions](06-tile-management.md#delegate-permissions) for the full list. In short: delegates can paint and edit metadata, but cannot transfer or re-delegate.

**Time limit**: Delegation expires automatically after the validity period (shown in UI). Skip revocation to save 1 USDC.

**Self-delegation error**: Cannot delegate to yourself.

**Network settings**: Configure facilitator, RPC, and other network options via the gear icon in the bottom-right controls. See [Settings & Configuration](09-settings.md).

## Phase 3: Execute

### Operations Review

**Operations list** shows all operations to be executed:

**For painting**:
- `SET_PIXELS` or `SET_PIXELS_RL`: Number of pixels and colors
- Cost: 0.01 USDC × number of pixels

**For URL**:
- `SET_URL_FRAGMENT` (1-4 operations): Fragment content
- Cost: 0.01 USDC × number of fragments
- Or `CLEAR_URL` (if removing URL): 0.01 USDC

**For description**:
- `SET_DESCRIPTION_FRAGMENT` (1-10 operations): Fragment content
- Cost: 0.01 USDC × number of fragments
- Or `CLEAR_DESCRIPTION` (if removing): 0.01 USDC

**For delegation**:
- `DELEGATE`: Temporary account address and expiry
- Cost: 1 USDC

**Total cost**: Sum of all operation costs (displayed at top).

### Execution Options

**Three modes**:

1. **Sign Only**:
   - Signs all operations
   - Does NOT submit
   - Operations cached in session
   - Use when preparing operations for later execution

2. **Execute** (if already signed):
   - Submits cached operations
   - Available only if operations pre-signed
   - Use to complete a saved session

3. **Resume** (if interrupted):
   - Continues from where execution stopped
   - Available if some operations completed but not all
   - Use after network error or browser close

### Execution Process

**Progress indicators**:
- Each operation shows status: Pending → Signing → Submitting → Completed / Failed
- Progress bar shows overall completion percentage
- Elapsed time displayed

**Concurrent execution**:
- Operations execute in parallel (up to "Concurrent Operations" limit)
- Faster than sequential execution
- Safe: Each operation is independent

**Status messages**:
- "Signing operation X of Y..." - Wallet signature pending
- "Submitting operation X of Y..." - Sending to facilitator
- "Operation completed" - Blockchain confirmed
- "Operation failed" - Error occurred (see error message)

**Completion**:
- All operations succeeded: "All operations completed!" message
- Some failed: Shows count of failed operations
- Can retry failed operations

### Session Management

**Auto-save**:
- Session saved automatically after signing
- Saved to browser IndexedDB (local storage)
- Includes signed operations and state

**Download session**:
- Click "Download Session" button
- Saves JSON file to your device
- Use as backup before complex operations
- File contains signed operations (sensitive data)

**Load session**:
- Click "Load Session" button
- Select previously downloaded JSON file
- Wizard restores state (jumps to Phase 3)
- Can resume execution from saved state

**Session persistence**:
- Survives page refresh
- Survives browser close/reopen
- Cleared when clicking "Start Fresh"
- Cleared when browser data is cleared

**Use cases**:
- Long operations: Sign, save session, execute later
- Backup: Download before execution in case of failure
- Sharing: Sign operations, share session file (anybody with the file will be able to submit your operations!)

### Error Handling

**Quick fixes**:
- **"Authorization already used"**: Click "Retry with New Nonce" button
- **Network errors**: Check Settings (gear icon), increase "Max Retries"
- **Insufficient USDC**: Add funds, then resume execution

**Recovery**: Sessions persist across errors. You can retry failed operations or resume interrupted execution. If unrecoverable, click "Start Fresh".

For detailed error solutions, see [Troubleshooting](10-troubleshooting.md).

### Best Practices

1. **Review carefully**: Check all operations and total cost in Phase 3 before executing
2. **Download session**: Save session file before executing complex changes
3. **Execute promptly**: Don't leave signed sessions unused (signatures may expire)
4. **Monitor progress**: Watch for failed operations and retry if needed
5. **Avoid simultaneous edits**: Don't open multiple edit wizards for same tile

## Operation Costs Summary

| Operation | Unit Cost | Notes |
|-----------|-----------|-------|
| Paint pixel | 0.01 USDC | Per pixel painted |
| Set URL fragment | 0.01 USDC | Max 4 fragments (0.04 total) |
| Set description fragment | 0.01 USDC | Max 10 fragments (0.10 total) |
| Clear URL | 0.01 USDC | Single operation |
| Clear description | 0.01 USDC | Single operation |
| Delegate | 1 USDC | Prevents spam |

**Why these costs?**
- Prevents spam (every operation has cost)
- Covers protocol fees (x402 facilitator)
- Maintains economic incentives for tile ownership

**Total ownership cost**:
- $0.99/pixel to own tile (1% reserve + 98% complete)
- $0.01/pixel to paint
- Optional: URL/description ($0.01-$0.14 total)
- **Total**: $1 USDC per pixel for fully painted tile (plus optional metadata)

---

**Next**: Learn how to transfer tiles and manage delegation → [Tile Management](06-tile-management.md)
