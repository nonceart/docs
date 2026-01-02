# Getting Started

## Understanding USDC

**USDC** is a stablecoin: 1 USDC = 1 USD.

NonceArt uses USDC because:
- Stable pricing (not affected by ETH volatility)
- EIP-3009 support (enables gasless transactions and nonce storage)
- Available on Base and other networks

### Getting USDC on Base

**Option 1: Bridge from Ethereum**
- Use [Base Bridge](https://bridge.base.org)
- Transfer USDC from Ethereum mainnet to Base

**Option 2: Exchange**
- Buy directly on Base via Coinbase, Binance, or other exchanges
- Withdraw to your wallet address on Base network

**Option 3: On-ramp**
- Use fiat on-ramps that support Base (Coinbase, Moonpay)

### Gasless Transactions

NonceArt uses **x402 protocol** to sponsor gas fees.

**What this means for you**:
- No ETH required
- Pay only in USDC
- x402 facilitator submits transactions on your behalf
- You sign, facilitator pays gas

All you need is USDC in your wallet.

## Interface Overview

### Canvas View (Main Area)

**Canvas dimensions**: 2,400 pixels wide × 1,675 pixels tall

**Color scheme**:
- Light gray background: Unpainted pixels
- Colored pixels: Painted tiles
- Blinking rectangles: Active reservations (yours or others')

### Sidebar Menu (Left)

**Info Section**:
- Painted pixels / Total pixels
- Percentage sold
- Refresh indicator

**Activity Feed**:
- Real-time transaction log
- Shows recent reservations, paintings, transfers, sales
- WebSocket status indicator (crossed WiFi icon means delayed updates due to problems with WS RPC address)
- Subgraph status indicator (database icon means that subgraph is catching up with indexing blockchain events)

**Action Buttons** (when connected):
- Reserve
- Tiles
- Operations (advanced)

### Controls (Bottom-Right)

**Zoom Buttons**:
- `+` / `-`: Zoom in/out
- `Fit`: Fit canvas to screen
- `1:1`: 100% zoom (1 canvas pixel = 1 screen pixel)

**Other Controls**:
- Gear icon: Settings (x402 facilitator, RPC addresses, etc.)
- User icon: Show only my tiles
- Eye icon: Clean view mode (hide UI)

**Keyboard shortcuts**: See Navigating the Canvas section.

## Wallet Connection

### Supported Wallets

Any Web3 wallet works:
- MetaMask
- Coinbase Wallet
- WalletConnect-compatible wallets
- Rainbow, Frame, etc.

### Connecting

1. Click "Connect Wallet" button (top-right)
2. Select your wallet from the list
3. Approve connection in wallet popup
4. Wallet address appears in header when connected

### Network Requirements

**Supported network**: **Base** (mainnet, recommended)

If connected to wrong network:
- NonceArt will show "Wrong Network" warning
- Click "Switch Network" button
- Or manually switch in your wallet

Your wallet will prompt you to add Base network if not already configured.

### Disconnecting

Click connected address → Disconnect (wallet-specific UI varies)

## First Steps

Once connected:

1. **Explore the canvas**
   - Zoom and pan to see existing tiles
   - Click tiles to see details (owner, URL, price if listed)

2. **Check your USDC balance**
   - Ensure you have enough USDC for the tile you want
   - Minimum: 1 USDC/pixel (smallest reservable tile is 100 pixels)

3. **Reserve your first tile**
   - See [Reserving Tiles](04-reserving-tiles.md) for full guide
   - Start small (100-500 pixels) to learn the system

4. **Paint and customize**
   - See [Editing Your Tiles](05-editing-tiles.md)
   - Upload image, set URL, add description

---

**Next**: Learn how to navigate the canvas → [Navigating the Canvas](03-navigating-canvas.md)
