# Blockstream Jade Pinserver Flow - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/hardware/jade`.
> Canonical source: https://github.com/Blockstream/blind_pin_server
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/hardware/jade/SKILL.md

## Concept

Jade does not have a Secure Element. It runs on an ESP32, whose
encrypted-memory protections are good but not certified to fend off
sophisticated physical attacks (glitching, side channels). To compensate,
Blockstream designed a **Pinserver**: a remote service that participates
in the unlock protocol so that even if the attacker has the device, they
cannot brute-force the PIN locally.

The Pinserver is **blind**: it never learns the seed, the PIN, or even
whether a PIN attempt was correct. It just stores a per-device encrypted
key and rate-limits requests. Blockstream runs the canonical instance,
but it's open-source and self-hostable.

## Walkthrough

Setup flow (initial):

1. User chooses PIN on Jade (4-6 digits).
2. Jade: generate `aes_key = TRNG(256 bits)`. This will encrypt the seed
   in flash.
3. Jade derives `client_secret_pin` from PIN via Argon2id.
4. Jade contacts Pinserver with `pubkey_jade, ECDH-handshake_request`.
5. Pinserver replies with `pubkey_server, server-state`.
6. Jade computes `pinkey = HKDF(ECDH(jade_priv, server_pub) || client_secret_pin)`.
7. Jade encrypts `aes_key` with `pinkey`, stores ciphertext in flash.
8. Server stores per-device record indexed by `device_id_hash`.

Unlock flow:

1. User enters PIN guess on Jade.
2. Jade: rebuild `client_secret_pin` from guess via Argon2id.
3. Jade contacts Pinserver: "I'm `device_id`, here's the ECDH for this
   session".
4. Pinserver checks rate limit (e.g., 3 attempts then exponential
   backoff). If OK, returns its half of the secret.
5. Jade combines, attempts to decrypt `aes_key` from flash.
6. If decryption succeeds (MAC validates) -> seed unlocked.
7. If MAC fails -> wrong PIN; Jade shows error.

Crucially, the Pinserver cannot tell whether a guess was right - it just
sends its half. Wrong PIN means MAC failure on Jade's side.

## Worked example

### Self-hosted Pinserver

```bash
# Clone and run
git clone https://github.com/Blockstream/blind_pin_server
cd blind_pin_server
docker compose up -d
# Default port 8096

# Point Jade at it
# Settings -> Network -> Custom Pinserver -> "https://my-pinserver.local:8096"
```

### Programmatic interaction (jadepy)

```python
from jadepy.jade import JadeAPI

with JadeAPI.create_serial(device='/dev/ttyUSB0') as jade:
    # Standard unlock - jadepy handles Pinserver negotiation
    # under the hood when the user enters PIN on device.
    jade.auth_user(network='mainnet')
    # Once unlocked, can sign etc.
    xpub = jade.get_xpub('mainnet', "m/84h/0h/0h")
```

If the device is configured for "Custom Pinserver", jadepy honors that;
otherwise it uses Blockstream's at `https://jadepin.blockstream.com`.

### Offline-only mode

Skip the Pinserver entirely:

1. Settings -> Network -> "Offline only".
2. Jade unlocks purely with on-device-derived material.
3. Trade-off: no rate limiting; an attacker with physical access can
   brute-force the PIN faster.

## Common pitfalls

- **Pinserver dependency**: if `jadepin.blockstream.com` is unreachable
  (Tor down, ISP blocking), unlock fails. Jade has fallback timeout
  but if you live somewhere with bad connectivity, set up
  `--offline-only` or self-host.
- **Self-hosted Pinserver loss**: if you self-host and the server data
  is destroyed, ALL Jades enrolled to it are bricked PIN-wise (no
  rate-limit half). Always back up the server's database.
- **Tor recommended**: default Pinserver runs over HTTPS. Use Tor onion
  service for additional metadata privacy; Jade has Tor mode.
- **Re-enrollment** when switching Pinservers: changing the configured
  Pinserver requires you to re-enroll the device, which wipes flash
  state. Funds aren't lost (seed is in user's backup) but device must
  be re-initialized.
- **Confusion with "Liquid Pinserver"**: Liquid mode shares the same
  Pinserver infrastructure. No separate Liquid-only server.

## References

- Blind Pinserver: https://github.com/Blockstream/blind_pin_server
- Jade firmware repo: https://github.com/Blockstream/Jade
- Jade hardware paper: https://help.blockstream.com/hc/en-us/articles/4406560791833
