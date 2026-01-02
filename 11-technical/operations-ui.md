# Operations UI (Advanced)

**Audience**: Power users, developers, and those comfortable with low-level protocol operations.

**Warning**: This is an advanced feature that exposes raw protocol operations. It bypasses many safety checks and validations present in the regular UI flows. Invalid operations will still cost USDC but will be rejected by the subgraph.

## Overview

**Purpose**: Direct access to all NonceArt operations for advanced use cases.

**Access**: Sidebar menu → Operations button (wrench icon)

**Key difference from Edit Wizard**: The Operations UI provides raw operation creation without validation, while the Edit Wizard provides a guided, validated workflow.

## Features

### Direct Operation Execution

**Workflow**:
1. Select operation type from dropdown
2. Enter operation arguments manually in custom fields
3. View encoded nonce (32-byte hex representation)
4. Execute operation or copy nonce for external use

### Available Operations

All NonceArt protocol operations are available:

- `RESERVE`: Reserve new tile
- `SET_PIXELS`: Paint 1-8 pixels
- `SET_PIXELS_RL`: Paint run-length encoded pixels
- `SET_URL_FRAGMENT`: Set URL fragment
- `CLEAR_URL`: Remove URL
- `SET_DESCRIPTION_FRAGMENT`: Set description fragment
- `CLEAR_DESCRIPTION`: Remove description
- `TRANSFER`: Transfer ownership
- `DELEGATE`: Grant/revoke delegation
- `LIST`: List for sale
- `DELIST`: Remove listing
- `BUY`: Buy listed tile
- `TIP`: Send tip

### Argument Editors

Each operation type has custom input fields:

**Rectangle editor** (`RESERVE`, `BUY`):
- Top-left coordinates (x1, y1)
- Bottom-right coordinates (x2, y2)

**Point editor** (`SET_PIXELS`, `TRANSFER`, `DELEGATE`, etc.):
- Single coordinate pair (x, y)

**Color picker** (`SET_PIXELS`):
- RGB color selection or hex input
- Multiple colors for multi-pixel operations

**String editor** (`SET_URL_FRAGMENT`, `SET_DESCRIPTION_FRAGMENT`):
- Text input for fragment content
- Fragment ID selector (0-3 for URL, 0-9 for description)

**Address input** (`TRANSFER`, `DELEGATE`, `TIP`):
- Ethereum address (0x...)
- Validates checksum

## Nonce Encoding/Decoding

### View Encoded Nonce

After entering operation arguments, the 32-byte hex nonce is displayed:

```
0x00000000640c81e03c0000000000000000000000000000000000000000000000
```

This shows exactly what will be sent to the facilitator.

### Decode Existing Nonce

**Use case**: Inspect nonces from transactions or other sources

**How**:
1. Paste 32-byte hex nonce into decoder
2. UI parses and displays operation type and arguments
3. Verify operation matches expectations

### Copy Nonce

Click copy button to save encoded nonce to clipboard for:
- External testing
- Batch processing scripts
- Documentation or debugging

## Execution

### Sign & Execute

**Process**:
1. Review operation cost (shown in USDC)
2. Review payment recipient (treasury or tile owner)
3. Click "Execute" button
4. Sign transaction in wallet
5. Wait for confirmation

**Same payment flow as Edit Wizard**:
- USDC approval (if needed)
- x402 facilitator submission
- Transaction monitoring
- Activity feed tracking

### Cost Display

Shows exact operation cost based on:
- Operation type
- Number of pixels (for `SET_PIXELS` operations)
- Area size (for `RESERVE` operations)

**Examples**:
- `RESERVE` (100px): 1.00 USDC (reserve payment)
- `SET_PIXELS` (5 pixels): 0.05 USDC
- `TRANSFER`: 0.10 USDC
- `DELEGATE`: 1.00 USDC
- `LIST`: 1.00 USDC

### Payment Recipients

Operations route payments to different recipients:

**Treasury** (most operations):
- `RESERVE`
- `SET_PIXELS`, `SET_PIXELS_RL`
- `SET_URL_FRAGMENT`, `CLEAR_URL`
- `SET_DESCRIPTION_FRAGMENT`, `CLEAR_DESCRIPTION`
- `DELEGATE`
- `LIST`, `DELIST`, `BUY` (reserve phase)

**Tile owner**:
- `TIP` (100% to owner)
- `BUY` complete phase (99% to seller)

### Error Handling

**Same as Edit Wizard**:
- Nonce collision retry with random nonce
- Network error retry
- Payment failure recovery
- Transaction status monitoring

**Errors unique to Operations UI**:
- Invalid arguments (detected during encoding)
- Out-of-bounds coordinates
- Invalid addresses

## Use Cases

### When to Use Operations UI

**Good use cases**:

1. **Testing new operation types**: Experiment with protocol features
2. **Debugging encoding issues**: Verify nonce structure
3. **Custom workflows**: Operations not supported by main UI
4. **Learning**: Understand nonce structure and encoding
5. **Advanced tile management**: Direct control over all operations
6. **Batch processing**: Pre-generate nonces for bulk operations

**Example workflows**:
- Set multiple URL fragments without Edit Wizard session
- Test delegation with specific time windows
- Create custom transfer workflows
- Generate test nonces for subgraph testing

### When NOT to Use

**Not recommended for**:

1. **Regular painting**: Use Edit Wizard instead
   - Wizard validates ownership
   - Wizard optimizes pixel operations
   - Wizard handles batching automatically

2. **First-time users**: Use guided flows
   - Purchase flow validates overlap
   - Edit Wizard prevents common mistakes
   - Regular UI provides better error messages

3. **Complex multi-operation edits**: Use Edit Wizard's session management
   - Wizard tracks operation state
   - Wizard provides retry/resume
   - Wizard shows progress clearly

## Risks & Limitations

### Skipped Validations

**What Operations UI does NOT check**:

- ❌ Overlap checking (`RESERVE`)
- ❌ Owner verification (`TRANSFER`, `DELEGATE`)
- ❌ Balance checking (upfront)
- ❌ Coordinate bounds (until execution)
- ❌ Confirmation dialogs

**Result**: Invalid operations still cost USDC but get rejected by subgraph.

### Examples of Invalid Operations

**`RESERVE` overlap**:
- Reserving area that overlaps existing tile
- Cost: 1 USDC (minimum reserve)
- Result: Operation fails, payment forfeited

**`SET_PIXELS` on non-owned tile**:
- Painting pixels on someone else's tile
- Cost: 0.01-0.08 USDC (per pixel)
- Result: Operation fails, payment forfeited

**`TRANSFER` non-owned tile**:
- Transferring tile you don't own
- Cost: 0.10 USDC
- Result: Operation fails, payment forfeited

### Best Practices

1. **Verify ownership** before painting/transferring
2. **Check coordinates** are within canvas bounds
3. **Test with small amounts** first
4. **Use Edit Wizard** for complex workflows
5. **Save nonces** before executing (for reference)
6. **Monitor Activity feed** for operation status

## Comparison: Operations UI vs Edit Wizard

| Feature | Operations UI | Edit Wizard |
|---------|---------------|-------------|
| **Validation** | Minimal | Comprehensive |
| **Batch operations** | Manual | Automatic |
| **Session management** | None | Full state tracking |
| **Error recovery** | Basic | Advanced (retry/resume) |
| **Use case** | Testing, debugging | Production painting |
| **User level** | Advanced | All users |
| **Safety** | User responsible | UI prevents mistakes |

## Related Documentation

- **Operation encoding**: [Data Encoding](data-encoding.md)
- **Operation semantics**: [Operations Reference](operations.md)
- **Cost calculation**: [Architecture Overview](architecture.md)
- **Payment flow**: [Security Model](security.md)

---

**Next**: [State Machine →](11-technical/state-machine.md)
