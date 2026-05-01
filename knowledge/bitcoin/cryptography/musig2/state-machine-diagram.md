# MuSig2 State Machine Diagram - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/cryptography/musig2`.
> Canonical source: BIP327 sec. "Signing" + libsecp256k1 musig API
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/cryptography/musig2/SKILL.md

## Concept

A MuSig2 signer is a finite state machine with strict, non-cyclic transitions.
A correct implementation refuses to advance unless every prerequisite for the
next state is present, and refuses to revisit any state once left. Violating
this discipline (e.g., re-sending Round-1 nonces, or signing Round 2 twice with
the same secnonce) is what triggers the catastrophic key leak from
[attacks.md](../../../skills/bitcoin/cryptography/musig2/quick-ref/attacks.md).
This article maps each state, the inputs/outputs, the persistence requirements,
and the legal/illegal transitions.

## Walkthrough / mechanics

States and transitions for one signer in one session:

```
   [INIT]
      |  set sk, msg, signer set X[]
      v
   [KEY_AGG_DONE]
      |  KeyAgg(X) -> KeyAggCtx (Q, gacc, tacc)
      |  optional: ApplyTweaks(...)
      v
   [NONCE_GEN]
      |  secnonce = NonceGen(sk, Q, msg, extra_in, rand)
      |  emit pubnonce; PERSIST secnonce; mark `used=false`
      v
   [NONCE_AGG_WAIT]
      |  receive pubnonces from peers
      v
   [SESSION_CTX_BUILT]
      |  aggnonce = NonceAgg(pubnonces)
      |  SessionCtx = (aggnonce, X, tweaks, msg)
      v
   [SIGN_READY]
      |  s_i = Sign(secnonce, sk, SessionCtx)
      |  IMMEDIATELY mark secnonce.used=true and zeroize
      v
   [PARTIAL_SIG_SENT]
      |  await peer partial sigs
      v
   [AGG_SIG_DONE]   (final 64-byte BIP340 sig under Q)
```

Constraint table:

```
| From state              | Legal next            | Illegal           |
|-------------------------|-----------------------|-------------------|
| INIT                    | KEY_AGG_DONE          | any other         |
| KEY_AGG_DONE            | NONCE_GEN             | re-key-agg, sign  |
| NONCE_GEN               | NONCE_AGG_WAIT        | re-NonceGen       |
| NONCE_AGG_WAIT          | SESSION_CTX_BUILT     | sign without aggn |
| SESSION_CTX_BUILT       | SIGN_READY            | re-aggregate      |
| SIGN_READY              | PARTIAL_SIG_SENT      | sign twice        |
| PARTIAL_SIG_SENT        | AGG_SIG_DONE          | re-sign           |
| AGG_SIG_DONE            | (terminal; new sess.) | any reuse         |
```

Two transitions are **destructive** and must be encoded as one-way
transformations in code:

1. `NONCE_GEN -> NONCE_AGG_WAIT`: persist secnonce so a crash does not lose it.
2. `SIGN_READY -> PARTIAL_SIG_SENT`: zeroize secnonce *before* releasing the
   partial sig on the wire. If the wire send fails after zeroize, the session
   is permanently dead — restart with fresh nonces. This is correct behavior.

## Worked example

libsecp256k1's musig API maps each state to a struct, and rejects out-of-order
calls at the C type level:

```c
secp256k1_musig_keyagg_cache cache;
secp256k1_musig_secnonce     secnonce;
secp256k1_musig_pubnonce     pubnonce;
secp256k1_musig_aggnonce     aggnonce;
secp256k1_musig_session      session;
secp256k1_musig_partial_sig  partial;

// INIT -> KEY_AGG_DONE
secp256k1_musig_pubkey_agg(ctx, NULL, &agg_pk, &cache, pks, n);

// KEY_AGG_DONE -> NONCE_GEN
secp256k1_musig_nonce_gen(ctx, &secnonce, &pubnonce, session_id,
                          sk, &my_pk, msg32, &cache, extra_in);
// PERSIST secnonce here. If the process dies, on restart the only
// safe action is to abandon this session — secnonce is single-use.

// NONCE_AGG_WAIT -> SESSION_CTX_BUILT
secp256k1_musig_nonce_agg(ctx, &aggnonce, pubnonces, n);
secp256k1_musig_nonce_process(ctx, &session, &aggnonce, msg32, &cache);

// SESSION_CTX_BUILT -> SIGN_READY -> PARTIAL_SIG_SENT
// CRITICAL: this call CONSUMES secnonce. Internally it zeroes it.
secp256k1_musig_partial_sign(ctx, &partial, &secnonce, &keypair,
                             &cache, &session);
// secnonce is now scrambled — the function clobbers it on return.

// PARTIAL_SIG_SENT -> AGG_SIG_DONE (coordinator)
secp256k1_musig_partial_sig_agg(ctx, sig64, &session, partials, n);
```

The deliberate clobbering of `secnonce` inside `partial_sign` is the C-level
enforcement of the one-way arrow `SIGN_READY -> PARTIAL_SIG_SENT`. Calling
`partial_sign` a second time on the same `secnonce` produces nonsense, but more
importantly cannot leak the key, because the secret material is no longer there.

## Common pitfalls

- **Async signers serializing secnonce wrong.** A Lightning-style flow may
  serialize the entire session to disk between rounds. `secnonce` MUST be
  encrypted at rest and bound to a session UUID; restoring on a different
  session corrupts state.
- **Retry on partial-sig publish failure.** If the wire send fails after
  `partial_sign` returns, do NOT re-call `partial_sign`. Either re-broadcast
  the already-computed `partial` (it is deterministic for a frozen state) or
  abort the session and restart at INIT.
- **Reusing `KeyAggCtx` across sessions.** Safe — KeyAgg has no secret state.
  But `secnonce` and `session` MUST be fresh per message.
- **Out-of-order tweak application.** `ApplyTweaks` must run before
  `nonce_process`; doing it after produces a `Q` mismatch in the challenge.

## References

- BIP327: https://github.com/bitcoin/bips/blob/master/bip-0327.mediawiki
- libsecp256k1 musig API: `include/secp256k1_musig.h`
- musig2 paper (Nick / Ruffing / Seurin, 2020): https://eprint.iacr.org/2020/1261
- rust-secp256k1 musig: https://docs.rs/secp256k1/latest/secp256k1/musig/
