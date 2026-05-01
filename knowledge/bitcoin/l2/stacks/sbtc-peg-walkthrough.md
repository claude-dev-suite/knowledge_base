# sBTC Peg Walkthrough - Deep Dive

> Phase B article. Companion to dev-suite skill `bitcoin/l2/stacks`.
> Canonical source: https://github.com/stacksgov/sips/blob/main/sips/sip-024/sip-024-bitcoin-finality.md and SIP-021/-026 sBTC
> Skill source: https://github.com/claude-dev-suite/claude-dev-suite/blob/main/skills/bitcoin/l2/stacks/SKILL.md

## Concept

sBTC is a 1:1 BTC-pegged asset on the Stacks chain. Unlike trusted
wrapped-BTC (e.g. WBTC on Ethereum), sBTC custody is held by a rotating
**threshold-signature signer set** elected from Stackers. Pegging in
("deposit") is permissionless; pegging out ("withdrawal") triggers a
threshold-signed Bitcoin tx returning native BTC.

## Walkthrough / mechanics

### Signer set

Each reward cycle elects ~150 signers via Stacking with the largest STX
locks. They run **WSTS (Wrapped Schnorr Threshold Signatures)**, an
FROST-style FROST/Schnorr threshold scheme over secp256k1. The aggregated
signer pubkey `P_signers` is published in a Stacks chainstate variable.

### Peg-in deposit

1. User chooses amount `X BTC`. They construct a Bitcoin tx with:
   - **Output 0**: P2TR or P2WSH spending to `P_signers`.
   - **Output 1**: OP_RETURN with sBTC magic bytes + 32-byte recipient
     STX principal + 8-byte amount.
2. User broadcasts on Bitcoin.
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
     `P_signers` (pubkey rotated each cycle).
4. Threshold-signed tx is broadcast on Bitcoin. After confs, the
   Stacks chain marks the request "fulfilled".

### Signer rotation

At each reward-cycle boundary the signer set changes. Sequence:

1. Newly-elected signers run a DKG (Distributed Key Generation) to
   compute the next `P_signers'`.
2. Outgoing signers sign one final Bitcoin tx that **transfers the
   treasury UTXO** to a new output paying `P_signers'`. This rolls
   custody forward without any user interaction.
3. Failure mode: if old signers refuse to rotate, treasury freezes.
   Recovery requires emergency Stacker intervention via SIP.

### Security model

- 100 of 150 signers (≥ 67 %) must collude to steal funds. No fraud
  proofs; only economic disincentive (their stacked STX gets slashed via
  social slashing — there is no automatic slash contract yet).
- Bitcoin clients verifying sBTC mint/burn events use an SPV-like proof
  embedded in Stacks anchor blocks (Bitcoin block header + Merkle path).

## Worked example

Alice deposits 0.5 BTC.

```
Bitcoin tx:
  Input  : Alice UTXO 0.51 BTC
  Output : P2TR(P_signers) 0.5 BTC
  Output : OP_RETURN <0x73 'sBTC' depositor_principal 50000000>
  Output : Alice change ~0.0099
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

- **OP_RETURN encoding**: format is strict (84 bytes max). Wrong magic
  prefix => Stacks ignores deposit; Alice loses BTC unless she
  reorganises and re-spends from the treasury (impossible).
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
- sbtc-stacks/sbtc reference repo.
