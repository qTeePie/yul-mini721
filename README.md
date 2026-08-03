# MiniNFT 🐥

**⚠️ DISCLAIMER: This NFT is for educational purposes only. It is not ERC-721 compliant, not production-ready, and not intended for wallet or marketplace support.**

I wanted to understand the EVM at the opcode level, and implementing an NFT felt like a fun way to do exactly that.

MiniNFT is a deliberately unconventional NFT implementation. While it is **not ERC-721 compliant**, it is a genuine non-fungible token that demonstrates many of the low-level concepts hidden behind Solidity. Things like bitpacking metadata into storage, manually decoding calldata, and implementing a function dispatcher make it a fun *play-and-learn* project.

✅ **Mints NFTs**
✅ **Tracks ownership**
✅ **Emits Transfer events**
❌ **Does not implement the ERC-721 specification**
🎯 **Built purely as a playground for learning low-level EVM internals**

---

# What You'll Learn

This project demonstrates:

* EVM storage layout
* Constructor vs runtime bytecode
* Manual function dispatching
* Manual calldata decoding
* ABI encoding
* Bitpacking
* Custom errors
* Memory management
* Event logs
* On-chain SVG generation
* Reading and writing storage directly from Yul

---

# Setup

```bash
cast abi-decode "svg()(string)" $(cast call <CONTRACT_ADDR> "svg()" --rpc-url <RPC_URL>) > output.svg
```

Open `output.svg` in any browser or image viewer.

---

## Available Make Commands

| Command            | Description                                         |
| ------------------ | --------------------------------------------------- |
| `make build`       | Compile the Yul contract into raw bytecode (`.bin`) |
| `make deploy`      | Deploy via Foundry (`DeployMini721.s.sol`)          |
| `make mint`        | Mint a token to `USER_ADDR`                         |
| `make totalSupply` | Read on-chain total supply                          |
| `make fork-anvil`  | Start an Anvil mainnet fork                         |
| `make clean`       | Remove build artifacts                              |

Configuration is loaded automatically from `.env`.

Example:

```text
RPC_URL=http://127.0.0.1:8545
PRIVATE_KEY=0x...
CONTRACT_ADDR=0x...
USER_ADDR=0x...
```

Once configured:

```bash
make deploy
make mint
make totalSupply
```

---

# Design Notes

MiniNFT intentionally takes several shortcuts compared to a production NFT contract. The goal is to expose how the EVM actually works rather than hide those details behind Solidity abstractions.

## Storage Layout

MiniNFT demonstrates two different storage strategies.

| Slot   | Purpose                                |
| ------ | -------------------------------------- |
| `0x00` | `totalSupply`                          |
| `0x09` | Base slot for `balanceOf` mapping      |
| `0x10` | Base slot for sequential owner storage |

### `balanceOf` — Real EVM Mapping

Balances use the exact storage layout Solidity generates for mappings.

```text
balanceOf[address]
→ keccak256(address, balancesBaseSlot)
```

This demonstrates how mappings are actually represented inside EVM storage rather than existing as native hash maps.

---

### `ownerOf` — Sequential Storage

Ownership intentionally does **not** use a mapping.

Instead:

```text
slot = ownersBase + tokenId
```

Since token IDs are sequential, ownership can simply be computed as an offset from a base slot.

Each owner slot is also bitpacked:

```text
[ 24-bit RGB Color | 72-bit Padding | 160-bit Owner Address ]
```

This allows the NFT's color and owner to live inside a single storage slot.

MiniNFT intentionally demonstrates both approaches side by side so their trade-offs become easier to understand.

---

## Bitpacking

Instead of storing each property separately, MiniNFT packs multiple values into a single 256-bit storage slot.

Current owner layout:

```text
[ RGB Color | Owner Address ]
```

This makes the project a good playground for learning masking, shifting, packing, and unpacking operations.

Future ideas (currently commented out) include packing additional metadata into the `totalSupply` slot simply as an educational exercise.

---

## Function Dispatch

MiniNFT does not rely on Solidity's generated dispatcher.

Instead it manually:

1. Reads the first four calldata bytes.
2. Extracts the function selector.
3. Uses a Yul `switch`.
4. Dispatches execution manually.

```text
selector()
        │
        ▼
switch selector
├── mint()
├── transfer()
├── ownerOf()
└── ...
```

This closely mirrors the dispatcher Solidity generates automatically.

---

## ABI Decoding

Instead of relying on Solidity's ABI decoder, MiniNFT manually decodes every argument.

Helper functions include:

* `decode_as_uint()`
* `decode_as_address()`

These perform bounds checking while demonstrating how calldata is interpreted.

---

## On-chain SVG

The artwork is embedded directly inside the deployed runtime bytecode.

Calling `svg(tokenId)`:

1. Loads the packed owner slot.
2. Extracts the RGB value.
3. Converts each byte into hexadecimal.
4. Inserts the color into the SVG template.
5. Returns the finished SVG string.

No JSON metadata or Base64 encoding is involved.

---

## Memory Usage

Many functions intentionally demonstrate common Yul memory patterns, including:

* Using the free memory pointer (`0x40`)
* Reusing scratch memory before `revert`
* Building ABI return values manually
* Constructing revert payloads directly in memory

The source code contains additional comments explaining why particular memory regions are safe to overwrite.

---

## Events

MiniNFT emits standard `Transfer` events for both minting and transfers.

Although the contract is **not ERC-721 compliant**, emitting these events makes activity easy to inspect using tools like Foundry and block explorers.

---

## Naming Conventions

To make different abstraction levels easier to distinguish:

* **camelCase** — External ABI functions (`ownerOf`, `balanceOf`, ...)
* **snake_case** — Internal helpers and low-level operations (`decode_as_uint`, `unpack_rgb`, ...)

---

# Error Handling

MiniNFT demonstrates two common revert styles used throughout EVM development.

## 1. Custom Errors (selector-only)

* ABI returns only a 4-byte selector (for example `InvalidToken()`).
* No offsets, dynamic data, or padding.
* Extremely gas efficient.
* Preferred by most modern Solidity codebases.

Yul implementation:

```yul
mstore(ptr, shl(224, selector))
revert(ptr, 0x04)
```

---

## 2. Classic `Error(string)`

This is the older Solidity `require()` style.

The ABI encodes:

```
selector
↓
offset
↓
length
↓
string bytes
↓
padding
```

Yul implementation:

```solidity
function revert_error_classic(msg, msg_size) {
    let ptr := mload(0x40)

    mstore(ptr, shl(224, 0x08c379a0))
    mstore(add(ptr, 0x20), 0x20)

    let data_ptr := add(ptr, 0x40)

    mstore(data_ptr, msg_size)
    datacopy(add(data_ptr, 0x20), msg, msg_size)

    let padded := and(add(msg_size, 0x1f), not(0x1f))

    revert(ptr, add(padded, 0x60))
}
```

---

> **MiniNFT uses only custom errors**, but both styles are included so readers can understand how revert payloads differ at the ABI level.
