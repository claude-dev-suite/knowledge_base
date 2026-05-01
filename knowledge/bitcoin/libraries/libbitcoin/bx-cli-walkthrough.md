# libbitcoin bx CLI Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/libbitcoin`.
> Canonical source: https://github.com/libbitcoin/libbitcoin-explorer
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/libbitcoin/SKILL.md

## Concept

`libbitcoin` is a comprehensive C++ Bitcoin library suite organised as
several independent components: `libbitcoin-system` (primitives),
`libbitcoin-server`, `libbitcoin-blockchain`, `libbitcoin-network`, and
the developer-facing CLI `libbitcoin-explorer` shipped as the `bx`
binary. While the daemon side has limited adoption today, `bx` remains
a useful Swiss-army-knife CLI for ad-hoc Bitcoin operations: seed
generation, BIP39 mnemonics, BIP32 derivation, address conversion,
script encoding/decoding, Electrum-protocol queries.

Compared to `bitcoin-cli`, `bx` doesn't require a running node for the
offline operations -- it's a self-contained toolbox.

## Install

```bash
# Debian / Ubuntu
sudo apt install libbitcoin-explorer
# macOS
brew install libbitcoin-explorer
# From source: see https://github.com/libbitcoin/libbitcoin-explorer/wiki/Build-Linux
```

Verify:

```bash
bx --version
bx help
```

## Worked example: end-to-end HD key generation

```bash
# 1. Generate 16 bytes of entropy
ENTROPY=$(bx seed --bit_length 128)
echo "entropy: $ENTROPY"

# 2. Encode as a BIP39 mnemonic
MNEMONIC=$(bx mnemonic-new $ENTROPY)
echo "mnemonic: $MNEMONIC"

# 3. Mnemonic -> seed -> master xprv
SEED=$(bx mnemonic-to-seed "$MNEMONIC")
XPRV=$(bx hd-new $SEED)
echo "master xprv: $XPRV"

# 4. Derive m/84'/0'/0'/0/0 (native segwit)
ACCOUNT=$(bx hd-private --hard --index 84 $XPRV)
ACCOUNT=$(bx hd-private --hard --index 0  $ACCOUNT)
ACCOUNT=$(bx hd-private --hard --index 0  $ACCOUNT)
ACCOUNT=$(bx hd-private --index 0 $ACCOUNT)
LEAF=$(bx hd-private --index 0 $ACCOUNT)

# 5. Get x-pub leaf, then address
XPUB_LEAF=$(bx hd-to-public $LEAF)
PUB=$(bx hd-to-ec $XPUB_LEAF)
ADDR=$(bx ec-to-address --version 0 $PUB)     # P2PKH legacy address
echo "address: $ADDR"
```

For BIP84 native segwit addresses, encode the pubkey hash via
`bx bitcoin160` + `bx address-encode --prefix bc1`:

```bash
PKHASH=$(bx ec-to-public $PUB | bx bitcoin160)
SEGWIT=$(echo "0014$PKHASH" | bx script-encode | bx witness-encode bc)
```

(Helper composability: pipe outputs through `bx` subcommands.)

## Useful subcommands

```bash
# WIF roundtrip
bx ec-new                     # random secret
bx ec-to-wif <hex_secret>     # WIF
bx wif-to-ec <wif>            # back to hex secret

# Address conversion
bx address-decode 1Boatxxx... # show version + payload
bx ec-to-address --version 5 <hex_pub>   # P2SH address

# Tx inspection
bx tx-decode <raw_hex>        # human readable JSON
bx tx-sign --hash <sighash> <wif>

# Script
bx script-decode <hex>        # disassemble
bx script-encode "OP_DUP OP_HASH160 [20] OP_EQUALVERIFY OP_CHECKSIG"

# Electrum query (needs --server config)
bx fetch-balance 1Boatxxx...
bx fetch-history 1Boatxxx...
```

## Configuration

`bx` reads `~/.config/libbitcoin/bx.cfg`:

```ini
[network]
identifier = mainnet
[server]
url = tcp://mainnet.libbitcoin.net:9091
public_key = ...
```

Set up server URL + public key for Electrum-style queries; offline
subcommands (`seed`, `hd-*`, `ec-*`, `address-*`) need no config.

## Common pitfalls

- Many subcommands accept either positional arg or stdin; mixing them
  in pipelines can drop input. Use explicit `--` and one or the other.
- `bx hd-private --hard --index N` adds 0x80000000 to the index;
  `--index N` without `--hard` is the unhardened variant. Forgetting
  `--hard` on BIP44/49/84 account paths produces an unrelated key.
- The default address version is mainnet legacy (1...). For P2SH pass
  `--version 5`; for testnet legacy `--version 111`. Native segwit /
  Taproot addresses require `script-encode` + bech32 wrappers.
- Bundled servers may be offline; for production fetch operations run
  your own libbitcoin-server or use Electrum / Esplora / mempool.space
  via different tooling.
- libbitcoin gets soft-fork updates more slowly than Core; treat `bx`
  results for new features (Taproot script paths) with caution and
  cross-check against `bitcoin-cli decoderawtransaction`.

## References

- Repo: https://github.com/libbitcoin/libbitcoin-explorer
- Wiki: https://github.com/libbitcoin/libbitcoin-explorer/wiki
- Subcommand index: https://github.com/libbitcoin/libbitcoin-explorer/wiki/Configuration-Settings
- Companion: [secp256k1-c/SKILL.md](../secp256k1-c/SKILL.md), [libwally/SKILL.md](../libwally/SKILL.md)
