# Coldcard Trick PIN Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/coldcard`.
> Canonical source: https://coldcard.com/docs/trick-pins/
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/coldcard/SKILL.md

## Concept

Trick PINs let a Coldcard owner set up secondary PINs that, when
entered under coercion, do something **other than** unlock the real
wallet. The Mk4 / Q both ship with this; the SE608A secure element
stores up to ~14 trick slots with per-slot policies.

Common trick policies:

- **Wipe wallet** - silently erase the seed, then either reboot or
  pretend to unlock to an empty fresh seed.
- **Brick device** - permanently disable the SE.
- **Login countdown** - delay each attempt (12 hours, 1 day, etc.) so
  attackers can't try many PINs.
- **Decoy wallet** - unlock to a secondary BIP-39 seed (small balance)
  while the real wallet stays hidden.
- **Just say "wrong PIN"** - log the attempt, do nothing.

These are enforced inside the SE itself. If the host (or attacker)
modifies firmware, the SE still applies its trick policy because the
PIN-check happens in SE silicon, not in the main MCU.

## Walkthrough

Setup via on-device menu (Mk4 or Q):

```
Settings -> Login Settings -> Trick PINs -> Add New Trick

Enter Trick PIN:    9999-1234
Pick action:        Wipe + Reboot
                    Wipe + Reboot to alt seed
                    Brick the Coldcard
                    Just say "wrong"
                    Login countdown (delay)
                    Login + decoy
Confirm trick:      yes
```

Setting a decoy:

```
Add New Trick -> 1234-5678 -> "Login + decoy"
Setup decoy seed:
  Generate new (24 words)
  Or import existing BIP-39
Decoy now active under trick PIN.
```

Internally, the SE stores per-slot:

```
slot[i] = {
  pin_hash:      SHA256(salt || pin_attempt),
  policy_flags:  WIPE | BRICK | DECOY | DELAY,
  decoy_seed:    [encrypted blob, only present if DECOY set],
  delay_seconds: u32,
  attempt_log:   counter
}
```

On a PIN entry, the SE iterates: real PIN match -> normal unlock;
trick match -> apply that slot's policy and report fake success/failure
to the MCU.

## Worked example

A duress scenario: someone forces you to unlock the device. You enter
`1234-5678` (the decoy trick PIN).

1. SE matches trick slot 3, action = "Login + decoy".
2. SE silently swaps seed material to the decoy seed.
3. MCU boots into normal Coldcard UI showing decoy wallet (small btc).
4. Attacker sees a "real-looking" wallet, sends test tx.
5. After ejection, the next time you enter your real PIN, the original
   seed is restored seamlessly.

Verification you set this up correctly: before live use, test by
entering each trick PIN on a non-funded device. Verify the wipe-and-
reboot does not leave seed traces in dump:

```bash
ckcc dump-state          # via ckcc-protocol; requires Coldcard's
                         # advanced developer mode
```

## Common pitfalls

- **Trick PIN shares a prefix with real PIN**: the device requires
  significant divergence in the digit pattern; otherwise typos can
  trigger trick actions. Use distinct prefixes.
- **Forgetting the trick PIN**: there is no "list trick PINs" without
  knowing the master PIN. Save them in your existing secure backup.
- **Decoy seed stored in plaintext on SE**: anyone who extracts the SE
  (very hard, but not impossible) can read both. Consider: the decoy
  is a denial-of-knowledge tool, not a secrecy tool.
- **Wipe-on-fail counters**: `--login-countdown` policies that wipe
  after N failures can permanently destroy the wallet if you fat-finger
  several times. Always have your seed backed up first.
- **HSM mode interaction**: in HSM mode, trick PINs may behave
  differently (some policies disabled). Read coldcard docs for current
  HSM matrix.

## References

- Coldcard trick PIN docs: https://coldcard.com/docs/trick-pins/
- ckcc-protocol Python: https://github.com/Coldcard/ckcc-protocol
- Coldcard firmware repo: https://github.com/Coldcard/firmware
