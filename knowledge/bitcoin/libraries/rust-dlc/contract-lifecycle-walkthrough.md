# rust-dlc Contract Lifecycle - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/libraries/rust-dlc`.
> Canonical source: https://github.com/p2pderivatives/rust-dlc
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/libraries/rust-dlc/SKILL.md

## Concept

A Discreet Log Contract (DLC) is a 2-of-2 multisig funding output spent
by one of N pre-signed Contract Execution Transactions (CETs). Which
CET becomes valid is decided by a Bitcoin oracle that publishes a
Schnorr attestation; the attestation is the missing scalar that turns
an adaptor signature into a valid signature for exactly one CET.

`rust-dlc` splits responsibilities:

- `dlc` -- low-level types: CET construction, oracle messages, adaptor
  application.
- `dlc-messages` -- BOLT-style wire messages: `Offer`, `Accept`, `Sign`.
- `dlc-trie` -- compresses many CETs (numerical outcomes) via a digit
  decomposition tree.
- `dlc-manager` -- orchestrates the full state machine across blockchain,
  wallet, signing, and oracle interfaces.

The protocol has three message exchanges (offer / accept / sign), one
funding broadcast, then -- much later -- one settlement broadcast using
the oracle attestation, or a refund tx after the timelock.

## API walkthrough

```rust
use dlc_manager::{
    contract::contract_input::ContractInput, manager::Manager,
};
use dlc_messages::Message;

fn pump_msg(manager: &mut Manager<...>, msg: Message,
            counter_party: secp256k1::PublicKey) -> anyhow::Result<()> {
    if let Some(reply) = manager.on_dlc_message(&msg, counter_party)? {
        // send `reply` over your transport (Lightning peer msg, HTTP, etc.)
    }
    Ok(())
}

// Offer side
let contract_input: ContractInput = serde_json::from_str(
    include_str!("./contract.json"))?;
let offer = manager.send_offer(&contract_input, counter_party_pk)?;
// transmit `offer` to counter-party

// Inbound messages (Accept, Sign) are pumped through `on_dlc_message`
// which runs validation, signing, and persistence atomically.
```

`ContractInput` is the human-authored description (collateral, fee rate,
oracle pubkey + nonce, payout function). The manager turns it into
funding and CETs after the offer/accept/sign handshake.

## Worked example: settle with an oracle attestation

```rust
use dlc_manager::Storage;

fn settle(manager: &mut Manager<...>, contract_id: [u8; 32],
          attestation_bytes: &[u8]) -> anyhow::Result<()> {
    let oracle_attestation: dlc_messages::oracle_msgs::OracleAttestation =
        bitcoin::consensus::deserialize(attestation_bytes)?;

    // Manager picks the matching CET, applies adaptor sig, signs, broadcasts
    manager.close_confirmed_contract(&contract_id,
        vec![(0, oracle_attestation)])?;
    Ok(())
}
```

If the oracle never publishes, the refund tx becomes spendable after
`refund_locktime`. `manager.refund(&contract_id)` builds and broadcasts it.

## Common pitfalls

- **Oracle outage** locks funds until refund timeout (typically days).
  Always advertise refund_locktime to users; don't make it shorter than
  your tolerance for oracle downtime.
- **CET storage**: numerical contracts can have thousands of pre-signed
  CETs. The `Storage` trait must persist all of them atomically with the
  funding broadcast -- partial persistence after a crash is unrecoverable.
- **Pre-signed fees**: CETs lock in a fee rate at contract creation. If
  mempool spikes at settlement, your CET may not propagate. Mitigation:
  CPFP via the change output, or use anchor outputs in the funding tx.
- **Adaptor sig validation**: always verify the counterparty's adaptor
  before broadcasting funding -- otherwise you fund a contract you can't
  actually settle.
- **Version pinning**: `dlc`, `dlc-manager`, `dlc-messages` must match
  the same minor; mixed versions silently break wire compatibility.

## References

- Repo: https://github.com/p2pderivatives/rust-dlc
- DLC spec: https://github.com/discreetlogcontracts/dlcspecs
- Protocol overview: https://adiabat.github.io/dlc.pdf
- Companion: [../../cryptography/dlcs/SKILL.md](../../cryptography/dlcs/SKILL.md), [../../cryptography/adaptor-sigs/SKILL.md](../../cryptography/adaptor-sigs/SKILL.md)
