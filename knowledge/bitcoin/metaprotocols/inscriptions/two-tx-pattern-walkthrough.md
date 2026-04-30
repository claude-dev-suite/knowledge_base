# Inscription Two-Tx (Commit/Reveal) Pattern - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/metaprotocols/inscriptions`.
> Canonical source: https://docs.ordinals.com/inscriptions.html
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/metaprotocols/inscriptions/SKILL.md

## Concept

Inscriptions cannot be embedded in a single transaction because the
envelope must appear in a Taproot script-path witness, and a Taproot
output's script tree is committed to before it exists. The ord
workflow therefore splits the operation in two: a **commit** tx
that pays to a Taproot address committing to the envelope script
tree, then a **reveal** tx that spends that output along the script
path, exposing the envelope on chain. The reveal tx output sits on
the dust-limit at 546 sats and "carries" the inscription via ordinal
attribution.

## Walkthrough / mechanics

```
[funding UTXO] -- commit tx ---> [P2TR(commit_addr) = 546+fees sats]
                                          |
                                          v
                                 [reveal tx, script-path]
                                          |
                                          v
                                 [P2TR(receiver) = 546 sats]  <-- inscription here
```

Construction steps:

1. Generate a fresh keypair `(d, P)`. Build the leaf script
   `P OP_CHECKSIG <envelope>`.
2. Build the Taproot output key as `Q = P_internal + t*G` where
   `t = TaggedHash("TapTweak", P_internal || merkle_root)` and the
   merkle root is just the single leaf hash (no tree, since one leaf).
   ord uses an unspendable internal key (h(G) per BIP341) so that
   only the script path is reachable.
3. **Commit tx**: send `546 + reveal_fee` sats to `bc1p<Q>`.
4. Wait for at least one confirmation (technically not required, but
   reduces footgun risk).
5. **Reveal tx**: input spends the commit output via script path.
   Witness stack: `[signature, leaf_script, control_block]`.
6. Output of the reveal tx is a single P2TR worth 546 sats whose
   first sat owns the inscription.

## Worked example  (encoded data, real txs, configs)

Bytes layout for a simple reveal witness:

```
witness[0] = <64-byte BIP340 schnorr signature for sighash=DEFAULT>
witness[1] = <leaf script: P OP_CHECKSIG OP_FALSE OP_IF "ord" 1 "text/plain" 0 "hello" OP_ENDIF>
witness[2] = <control block: 0xc0 || P_internal || merkle_path>
```

For one leaf the merkle_path is empty, so the control block is exactly
33 bytes (1 byte version+parity || 32 byte internal pubkey).

Commit tx fee math at 50 sat/vB:

```
commit tx vsize  = ~110 vB         -> 5_500 sats
reveal tx vsize  = ~150 vB (with 100-byte envelope)
                                     -> 7_500 sats
output dust       = 546 sats
total funding     = 13_546 sats
```

A real commit/reveal pair on testnet:

```
commit:  d6e5...c1a   value 13_546   to bc1p<Q>
reveal:  9f02...abc   spends commit  produces 546-sat output
                                     at bc1p<recv>; witness contains envelope
```

ord CLI generates the pair with one command:

```
$ ord wallet inscribe --fee-rate 50 --file art.png
{
  "commit": "d6e5...c1a",
  "inscription": "9f02...abci0",
  "reveal":  "9f02...abc",
  "total_fees": 13546
}
```

## Trade-offs / pitfalls

- The reveal tx must spend the commit output exactly. If you spend a
  different output that happens to have the same script tree (none
  exists, but the failure mode is "no envelope reaches chain"), no
  inscription is created.
- The first sat of the first input of the reveal tx becomes the
  inscription's owning sat. If you mistakenly add other inputs, those
  sats end up before the inscription and the inscription rides on
  whatever sat the input contributes - usually fine but breaks rare-
  sat planning.
- A commit output left unrevealed is a stuck UTXO: the script tree
  must be remembered to spend it. Lose the keys and the sats are
  gone.
- Replace-by-fee on the reveal tx requires constructing a new reveal
  with higher fee that still spends the same commit output. Ord
  supports this; bumping the commit only is useless.
- Mempool stuck: if the commit confirms but the reveal sits below
  the next-block fee floor, you must CPFP-bump the reveal. The
  inscription is not "live" until the reveal mines.
- Anti-pattern: batching unrelated inscriptions into a single commit
  output is fine but ties their fates together. A failed reveal kills
  every inscription in that commit.

## References

- ord inscribe flow: https://docs.ordinals.com/guides/inscriptions.html
- BIP341 script-path spend: https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki
- ord wallet source: https://github.com/ordinals/ord/blob/master/src/wallet/inscribe.rs
