# Fedimint Guardian Multisig Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/fedimint`.
> Canonical source: https://github.com/fedimint/fedimint
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/fedimint/SKILL.md

## Concept

A Fedimint federation is a **community custodian**: members ("guardians")
collectively run a Chaumian e-cash mint backed by an on-chain BTC reserve.
Custody is enforced by a `t-of-n` Bitcoin multisig (typical 3-of-4 to 5-of-7),
with consensus over user requests via **AlephBFT** (the project moved off
Honey Badger BFT; AlephBFT is what `fedimint-server` runs as of v0.12.0,
August 2026). Users hold cryptographic blinded e-cash notes; the
federation cannot link withdrawals to deposits.

## Walkthrough / mechanics

### Setup ceremony

1. Each guardian generates an Ed25519 long-term identity key.
2. Guardians exchange and generate three key sets:
   - The **consensus broadcast keys**: one secp256k1 keypair per
     guardian, generated locally and swapped with the other peers
     during setup (`exchange_encodable(broadcast_pk)`), used to sign
     AlephBFT broadcast units (`broadcast_secret_key` /
     `broadcast_public_keys` in `fedimint-server` at v0.12.0). No DKG
     is involved.
   - The **on-chain peg key**: a set of per-guardian secp256k1 keys
     combined into a `wsh(sortedmulti(t, ...))` descriptor — a plain
     P2WSH `t-of-n` multisig, not a key-aggregation scheme. Verified
     against `fedimint-wallet-common` and `fedimint-walletv2-common`
     at tag v0.12.0 (27 August 2026); a single-guardian federation
     degrades to `wpkh(...)`.
   - The **mint blind-signature keys**: this is the one genuine
     **DKG**. The mint module runs `run_dkg_g2()` once per e-cash
     denomination, leaving every guardian a BLS12-381 secret share
     (`tbs_sks`) of that denomination's federation key, plus the
     matching `peer_tbs_pks` (`modules/fedimint-mint-server/src/lib.rs`
     at v0.12.0).
3. The result is broadcast in the federation's `client-config`: a
   document containing the on-chain script template, public consensus
   key, fee policies, and module list.

### Peg-in (deposit)

1. User scans a federation invoice or QR. The client derives a
   **per-deposit P2WSH output**: every guardian pubkey in the
   descriptor is tweaked by the same per-deposit tweak, so the
   deposit address is unlinkable to the federation's other addresses
   without knowing the tweak.
2. User funds the output on Bitcoin.
3. Guardians watch chain. When confs >= 6, consensus rounds:
   - One round to recognise the deposit and credit `(blinded_note,
     value)` requests.
   - Threshold-sign the blinded notes — an ad-hoc threshold blind
     signature scheme built on BLS signatures over the BLS12-381
     curve (`crypto/tbs` at v0.12.0), not RSA and not Schnorr-blind
     — so the user can unblind into spendable e-cash notes.

### E-cash mechanics

E-cash uses **blinded signatures**. All arithmetic below is in
BLS12-381, not RSA: `H(serial)` and every signature value are G1
points, `r` and the secret keys are scalars, and `pk_fed` lives in G2
(`crypto/tbs` at v0.12.0). To withdraw e-cash worth `v`:

1. User picks random scalar `r`, computes blinded value
   `B = r * H(serial)` (scalar multiplication of the G1 point).
2. Sends `B` to federation. Each guardian returns a share
   `S_i = sk_i * B`; `t` valid shares Lagrange-interpolate to
   `S = sk_fed * B`.
3. User unblinds: `s = r^-1 * S`, yielding signature on `serial`.
4. To spend, user reveals `(serial, s)`. Federation verifies with the
   pairing check `e(H(serial), pk_fed) == e(s, G2_generator)` and
   rejects if `serial` is in the spent set.

Properties: federation cannot link `serial` to the original `B` (blind
signature property). `t` or more guardians can collude to mint unbacked
e-cash; outsiders cannot detect this on chain, because only the peg
UTXO balance is public and the issued-note total is not (see Common
pitfalls).

### Peg-out (withdrawal)

1. User submits a Bitcoin scriptPubKey + e-cash payload worth `v`.
2. Federation verifies notes, marks them spent, and queues a Bitcoin tx
   spending peg UTXO to user.
3. Guardians each add an ordinary ECDSA signature to a shared PSBT;
   once `t` signatures are present the PSBT is finalized via
   miniscript into a P2WSH witness.
4. Tx broadcast. After 1 conf, user has BTC.

### Lightning gateway

Most peg-outs go via a **Lightning gateway** (separate role): a
federation member or third-party node that swaps in-federation e-cash
for Lightning payments. This keeps on-chain footprints rare. See
`lightning-gateway-flow.md`.

### Consensus failure modes

- **`t` guardians offline**: federation halts. Recovery requires
  replacement guardian via SIP-style social process.
- **Equivocation**: the BFT layer detects double-proposal; the
  offending guardian's identity key is logged. No automatic slash.

## Worked example

3-of-4 Fedimint with peg-out:

```
Bitcoin peg UTXO (P2WSH sortedmulti 3-of-4): 5.0 BTC
User redeems 0.1 BTC e-cash.

Bitcoin tx:
  Input  : peg UTXO 5.0
  Output : user 0.1
  Output : new peg UTXO 4.8999 (after fee)
Witness : 3 x ECDSA sigs + the 3-of-4 witness script
```

The spend reveals the witness script, so the federation's threshold
and guardian count become public on chain; only the per-deposit tweak
keeps deposit addresses from being trivially clustered. Fedimint does
not hide the multisig behind key aggregation (checked at v0.12.0,
August 2026).

## Common pitfalls

- **No on-chain audit of e-cash**: guardians know the total minted but
  outsiders see only the peg UTXO balance. A federation can pretend
  reserves are healthy while issuing e-cash beyond reserves; mitigated
  by transparency tooling (regular signed reserve attestations).
- **Do not assume key aggregation**: through v0.12.0 (August 2026)
  both the `wallet` and `walletv2` modules build a P2WSH
  `sortedmulti` descriptor and collect one ordinary ECDSA signature
  per guardian. There is no MuSig2, no Taproot peg output and no
  threshold-ECDSA (GG18) ceremony on the on-chain path, so peg-out
  witness size and fee grow linearly with the threshold `t`.
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
- `modules/fedimint-walletv2-common/src/lib.rs` at tag v0.12.0 —
  the `descriptor()` / `tweak_public_key()` peg construction.
- `crypto/tbs/src/lib.rs` at tag v0.12.0 — the BLS12-381 threshold
  blind signature scheme behind the e-cash notes.
