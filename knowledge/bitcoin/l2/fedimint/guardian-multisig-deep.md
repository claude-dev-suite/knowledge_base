# Fedimint Guardian Multisig Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/fedimint`.
> Canonical source: https://github.com/fedimint/fedimint
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/fedimint/SKILL.md

## Concept

A Fedimint federation is a **community custodian**: members ("guardians")
collectively run a Chaumian e-cash mint backed by an on-chain BTC reserve.
Custody is enforced by a `t-of-n` Bitcoin multisig (typical 3-of-4 to 5-of-7),
with consensus over user requests via **HBBFT** (Honey Badger BFT). Users
hold cryptographic blinded e-cash notes; the federation cannot link
withdrawals to deposits.

## Walkthrough / mechanics

### Setup ceremony

1. Each guardian generates an Ed25519 long-term identity key.
2. Guardians run **DKG** for two key sets:
   - The **HBBFT consensus key** (BLS threshold for proposals).
   - The **on-chain peg key**: a Bitcoin Taproot MuSig2 (or threshold
     ECDSA depending on version) `t-of-n` aggregated key.
3. The result is broadcast in the federation's `client-config`: a
   document containing the on-chain script template, public consensus
   key, fee policies, and module list.

### Peg-in (deposit)

1. User scans a federation invoice or QR. The client derives a
   **per-deposit Taproot output**: a tweaked version of the federation
   key that commits to the user's blinded notes.
2. User funds the output on Bitcoin.
3. Guardians watch chain. When confs >= 6, consensus rounds:
   - One round to recognise the deposit and credit `(blinded_note,
     value)` requests.
   - Threshold-sign the blind signatures (Chaumian RSA or Schnorr-blind)
     so the user can unblind into spendable e-cash notes.

### E-cash mechanics

E-cash uses **blinded signatures**. To withdraw e-cash worth `v`:

1. User picks random `r`, computes blinded value `B = r * H(serial)`.
2. Sends `B` to federation. Guardians threshold-sign `B`, producing
   `S = sk_fed * B`.
3. User unblinds: `s = r^-1 * S`, yielding signature on `serial`.
4. To spend, user reveals `(serial, s)`. Federation verifies `s` against
   `pk_fed` and rejects if `serial` is in the spent set.

Properties: federation cannot link `serial` to the original `B` (blind
signature property). Guardians can collude to mint "fake" e-cash but
this is detectable via on-chain reserve audit.

### Peg-out (withdrawal)

1. User submits a Bitcoin scriptPubKey + e-cash payload worth `v`.
2. Federation verifies notes, marks them spent, and queues a Bitcoin tx
   spending peg UTXO to user.
3. Guardians run threshold MuSig2 ceremony; partial sigs aggregate
   into one Schnorr signature.
4. Tx broadcast. After 1 conf, user has BTC.

### Lightning gateway

Most peg-outs go via a **Lightning gateway** (separate role): a
federation member or third-party node that swaps in-federation e-cash
for Lightning payments. This keeps on-chain footprints rare. See
`lightning-gateway-flow.md`.

### Consensus failure modes

- **`t` guardians offline**: federation halts. Recovery requires
  replacement guardian via SIP-style social process.
- **Equivocation**: HBBFT detects double-proposal; offending guardian's
  identity key is logged. No automatic slash.

## Worked example

3-of-4 Fedimint with peg-out:

```
Bitcoin peg UTXO (P2TR aggregated key Q): 5.0 BTC
User redeems 0.1 BTC e-cash.

Bitcoin tx:
  Input  : peg UTXO 5.0
  Output : user 0.1
  Output : new peg UTXO 4.8999 (after fee)
Witness : 64-byte Schnorr (MuSig2 aggregated)
```

Looks like an ordinary single-key taproot spend: privacy preserved on
chain.

## Common pitfalls

- **No on-chain audit of e-cash**: guardians know the total minted but
  outsiders see only the peg UTXO balance. A federation can pretend
  reserves are healthy while issuing e-cash beyond reserves; mitigated
  by transparency tooling (regular signed reserve attestations).
- **Threshold ECDSA legacy modules**: some early Fedimint versions used
  GG18 threshold ECDSA, which is heavier and has historic bug surface.
  Modern versions use BIP327 MuSig2 + Taproot.
- **Note expiry**: clients must back up e-cash notes; lost wallet =
  lost notes since federation cannot recover (privacy property).
  Some implementations support **note recovery** via guardian-encrypted
  backup of `serial` lists.
- **Federation rebalance** between cold/hot peg UTXOs requires a
  consensus round. Avoid splitting reserve into many UTXOs unless
  necessary.

## References

- Fedimint whitepaper (Sirion Labs / fedimint.org).
- David Chaum 1982 — original blind signatures.
- BIP327 — MuSig2 threshold Schnorr.
