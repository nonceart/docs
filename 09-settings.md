# Settings & Configuration

## Global Settings

**Purpose**: Configure NonceArt backend connections and execution parameters.

**Access**: Click the gear icon in the bottom-right controls (below zoom buttons)

### Configuration Options

**Facilitator URL**:
- x402 facilitator endpoint
- Default: NonceArt x402 facilitators
- Format: https://facilitator.example.com
- Used for: Submitting signed authorizations
- In case of issues, you can use these facilitators:
   - [Coinbase CDP](https://docs.cdp.coinbase.com/x402/network-support#cdp-hosted-recommended) - requires creating a free account
   - List of x402 facilitators on [x402scan](https://www.x402scan.com/facilitators)

**RPC URL**:
- Blockchain read/write endpoint
- Default: NonceArt dedicated RPC endpoints
- Format: https://rpc.example.com
- Used for: Reconciliation of the blockchain state with the session to resume
- In case of issues, you can use these RPCs:
   - [Infura](https://www.infura.io) - requires creating a free account
   - [Alchemy](https://www.alchemy.com) - requires creating a free account
   - List of public Base RPCs on [Chainlist]](https://chainlist.org/chain/8453)

**WebSocket URL**:
- Real-time blockchain updates
- Default: Base network WebSocket
- Format: wss://ws.example.com
- Used for: Live transaction monitoring

**Max Retries**:
- Number of retry attempts if operation fails
- Default: 3
- Range: 0-10
- Used for: Network error recovery

**Concurrent Operations**:
- Maximum parallel operations in Edit Wizard
- Default: 5
- Range: 1-20
- Used for: Phase 3 execution of operations that set content

### When to Change Settings

**Use defaults when**:
- Starting out (recommended for all new users)
- Using official NonceArt frontend
- Network is stable

**Customize when**:
- Using custom x402 facilitator
- Using custom RPC provider (Alchemy, Infura, self-hosted)
- Experiencing rate limits (lower concurrent ops)
- Network is unstable (increase max retries)
- Network has extra capacity (increase concurrent operations, but you may get rate-limited)
- Testing/development (custom endpoints)

### Reset to Defaults

**How to reset**: Click "Reset to Defaults" button in Settings dialog

**When to reset**:
- Settings misconfigured (operations failing)
- Want to revert customizations
- Troubleshooting connection issues

### Settings Persistence

**Storage**: Settings saved in browser localStorage

**Scope**: Per-domain (separate settings for testnet vs mainnet if different domains)

**Persistence**: Survives page refresh and browser restart

**Clearing**: Browser data clear removes settings (resets to defaults)

### Validation

**URL validation**:
- Must be valid HTTP/HTTPS URLs
- Must be reachable (not validated upfront, errors appear during operation execution)

**Numeric validation**:
- Max Retries: Must be 0-10
- Concurrent Operations: Must be 1-20

**Invalid settings**: Submit button disabled until all fields valid.

## Mobile Support

**Current status**: NonceArt is optimized for desktop but mostly functional on mobile.

**Mobile experience**:
- ✅ Touch zoom (pinch)
- ✅ Touch pan (drag)
- ✅ Tap to view tiles
- ✅ Wallet connection (mobile wallet apps)
- ⚠️ Selection mode challenging (small touch targets)
- ⚠️ Edit Wizard works but smaller screen

---

**Next**: Troubleshoot common issues → [Troubleshooting](10-troubleshooting.md)
