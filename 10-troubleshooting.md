# Troubleshooting

Common issues and solutions when using NonceArt.

## Payment & Transaction Errors

### "Authorization already used" Error

**Cause**: Nonce collision (random local nonce was already used)

**Solution**:
1. Click "Retry with New Nonce" button (generates new random nonce automatically)
2. If still failing, click "Advanced" → Specify custom nonce (0-65535)
3. Click "Generate Random" for new suggestion or enter manually
4. Click "Retry with Custom Nonce"

**Probability**: Very low (~0.076% with 100 operations). Retry almost always succeeds.

### Transaction Stuck or Pending

**Symptoms**: Transaction submitted but not confirming

**Solutions**:
1. **Check Activity feed**: Look for your transaction status
2. **Check block explorer**: Click transaction hash to view on explorer
3. **Wait**: Transactions typically confirm very quickly (sub-second)

**If stuck >1 minute**:
- Refresh the page
- Check wallet to ensure transaction actually submitted
- Verify transaction appears in block explorer

### Insufficient Balance Errors

**"Insufficient USDC balance"**:
- Check USDC balance in wallet
- Ensure you have enough for the full operation cost
- Remember: 0.99 USDC/pixel to own + 0.01 USDC/pixel to paint

## Reservation Issues

### Reservation Not Appearing

**Check**:
1. Activity feed (sidebar) for your transaction
2. Transaction hash in wallet history
3. Block explorer (click hash in activity feed)

**If missing**:
- Backend may be delayed
- Check blockchain explorer to verify payment succeeded
- Refresh the page

### Payment Completed But Tile Not Owned

**Visual progression**:
1. Payment transaction settles → Overlay turns solid blue
2. Backend processes → Overlay disappears, tile appears owned (click to verify)

**If stuck**:
- Refresh page
- If still not owned after waiting, contact support with transaction hash

### Reservation Expired

**What happened**: 15-minute timer expired before full payment

**Result**:
- Reservation is forfeited
- **No refund** on reserve payment (1%)
- Area becomes available for others to reserve

**Prevention**:
- Watch countdown timer on reservation overlay
- Complete payment before timer expires
- Make partial payments to secure your claim

## Wallet Connection Issues

### Wallet Not Connecting

**Solutions**:
1. **Refresh page** and try again
2. **Check wallet extension** is installed and unlocked
3. **Switch networks** in wallet to Base
4. **Clear browser cache** and retry
5. **Try different wallet** (MetaMask, Coinbase Wallet, etc.)

### Wrong Network

**Symptom**: UI shows "Wrong network" or operations disabled

**Solution**:
1. Open wallet extension
2. Switch to "Base" network
3. UI should automatically detect and re-enable features

**If Base not listed**:
- Add Base network to wallet (most wallets auto-detect)
- Base Chain ID: 8453
- RPC URL: https://mainnet.base.org

### Wallet Disconnects Frequently

**Causes**:
- Browser extension update
- Wallet locked automatically
- Network switch

**Solutions**:
- Reconnect when prompted
- Keep wallet unlocked while using NonceArt
- Check browser extension permissions

## Canvas Display Issues

### Tiles Not Loading or Appearing Blank

**Solutions**:
1. **Refresh page** (most common fix)
2. **Clear browser cache**
3. **Check console** for errors (F12 → Console tab)
4. **Try different browser** (Chrome, Firefox, Safari, Brave)

### Canvas Zoom or Pan Not Working

**Solutions**:
1. **Disable selection mode** (press Escape if in selection mode)
2. **Refresh page**
3. **Check browser zoom** (reset to 100%)
4. **Try different input device** (different mouse/trackpad)

### "Show Only Mine" Not Working

**Possible causes**:
- Wallet not connected (filter requires wallet connection)
- No tiles owned yet

**Solutions**:
1. Ensure wallet is connected
2. Verify you own tiles (click "Tiles" in sidebar, filter by "Show Only Mine")
3. Press `M` key or click user icon button again

## Edit Wizard Issues

### Image Upload Fails

**Supported formats**: PNG, JPEG, GIF, WebP

**Maximum size**: No hard limit, but very large images may be slow to process

**Solutions**:
1. **Check format**: Ensure file is a valid image
2. **Reduce size**: If image is very large, resize it first
3. **Try different browser**: Some browsers handle large images better
4. **Convert format**: Try saving as PNG if other formats fail

### "No changes detected" When Painting

**Cause**: Image exactly matches existing pixels (nothing to paint)

**Solutions**:
1. Verify you're editing the correct tile
2. Check that your uploaded image differs from current state
3. Try changing at least one pixel in your image

### Operations Fail During Execution (Phase 3)

**Symptoms**: Some operations succeed, others fail

**Solutions**:
1. **Check failed operations**: Use status filter in wizard to view failed operations
2. **Common causes**:
   - Insufficient USDC (ran out mid-batch)
   - Network issues (temporary RPC failure)
   - Invalid operations (e.g., painting outside tile bounds)
3. **Download session**: Save session file, fix issues, then load and retry

### Session Won't Load

**Symptoms**: Uploaded session file doesn't restore operations

**Solutions**:
1. **Verify file format**: Must be `.json` file from NonceArt
2. **Check orchestrator**: Wait for orchestrator to initialize (loading screen)
3. **Try refreshing**: Reload page and try again
4. **Check file integrity**: Ensure file wasn't corrupted (open in text editor)

## Backend Connection Issues

### Activity Feed Not Updating

**Symptoms**: New operations not appearing in feed

**Solutions**:
1. **Check WebSocket connection**: Look for connection indicator
2. **Refresh page**: Re-establishes WebSocket
3. **Check firewall**: Ensure WebSocket (wss://) traffic allowed
4. **Try different network**: Corporate networks may block WebSockets

### Subgraph Data Delayed

**Symptoms**: Tiles not appearing immediately after transaction confirms

**If delayed**:
1. **Refresh page**
2. **Check The Graph status**: Subgraph may be temporarily behind
3. **Verify transaction confirmed**: Check block explorer

### Settings Not Saving

**Symptoms**: Custom RPC/facilitator URLs reset after refresh

**Solutions**:
1. **Check localStorage**: Browser may be blocking storage
2. **Disable private/incognito mode**: localStorage doesn't persist
3. **Check browser extensions**: Ad blockers may interfere
4. **Try different browser**

## Common Mistakes

### Trying to Paint Before Owning

**Error**: "Lot not found or not owned by user"

**Solution**: You must own a tile before painting it. Complete reservation payment first.

### Painting Outside Tile Bounds

**Error**: Operation fails silently or shows "Invalid coordinates"

**Solution**:
- Verify coordinates are within your tile
- Check tile bounds in tooltip
- Use Edit Wizard (validates automatically)

### Listing Already-Listed Tile

**Error**: "Lot already listed"

**Solution**: Delist first, then create new listing

### Transferring Listed Tile

**Error**: "Cannot transfer listed lot"

**Solution**: Delist the tile before transferring ownership

### Delegate Trying to Transfer

**Error**: "Operation not authorized"

**Solution**: Only owner can transfer. Delegates can only paint/edit metadata.

## Getting Help

### In-App Resources

1. **Activity Feed**: Monitor all your operations in real-time
2. **Block Explorer**: Click transaction hash to view on Base explorer
3. **Error Messages**: Read carefully - often contain specific solution
4. **Help widget**: You can report problems using the widget in bottom left corner

### Community Support

- **GitHub**: [github.com/nonceart](https://github.com/nonceart)
- **Documentation**: Review relevant sections (below)

### Reporting Bugs

**Include**:
1. Transaction hash (if applicable)
2. Wallet address
3. Browser and version
4. Steps to reproduce
5. Screenshots or browser console errors

### Related Documentation

- **Reserving issues**: See [Reserving Tiles](04-reserving-tiles.md)
- **Painting issues**: See [Editing Your Tiles](05-editing-tiles.md)
- **Wallet issues**: See [Getting Started](02-getting-started.md)
- **Advanced errors**: See [Operations UI](11-technical/operations-ui.md)

---

**Previous**: [Marketplace](07-marketplace.md) | **Next**: [Technical Specification](11-technical/architecture.md)
