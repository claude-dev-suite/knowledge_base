# sBTC Peg Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/stacks`.
> Canonical source: https://github.com/stacksgov/sips/blob/main/sips/sip-024/sip-024-bitcoin-finality.md and SIP-021/-026 sBTC
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/stacks/SKILL.md

## Concept

sBTC is a 1:1 BTC-pegged asset on the Stacks chain. Unlike trusted
wrapped-BTC (e.g. WBTC on Ethereum), sBTC custody is held by a rotating
**threshold-signature signer set** — a permissioned group of institutional
operators, *not* a set elected from Stackers. Pegging in ("deposit") is
permissionless; pegging out ("withdrawal") triggers a threshold-signed
Bitcoin tx returning native BTC.

Mainnet timeline: the sBTC contracts were deployed 13 December 2024 and
the first completed deposit landed 14 December 2024. The earliest
accepted withdrawal request on mainnet is 25 March 2025, but it is an
isolated cluster: `sbtc-withdrawal` carries under 20 transactions in
total before 30 April 2025, then 36 on that day alone. Peg-outs opened in
practice on 30 April 2025. These dates read from the mainnet contract
histories on 15 September 2026.

## Walkthrough / mechanics

### Signer set

The sBTC signer set is a small **permissioned** group of institutional
operators. It is *not* the Stacking-elected Nakamoto block-signer set
(reward cycle 143, September 2026, carried 29 block-signer entries) —
conflating the two is the most common error about sBTC. Stacks' own
marketing language calls the sBTC signers community-elected, describing
Phase 1 as an initial set of ~15 signers approving transactions at a
~70 % consensus threshold with expansion deferred to a later phase; entry
is nonetheless gated by the incumbent set, which alone may call the
rotation function.

Live values read from the mainnet registry contract
`SM3VDXK3WZZSA84XXFKAFAF15NNZX32CTSG82JFQ4.sbtc-registry` on
15 September 2026:

- `current-signer-set` — 11 signer pubkeys.
- `current-signature-threshold` — `u8`, i.e. **8-of-11** (~73 %).

History of the `rotate-keys-wrapper` calls: 14 keys / threshold `u10`
(10-of-14, ~71 %) from the 13 December 2024 launch through the
11 February 2026 rotation, then 11 keys / threshold `u8` from the
27 July 2026 rotation onward (most recent rotation 24 August 2026).
The Clarity guard only enforces `threshold > len(keys)/2` and
`threshold <= len(keys)`; the ~70 % figure is signer policy, not a
contract constant.

They run **WSTS (Wrapped Schnorr Threshold Signatures)**, a FROST-style
Schnorr threshold scheme over secp256k1. The aggregated signer pubkey
`P_signers` is published in the `current-aggregate-pubkey` chainstate
variable.

### Peg-in deposit

1. User chooses amount `X BTC`. They construct a Bitcoin tx with a
   single **P2TR deposit output** whose taproot tree holds two
   leaves — there is *no* OP_RETURN in a deposit transaction
   (sBTC 1.3.4, the current release as of September 2026):
   - **Deposit leaf**: `<deposit-data> OP_DROP OP_PUSHBYTES_32
     <P_signers x-only> OP_CHECKSIG`, where `deposit-data` is an 8-byte
     big-endian *max signer fee* followed by the SIP-005 recipient
     principal (22 bytes for a standard Stacks address, longer for a
     contract principal).
   - **Reclaim leaf**: the depositor's escape hatch, typically
     `<lock-time> OP_CSV OP_DROP <depositor pubkey> OP_CHECKSIG`. Only
     the `<lock-time> OP_CSV` prefix is fixed — the tail is any
     user-supplied script, capped at `MAX_RECLAIM_SCRIPT_LENGTH` =
     2048 bytes and rejected if it contains an OP_SUCCESSx opcode
     (sBTC 1.3.4, September 2026).
2. User broadcasts on Bitcoin and registers the outpoint plus the raw
   deposit and reclaim scripts with the **Emily** API, which is how
   signers learn the recipient; they re-derive the P2TR address and
   reject the request if it does not match the output on chain.
3. Stacks signers monitor Bitcoin via their full nodes. After 6 confs,
   the next Stacks miner includes a **mint event** in a Stacks block,
   crediting the recipient principal with `X` sBTC.
4. The Bitcoin UTXO joins a multi-billion-sat "treasury" UTXO under
   threshold control.

### Peg-out withdrawal

1. User sends `X` sBTC to the sBTC contract's `request-withdrawal`
   function on Stacks, supplying their Bitcoin destination address.
2. Stacks burns the sBTC. The withdrawal request is queued in a
   Clarity map.
3. In the next sortition window, signers run a FROST signing ceremony
   over a Bitcoin tx that:
   - Spends part of the treasury UTXO.
   - Creates outputs: `X BTC` to user + change back to a fresh
     `P_signers` (pubkey rotated at each key rotation).
4. Threshold-signed tx is broadcast on Bitcoin. After confs, the
   Stacks chain marks the request "fulfilled".

### Signer rotation

Rotation is **not** automatic and not tied to reward-cycle boundaries: it
is an explicit `rotate-keys-wrapper` call on `sbtc-bootstrap-signers`,
access-controlled so that only the *current* signer principal may invoke
it. Mainnet saw seven successful rotations between 13 December 2024 and
24 August 2026 — every few months, not every cycle. Permissionless
rotation (arbitrary operators qualifying into the set) is still
outstanding as of September 2026. Sequence:

1. Incoming signers run a DKG (Distributed Key Generation) to
   compute the next `P_signers'`.
2. Outgoing signers sign one final Bitcoin tx that **transfers the
   treasury UTXO** to a new output paying `P_signers'`. This rolls
   custody forward without any user interaction.
3. Failure mode: if old signers refuse to rotate, treasury freezes.
   Recovery requires emergency Stacker intervention via SIP.

### Security model

- The signing threshold is the entire security budget: **8 of the 11
  current signers** must collude to steal the treasury (as of
  15 September 2026; 10-of-14 before the July 2026 rotation). No fraud
  proofs; because the signers are named institutions rather than elected
  Stackers, the disincentive is reputational and contractual — there is
  no automatic slashing contract.
- Bitcoin clients verifying sBTC mint/burn events use an SPV-like proof
  embedded in Stacks anchor blocks (Bitcoin block header + Merkle path).

## Worked example

Alice deposits 0.5 BTC.

```
Bitcoin tx:
  Input  : Alice UTXO 0.51 BTC
  Output : P2TR 0.5 BTC, tapleaves =
             deposit: <max_fee(8B BE) || alice_principal> OP_DROP
                      OP_PUSHBYTES_32 <P_signers> OP_CHECKSIG
             reclaim: <lock_time> OP_CSV OP_DROP <alice_pk> OP_CHECKSIG
                      (only <lock_time> OP_CSV is fixed; the tail is
                       any user script up to 2048 bytes)
  Output : Alice change ~0.0099
  (no OP_RETURN output; the two scripts go to Emily off-chain)
```

After 6 BTC confs and the next Stacks block, Alice's STX address holds
`0.5 sBTC` (8-decimal token). She can transfer, swap, or use it in
DeFi. Later she peg-outs:

```
Stacks tx: contract-call? sbtc-token request-withdrawal u50000000 'bc1q...alice'
=> burns 0.5 sBTC, queues withdrawal
After signer ceremony => Bitcoin tx pays Alice 0.5 BTC - fee
```

## Common pitfalls

- **Deposit script encoding**: a deposit commits its payload in a
  *taproot leaf*, not an OP_RETURN. The `sbtc` crate parses exactly
  `<max-fee||principal> OP_DROP OP_PUSHBYTES_32 <pubkey> OP_CHECKSIG`
  and rejects everything else, including a non-minimal `OP_PUSHDATA1`
  push of fewer than 76 bytes. A malformed script simply never mints;
  Alice recovers by spending the reclaim leaf once its `OP_CSV`
  lock-time expires (sBTC 1.3.4, September 2026).
- **The only OP_RETURN in sBTC is the signers' sweep output**: output 1
  of a sweep carries 2 magic bytes + 1 version byte + idpack-encoded
  withdrawal IDs, capped at **80 bytes** by the signer's own
  `OP_RETURN_MAX_SIZE`. That cap is sBTC's own, not Bitcoin's — Bitcoin
  Core 30.0 (13 October 2025) raised default `-datacarriersize` to
  100,000 and now relays multiple OP_RETURN outputs, but sBTC 1.3.4
  (September 2026) still refuses to build a sweep exceeding 80 bytes.
- **Signer collusion / liveness**: if signers are offline, withdrawals
  stall. There is currently no on-chain timeout that returns BTC to
  user. Reputational + social slashing is the main defence.
- **Rotation reorg**: if the rotation tx is reorged out, the new signer
  set has no UTXO to spend. Implementations require deep BTC confs
  (≥ 100 blocks) before considering rotation final.
- **Address-type compatibility**: peg-out can target any Bitcoin
  scriptPubKey; signers' tx must respect dust limits and policy. Some
  unusual scripts (legacy P2PKH with non-standard) may be filtered.
- **MEV via mempool sniping**: a malicious signer can withhold a
  withdrawal for fee bribing. WSTS aggregator and committee assignment
  are designed to randomise signer roles and reduce this.

## References

- SIP-021, SIP-024, SIP-026 (Stacks SIPs for Nakamoto, Bitcoin finality, sBTC).
- WSTS paper — "Threshold Schnorr with stateless deterministic signing" 2023.
- sBTC reference implementation — https://github.com/stacks-sbtc/sbtc
  (v1.3.4, 1 September 2026): `sbtc/src/deposits.rs` for the deposit and
  reclaim script formats, `sbtc/src/lib.rs` for
  `MAX_RECLAIM_SCRIPT_LENGTH`, `signer/src/bitcoin/utxo.rs` for the
  sweep OP_RETURN layout.
