# BRINE — Bitcoin UTXO Ocean

**Version 1.0**

A single-file Bitcoin wallet viewer that maps your UTXOs onto a living ocean, showing at a glance whether your funds can actually move — and at what fee rate they crystallise into Salt.

→ **[Open BRINE](https://blankworker1.github.io/brine-wallet/brine.html)**

---

## The problem BRINE solves

Most wallets show a balance. BRINE shows something more useful: **UTXO mobility**.

Every satoshi you own exists as a UTXO — an unspent transaction output. To spend it, you pay a mining fee. If that fee exceeds the UTXO's value, the sat is economically trapped. BRINE calls this state **Salt**.

A DCA wallet that receives weekly will accumulate many small UTXOs over time. As Bitcoin appreciates and fees rise, UTXOs created during earlier, cheaper periods silently become immobile. BRINE makes this visible before it becomes irreversible.

---

## The model

BRINE is built on the **Bitcoin Aquatic Model** — a framework that treats the 21 million BTC supply as an ocean of Brine, where **Satoshi Density** determines whether a UTXO is liquid or crystallised.

### Satoshi Density

```
SD(u) = V(u) ÷ C_tx(f, s)
```

- `V(u)` — UTXO value in sats
- `C_tx(f, s)` — cost to spend it: fee rate `f` (sat/vB) × transaction size `s` (vB)
- A UTXO is **mobile** when `SD > 1`
- A UTXO becomes **Salt** when `SD ≤ 1`

For a standard P2WPKH transaction (141 vB), at 50 sat/vB, the minimum mobile UTXO is 7,050 sats.

### The tier hierarchy

| Tier | Range |
|---|---|
| Humpback | > 5,000 BTC |
| Whale | 1,000 – 5,000 BTC |
| Shark | 500 – 1,000 BTC |
| Dolphin | 100 – 500 BTC |
| Fish | 50 – 100 BTC |
| Octopus | 10 – 50 BTC |
| Crab | 1 – 10 BTC |
| Shrimp | 0.01 – 1 BTC |
| Plankton | 0.0001 – 0.01 BTC |
| Brine | < 0.0001 BTC |

### The Sieve band

The Sieve is the variable threshold below which UTXOs crystallise into Salt. It is not a fixed number — it moves with network fee pressure.

BRINE shows a **variable band** derived from five years of observed fee data (2020–2026):

- **P50 floor** (12 sat/vB, ~1,692 sats minimum) — normal network conditions
- **P99 ceiling** (500 sat/vB, ~70,500 sats minimum) — major congestion events

A UTXO inside the band is conditionally mobile: liquid today, crystallised during a fee spike. A UTXO above the P99 ceiling is safe under all historically recorded conditions.

---

## How to use

### QR scan (HTTPS required)

1. In your Bitcoin wallet, export your **extended public key** (xpub/ypub/zpub) as a QR code
2. Open BRINE on a device with a camera
3. Point the camera at the QR code

Blue Wallet: Wallet → Settings → Show Wallet xpub

### Manual paste

1. Copy your xpub, ypub, or zpub from your wallet
2. Paste it into the text area on the BRINE scan screen
3. Tap **Scan wallet**

---

## What BRINE shows

### The ocean

Each dot is one of your UTXOs, placed at its satoshi density position in the ocean:

- 🟢 **Green** — above the P99 ceiling, liquid under all recorded fee conditions
- 🟡 **Amber** — inside the sieve band, conditionally mobile
- 🔴 **Red** — Salt, below the current fee threshold, cannot move economically

The sieve line moves in real time as fees change (polling every 30 seconds).

### Stats bar

Current fee rate · total UTXO count · liquid / sieve / salt breakdown · total balance · average UTXO size

### Wallet health

- Total balance
- Average UTXO vs current threshold (with progress bar)
- Verdict: all liquid / sieve exposure / salt present
- Consolidation advice when needed — with a note that merging UTXOs links them on-chain

### Threshold cards

- **Min safe UTXO** — the smallest UTXO that stays mobile at current pressure (DCA floor)
- **Consolidation floor** — minimum total balance for consolidation to cost less than your chosen tolerance
- **Wallet floor** — total balance needed for every individual UTXO to independently clear the sieve

---

## Privacy

**Your xpub never leaves your device.** All cryptographic derivation happens in the browser using the Web Crypto API. No data is sent to any server controlled by this project.

BRINE makes two types of outbound requests:

| Request | Destination | Purpose |
|---|---|---|
| Address/UTXO lookup | `blockstream.info/api` | Check which addresses have been used and retrieve unspent outputs |
| Fee rate | `mempool.space/api/v1/fees/recommended` | Live fee rate for sieve position |

Both Blockstream and mempool.space will see your IP address and the Bitcoin addresses being queried. If this matters to you, self-host BRINE behind a VPN or point it at your own node (v1.1 feature).

### Consolidation and privacy

When BRINE recommends consolidating UTXOs, be aware that merging multiple UTXOs in a single transaction links them on-chain — a blockchain analyst can infer they belong to the same wallet. This is a standard Bitcoin privacy trade-off. If input linkage is a concern, use a tool with coin control or CoinJoin support instead.

---

## Wallet compatibility

| Key format | Derivation | Address type | Status |
|---|---|---|---|
| `zpub` | BIP84 `m/84'/0'/0'` | Native SegWit (`bc1q…`) | ✅ Full support |
| `ypub` | BIP49 `m/49'/0'/0'` | P2SH-SegWit (`3…`) | ✅ Full support |
| `xpub` | BIP44 `m/44'/0'/0'` | Legacy (`1…`) | ✅ Full support |
| `tpub` / Taproot | BIP86 `m/86'/0'/0'` | P2TR (`bc1p…`) | ⏳ v1.1 |

**Blue Wallet** exports `zpub` by default for SegWit wallets. This is the recommended format.

**Coldcard**, **Sparrow**, **Electrum**, and **Ledger Live** can all export xpub/zpub from the wallet settings.

---

## Technical notes

### Cryptography

All cryptography is implemented in pure browser JavaScript with no external libraries:

- **secp256k1** — point multiplication using square-and-multiply modular exponentiation (`modpow`). Direct BigInt `**` exponentiation is avoided — it overflows V8's BigInt size limit for 256-bit exponents.
- **RIPEMD-160** — custom implementation. The f3 round function is `(x|(~y>>>0))^z` — the `>>>0` forces unsigned 32-bit treatment of bitwise NOT, and parenthesisation prevents JavaScript's operator precedence from silently computing `x|(~y^z)` instead.
- **SHA-256** — delegated to the browser's native `SubtleCrypto.digest()`.
- **HMAC-SHA-512** — delegated to `SubtleCrypto.sign()`.
- **BIP32 child key derivation** — unhardened only, as required for xpub-based scanning.
- **Base58Check** — includes 4-byte SHA256d checksum verification on xpub input.
- **Bech32** — full encoder for P2WPKH addresses.

### QR scanning

BRINE uses the browser-native `BarcodeDetector` API with a `getUserMedia` rear-camera stream. No external QR library is loaded. Supported on Chrome 83+, Android Chrome, and Edge. Falls back to paste on Firefox.

### Gap limit

BRINE scans both the external chain (`m/0`, receive addresses) and the internal chain (`m/1`, change addresses) with a gap limit of 20 consecutive empty addresses per chain. This matches the BIP44 standard and covers the vast majority of real-world wallet states.

---

## Self-hosting

BRINE is a single HTML file. To host it yourself:

```bash
git clone https://github.com/blankworker1/brine-wallet
# serve brine.html from any HTTPS server
# camera QR scanning requires HTTPS — http://localhost also works for development
```

QR scanning requires a [secure context](https://developer.mozilla.org/en-US/docs/Web/Security/Secure_Contexts) (HTTPS or localhost). Paste entry works from any origin.

---

## Roadmap

**v1.1**
- Taproot (P2TR) address support
- Configurable API endpoint (point at your own node)
- Consolidation window advisor (best time to merge based on historical fee patterns)

**v2.0**
- Live mempool fee chart with sieve band overlay
- UTXO history — track how your ocean changes over time
- Change address privacy mode

---

## Background

BRINE emerged from a conversation about why the Gini coefficient fails to describe Bitcoin wealth distribution. The core problem: Gini measures addresses, not mobility. A wallet with 1 BTC in a single UTXO and a wallet with 1 BTC split across 1,000 dust outputs look identical to Gini. They are not identical. One can move freely. The other is mostly Salt.

The Aquatic Model replaces the static wealth binary with a fluid, pressure-based framework. Satoshi Density is not about how much you have — it's about whether what you have can move.

---

## Licence

MIT. One file. No dependencies. Audit it yourself.
