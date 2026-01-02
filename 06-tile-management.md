# Tile Management

## Transferring Ownership

**Purpose**: Permanently give your tile to another address.

**Requirements**:
- Must own the tile
- Tile cannot be listed for sale (delist first)
- Recipient must have valid Ethereum address

**What transfers**:
- ✅ Ownership (full control)
- ✅ Painted pixels (visual content preserved)
- ✅ URL (link preserved)
- ✅ Description (text preserved)
- ❌ Delegations (cleared on transfer)

**How to transfer**:
1. Click your tile on canvas
2. Pinned tooltip appears
3. Click "Transfer" button
4. Transfer dialog opens

**Transfer dialog**:
- **Recipient address**: Enter Ethereum address (0x...)
- **Validation**: Address must be valid format (42 characters, starts with 0x)
- **Terms of Service**: Must accept ToS (checkbox)
- **Preview**: Shows tile info and recipient
- **Cost**: 0.10 USDC flat fee

**Confirmation**:
1. Review recipient address carefully
2. Accept Terms of Service
3. Click "Transfer" button
4. Sign transaction in wallet
5. Wait for confirmation (10-30 seconds)

**After transfer**:
- You no longer own the tile
- Recipient becomes owner
- Recipient can edit, transfer, or list for sale
- **Transfer is irreversible** (cannot undo)

**Cost**: 0.10 USDC
- Flat fee regardless of tile size
- Covers protocol fees

**Use cases**:
- Gift tiles to friends
- Move tiles between your accounts
- Sell tiles off-platform (arrange payment separately)

**Warning**: Verify recipient address carefully. If you send to wrong address or burn address (0x000...000), tile is lost permanently.

## Delegation

**Purpose**: Allow another address to paint on your tile without transferring ownership.

**Differences from Transfer**:

| Feature | Transfer | Delegation |
|---------|----------|------------|
| Ownership | Changes | Preserved |
| Risk | High (full control) | Low (limited permissions) |
| Reversible | No | Yes (can revoke) |
| Time limit | Permanent | Temporary (expires) |
| Cost | 0.10 USDC | 1 USDC |

### Granting Delegation

**Method 1: Edit Wizard** (recommended):
1. Open Edit Wizard for your tile
2. Phase 2 → Enable "Delegate to Temporary Account"
3. Generate temporary account private key
4. Wizard executes delegation operation
5. Share temporary account private key with delegate

**Method 2: Operations Modal** (advanced):
1. Open Operations Modal (sidebar → Operations button)
2. Select `DELEGATE` operation
3. Enter:
   - Top left point of your tile
   - Delegate address (Ethereum address)
   - Grant flag (1 = grant, 0 = revoke)
4. Sign and execute

**Cost**: 1 USDC (prevents spam)

**Time limit**:
- Based on wallet signature validity window
- Default: 30 days
- After expiry, delegation automatically invalid
- No on-chain revocation needed (lazy cleanup)

### Delegate Permissions

**Allowed operations** (delegate can perform):
- ✅ Paint pixels (`SET_PIXELS`, `SET_PIXELS_RL`)
- ✅ Set URL (`SET_URL_FRAGMENT`)
- ✅ Clear URL (`CLEAR_URL`)
- ✅ Set description (`SET_DESCRIPTION_FRAGMENT`)
- ✅ Clear description (`CLEAR_DESCRIPTION`)

**Forbidden operations** (only owner can perform):
- ❌ Transfer ownership
- ❌ Delegate to others (no sub-delegation)
- ❌ List for sale / Delist
- ❌ Accept buy offers

**Validation**: Subgraph checks delegate permissions. Forbidden operations are rejected.

### Revoking Delegation

**Automatic expiry**: Delegation expires when signature validity window ends (typically 1 hour). No action needed.

**Manual revocation** (before expiry):
1. Open Operations Modal
2. Select `DELEGATE` operation
3. Enter:
   - Top left point of your tile
   - Delegate address (same as granted)
   - Grant flag = 0 (revoke)
4. Sign and execute

**Cost**: 1 USDC (same as granting)

**Effect**: Delegate can no longer perform operations. Previously signed operations may still execute if submitted before revocation.

### Multiple Delegates

**Allowed**: Tile can have multiple active delegates simultaneously.

**Use cases**:
- Multiple collaborators
- Multiple painting services
- Backup delegates

**Management**: Each delegate is independent. Revoke individually by address.

### Security Considerations

**Delegate can**:
- Change all visual content (paint over your art)
- Change URL (replace your link)
- Change description (replace your text)
- Cost you USDC (operations they perform deduct from your balance)

**Delegate cannot**:
- Transfer tile away from you
- List tile for sale
- Grant additional delegations

**Best practices**:
1. Only delegate to trusted addresses
2. Use short validity windows (1 hour recommended)
3. Monitor delegate activity in Activity feed
4. Revoke immediately if suspicious activity
5. Keep temporary account keys secure (treat like passwords)

**When to use**:
- Hiring professional painter: Delegate temporarily, revoke after work done
- Collaborative art: Delegate to team members
- Testing: Delegate to test account for experiments

**When NOT to use**:
- Permanent access: Use Transfer instead
- Untrusted parties: Too risky (they can destroy your art)
- Simple edits: Use Edit Wizard yourself (safer)

## Clearing Content

### Clearing URL

**How to clear**:
1. Open Edit Wizard
2. Phase 1 → Click "Clear URL" button
3. Phase 3 → Execute (`CLEAR_URL` operation)

**Cost**: 0.01 USDC

**Effect**:
- URL removed from tile
- Tooltip shows "No URL set"
- All URL fragments deleted from subgraph

### Clearing Description

**How to clear**:
1. Open Edit Wizard
2. Phase 1 → Click "Clear Description" button
3. Phase 3 → Execute (`CLEAR_DESCRIPTION` operation)

**Cost**: 0.01 USDC

**Effect**:
- Description removed from tile
- Tooltip shows "No description set"
- All description fragments deleted from subgraph

### Clearing Pixels

**No direct "clear pixels" operation**. To remove painted content:

**Option 1**: Paint over with solid color
1. Create solid color image (matches canvas background)
2. Upload in Edit Wizard
3. Paint all pixels (costs 0.01 USDC per pixel)

**Option 2**: Paint new image
- Upload new image
- Replaces old pixels
- Costs 0.01 USDC per pixel changed

**Note**: Every pixel change costs 0.01 USDC, even if "clearing" back to gray.

## Tile States

**Owned, unpainted**:
- Canvas shows gray background with pastel color overlay
- You can edit and paint anytime
- Others see as unavailable

**Owned, painted**:
- Canvas shows your pixel art
- Others can view, click URL
- You can edit anytime

**Listed for sale**:
- Tooltip shows listing status and price when clicked
- Cannot edit while listed (must delist first)
- Cannot transfer while listed

**Being transferred**:
- Transaction pending (10-30 seconds)
- Still yours until confirmed
- Cannot edit during transfer

**Delegated**:
- Delegates can paint
- You retain ownership
- Can revoke anytime

## Best Practices

1. **Verify addresses**: Double-check recipient address before transferring
2. **Backup art**: Download screenshots before major changes
3. **Delegate cautiously**: Only to trusted addresses with short time limits
4. **Monitor Activity feed**: Watch for unexpected operations on your tiles
5. **Delist before transfer**: Cannot transfer while listed for sale
6. **Budget for operations**: Keep USDC balance for future edits

---

**Next**: Learn how to buy and sell tiles on the marketplace → [Marketplace](07-marketplace.md)
