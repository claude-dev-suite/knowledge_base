# Custom Signet Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/testing/signet`.
> Canonical source: https://github.com/bitcoin/bitcoin/blob/master/contrib/signet/README.md
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/testing/signet/SKILL.md

## Concept

Signet (BIP325) replaces proof-of-work with a signature on each block
header by holders of a fixed `signetchallenge` script. Anyone can run
their own signet by choosing a challenge (e.g. a 1-of-1 P2WSH or a
2-of-3 multisig) and operating the signing daemon. Validators on that
network do not need PoW power - they only need to verify the
signature against the agreed challenge.

The result is a deterministic public-style network where you control
block production cadence, supply, and reorgs without burning hashpower.
It is the closest thing to mainnet you can get while keeping the
"reset button". Used for protocol experiments (CTV, APO, Drivechains)
and Lightning testing where regtest is too local and testnet is too
chaotic.

## Walkthrough / mechanics

A custom signet has two components:

1. The challenge: an output script. Block headers carry a witness that
   must satisfy that script. The simplest is a single OP_CHECKSIG over
   a P2WPKH key. More robust is a 2-of-3 P2WSH multisig.
2. The signing infrastructure: `contrib/signet/miner` is the canonical
   signer. It builds a candidate block, computes the signet header
   commitment, signs it, and submits it back via `submitblock`.

Every other node only needs the `signetchallenge` and an `addnode` to
the signer to validate and follow.

The genesis block is fixed by the challenge bytes (mixed into the
genesis hash via the BIP325 commitment), so two networks with the same
challenge converge on the same genesis. Different challenge = different
genesis = different network.

## Worked example

Single-key custom signet end-to-end:

```bash
# 1. Generate a key
bitcoin-cli -regtest createwallet signer
ADDR=$(bitcoin-cli -regtest -rpcwallet=signer getnewaddress -addresstype bech32)
PUBKEY=$(bitcoin-cli -regtest -rpcwallet=signer getaddressinfo $ADDR | jq -r .pubkey)
# Privkey for the signer:
PRIV=$(bitcoin-cli -regtest -rpcwallet=signer dumpprivkey $ADDR)

# 2. Build a 1-of-1 challenge: <pk> OP_CHECKSIG, then wrap in length prefix
CHALLENGE=$(python3 -c "
import sys
pk = bytes.fromhex('$PUBKEY')
script = bytes([len(pk)]) + pk + bytes([0xac])   # OP_CHECKSIG
# signetchallenge wants the raw script in hex
print(script.hex())
")

# 3. Configure both signer and validator nodes
cat > /tmp/sig/bitcoin.conf <<EOF
signet=1
[signet]
signetchallenge=$CHALLENGE
fallbackfee=0.00001
EOF

bitcoind -datadir=/tmp/sig -daemon

# 4. Mine signet blocks via contrib/signet/miner
cd bitcoin/contrib/signet
./miner --cli="bitcoin-cli -datadir=/tmp/sig" generate \
  --address $ADDR \
  --grind-cmd="bitcoin-util grind" \
  --nbits=1d00ffff \
  --set-block-time=$(date +%s)

# Validator side (different machine, same challenge)
cat > /tmp/val/bitcoin.conf <<EOF
signet=1
[signet]
signetchallenge=$CHALLENGE
addnode=signer.example.com:38333
EOF
bitcoind -datadir=/tmp/val -daemon
```

Programmatically (Python, using the test framework signet helper):

```python
from test_framework.test_framework import BitcoinTestFramework
from test_framework.script import CScript, OP_CHECKSIG

class CustomSignetTest(BitcoinTestFramework):
    def set_test_params(self):
        self.num_nodes = 2
        self.chain = "signet"
        self.signet_challenge_bytes = CScript([
            bytes.fromhex("02..." ), OP_CHECKSIG
        ])
        self.extra_args = [
            [f"-signetchallenge={self.signet_challenge_bytes.hex()}"]
        ] * 2
```

## Common pitfalls

- Two nodes with different `signetchallenge` see different genesis
  hashes and silently fail to peer (handshake banner mismatch).
- The challenge must be a *script*, not a pubkey. `<pk> OP_CHECKSIG` is
  the minimum; bare pubkey is rejected.
- `nbits` of `1d00ffff` is the easy default; do not set it lower than
  `1e0377ae` or your signer wastes CPU grinding.
- Lost the signing key = network is dead. Use multisig challenges in
  production-style signets.
- Some block explorers do not auto-detect arbitrary signets; you may
  need a self-hosted Esplora pointed at your node.

## References

- BIP325: `https://github.com/bitcoin/bips/blob/master/bip-0325.mediawiki`
- Bitcoin Core `contrib/signet/` README and `miner` script
- Mutinynet operator notes: `https://blog.mutinywallet.com/`
