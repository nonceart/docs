# Data Encoding Reference

Complete reference for encoding NonceArt operations into 32-byte nonces.

## Nonce Structure

Every NonceArt operation fits within 32 bytes:

```
| Opcode | LocalNonce | Arguments |
| 1 byte |  2 bytes   |  29 bytes |
```

- **Opcode (1 byte)**: Operation type (`0x00`-`0x0D`)
- **LocalNonce (2 bytes, big-endian)**: User-specific counter for nonce uniqueness (0-65535)
- **Arguments (29 bytes)**: Operation-specific data

## Primitive Type Encoding

### Rectangle (6 bytes)

Encodes four 12-bit coordinates as 48 bits:

```
| x1 (12 bits) | y1 (12 bits) | x2 (12 bits) | y2 (12 bits) |
= 48 bits = 6 bytes (big-endian)

Decoding algorithm:
value = bytes[0] << 40 | bytes[1] << 32 | ... | bytes[5]
y2 = value & 0xFFF
x2 = (value >> 12) & 0xFFF
y1 = (value >> 24) & 0xFFF
x1 = (value >> 36) & 0xFFF
```

**Example**: Rectangle (50, 50, 60, 60)
```
Binary:  000000110010 | 000000110010 | 000000111100 | 000000111100
         └─── x1 ───┘   └─── y1 ───┘   └─── x2 ───┘   └─── y2 ───┘
              50             50             60             60

Hex: 0x00320c81e03c (leading 0x00 pads to 6 bytes)
```

**Constraints**:
- 0 ≤ x1 < x2 ≤ 2400 (`MAX_WIDTH`)
- 0 ≤ y1 < y2 ≤ 1675 (`MAX_HEIGHT`)
- (x2 - x1) × (y2 - y1) ≥ 100 (`MIN_LOT_AREA`)

### Point (3 bytes)

Encodes two 12-bit coordinates as 24 bits:

```
| x (12 bits) | y (12 bits) |
= 24 bits = 3 bytes (big-endian)

Decoding algorithm:
value = bytes[0] << 16 | bytes[1] << 8 | bytes[2]
y = value & 0xFFF
x = (value >> 12) & 0xFFF
```

### Color (3 bytes)

Standard RGB:

```
| R (1 byte) | G (1 byte) | B (1 byte) |

No encoding needed - direct byte array [r, g, b]
```

### PixelPrice (8 bytes)

Unsigned 64-bit integer in USDC units (10^6 = 1 USDC):

```
Big-endian uint64
Minimum: 10000 (0.01 USDC)
Maximum: 2^64 - 1 (impractical)
```

### Address (20 bytes)

Ethereum address:

```
20-byte array (no padding)
```

### String (24 bytes)

ASCII string:

```
Up to 24 ASCII characters
Null-terminated (0x00) if shorter
Remainder padded with arbitrary bytes (ignored after null)
```

### ToS Acceptance (23 bytes)

Required for `RESERVE`, `BUY`, `TIP`, `FUND` operations:

```
| Separator (1) | ToS String (22) |
      0x00        "I accept nonce.art/tos"

Exact string match required (case-sensitive)
Proves on-chain acceptance of terms
```

## Complete Operation Reference

| Opcode | Name | Nonce Layout | Recipient | Cost (USDC) |
|--------|------|--------------|-----------|-------------|
| `0x00` | `RESERVE` | Opcode (1) + LocalNonce (2) + Rectangle (6) + ToS (23) | Treasury | max(1, area×0.01) |
| `0x01` | `SET_PIXELS` | Opcode (1) + LocalNonce (2) + Point (3) + N (1) + Colors (N×3) + Padding | Treasury | N×0.01 |
| `0x02` | `SET_PIXELS_RL` | Opcode (1) + LocalNonce (2) + Point (3) + [Color (3) + Count (1)]×6 | Treasury | pixels×0.01 |
| `0x03` | `TRANSFER` | Opcode (1) + LocalNonce (2) + Rectangle (6) + Address (20) + Padding (3) | Treasury | 0.10 |
| `0x04` | `SET_URL_FRAGMENT` | Opcode (1) + LocalNonce (2) + Point (3) + LocalId (1) + String (24) + Padding (1) | Treasury | 0.01 |
| `0x05` | `CLEAR_URL` | Opcode (1) + LocalNonce (2) + Point (3) + Padding (26) | Treasury | 0.01 |
| `0x06` | `DELEGATE` | Opcode (1) + LocalNonce (2) + Point (3) + Flag (1) + Address (20) + Padding (5) | Treasury | 1 |
| `0x07` | `SET_DESCRIPTION_FRAGMENT` | Opcode (1) + LocalNonce (2) + Point (3) + LocalId (1) + String (24) + Padding (1) | Treasury | 0.01 |
| `0x08` | `CLEAR_DESCRIPTION` | Opcode (1) + LocalNonce (2) + Point (3) + Padding (26) | Treasury | 0.01 |
| `0x09` | `LIST` | Opcode (1) + LocalNonce (2) + Point (3) + PixelPrice (8) + Padding (18) | Treasury | 1 |
| `0x0A` | `DELIST` | Opcode (1) + LocalNonce (2) + Point (3) + Padding (26) | Treasury | 0.01 |
| `0x0B` | `BUY` | Opcode (1) + LocalNonce (2) + Rectangle (6) + ToS (23) | Treasury or Seller | 1%/99% split |
| `0x0C` | `TIP` | Opcode (1) + LocalNonce (2) + Point (3) + ToS (23) + Padding (3) | Lot Owner | any ≥ 0.01 |
| `0x0D` | `FUND` | Opcode (1) + LocalNonce (2) + Point (3) + ToS (23) + Padding (3) | Ephemeral Account | any ≥ 0.01 |

## Detailed Encoding Examples

### Rectangle Encoding Details

**Encoding Algorithm (SDK)**:
```typescript
// Pack coordinates into 48-bit value
const value =
  (BigInt(rect.x1) << 36n) |  // x1 at bit position 36-47
  (BigInt(rect.y1) << 24n) |  // y1 at bit position 24-35
  (BigInt(rect.x2) << 12n) |  // x2 at bit position 12-23
  BigInt(rect.y2);            // y2 at bit position 0-11

// Convert to 6 bytes (big-endian)
const bytes = new Uint8Array(6);
for (let i = 0; i < 6; i++) {
  bytes[5 - i] = Number((value >> BigInt(i * 8)) & 0xffn);
}
```

**Decoding Algorithm (Subgraph)**:
```typescript
// Read 6 bytes as 48-bit big-endian value
let value = BigInt.zero()
for (let i = 0; i < 6; i++) {
  value = value.leftShift(8).plus(BigInt.fromI32(data[i]))
}

// Extract 12-bit coordinates (RIGHT TO LEFT)
const mask12 = BigInt.fromI32(0xFFF)  // 4095
const y2 = value.bitAnd(mask12).toI32(); value = value.rightShift(12)
const x2 = value.bitAnd(mask12).toI32(); value = value.rightShift(12)
const y1 = value.bitAnd(mask12).toI32(); value = value.rightShift(12)
const x1 = value.bitAnd(mask12).toI32()
```

**Full Example**:
```
Input: (x1=50, y1=50, x2=60, y2=60)
Encoded: 0x00320c81e03c (6 bytes)

Bytes:  0x00   0x32   0x0c   0x81   0xe0   0x3c
        └──────┴──────┴──────┴──────┴──────┴───── Big-endian order

Binary (48 bits, left to right):
000000000110010000001100100000011110000000111100
│          │          │          │          │
└──── x1 ──┴──── y1 ──┴─── x2 ───┴─── y2 ───┘
     50         50         60         60

Bit positions: 47.........36|35.........24|23.........12|11..........0
Binary:        000000110010|000000110010|000000111100|000000111100
Fields:            x1     |     y1     |     x2     |     y2
Values:            50     |     50     |     60     |     60

Byte-by-byte breakdown:
  Byte 0 (0x00): bits 47-40 → [padding for x1]
  Byte 1 (0x32): bits 39-32 → [x1, y1 boundary]
  Byte 2 (0x0c): bits 31-24 → [y1]
  Byte 3 (0x81): bits 23-16 → [y1, x2 boundary]
  Byte 4 (0xe0): bits 15-8  → [x2, y2 boundary]
  Byte 5 (0x3c): bits 7-0   → [y2]

Decoding (right-to-left extraction):
  Step 1: value & 0xFFF → bits 0-11  → y2 = 60 ✅
  Step 2: value >> 12   → shift right 12 bits
  Step 3: value & 0xFFF → bits 0-11  → x2 = 60 ✅
  Step 4: value >> 12   → shift right 12 bits
  Step 5: value & 0xFFF → bits 0-11  → y1 = 50 ✅
  Step 6: value >> 12   → shift right 12 bits
  Step 7: value & 0xFFF → bits 0-11  → x1 = 50 ✅
```

**Key Points**:
- ✅ Bytes are written in **big-endian order** (MSB first)
- ✅ Fields are extracted **right to left** (y2, x2, y1, x1)
- ✅ SDK encoding and subgraph decoding are perfectly aligned
- ✅ Maximum coordinate value: 4095 (12 bits)

### String Fragment Encoding

**URL Fragments** (max 4 fragments = 96 bytes total):
- Fragment IDs: 0-3
- Each fragment: 24 bytes
- Null-terminated strings
- Missing fragments show as `[...]` in concatenated result

**Description Fragments** (max 10 fragments = 240 bytes total):
- Fragment IDs: 0-9
- Each fragment: 24 bytes
- Same null-termination and gap handling as URL

**Example**:
```
Fragment 0: "http://exa"
Fragment 1: "mple.com\0"
Fragment 2: (missing)
Fragment 3: "/page"

Concatenated URL: "http://example.com[...]/page"
```

**Note**: If a fragment contains `\0`, string assembly stops early.

---

**Next**: [Operation Specifications →](11-technical/operations.md)
