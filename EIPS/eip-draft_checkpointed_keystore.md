---
title: Checkpointed Keystore for EIP-8130
description: Add lazy multichain account configuration using checkpointed authority commitments
author: Pedro Gomes (@pedrouid) <pedro@walletconnect.com>, Chris Hunter (@chunter-cb) <chris.hunter@coinbase.com>, Agustin Aguilar (@Agusx1211) <aaguilar@polygon.technology>
discussions-to: https://ethereum-magicians.org/t/eip-8130-account-abstraction-by-account-configurations/25952
status: Draft
type: Standards Track
category: Core
created: 2026-09-03
requires: 712, 8130
---

## Abstract

This proposal adds an opt-in checkpointed Keystore mode to [EIP-8130](./eip-8130.md). Account actors and policies are committed into an `imageHash` and updated offchain as a strictly ordered configuration state channel. An [EIP-8130](./eip-8130.md) transaction carries the current configuration witness and an optional proof from a configured checkpointer, allowing a chain to validate the latest account authority only when the account transacts. The transaction type, batching, sponsorship, nonce, execution, receipt, and RPC semantics of [EIP-8130](./eip-8130.md) remain unchanged.

The checkpoint and checkpointer construction uses an implementation-neutral configuration model. The checkpointer interface, snapshot type, outer `imageHash` construction, checkpoint ordering, disabled-checkpointer sentinel, and proof carriage remain compatible with existing checkpointed-wallet infrastructure. [EIP-8130](./eip-8130.md) actor scopes, authenticators, policies, account creation, and gas sponsorship remain the authorization model exposed to applications and nodes.

## Motivation

[EIP-8130](./eip-8130.md) stores actor configuration on each chain. A multichain account must therefore submit authority changes to every chain on which current configuration is required. Although multiple changes can be batched, immediately propagating rotations, revocations, session keys, or policy changes across many chains still makes configuration cost proportional to both the number of updates and the number of chains.

Inactive chains create a second problem. Before an account can safely transact on a chain whose Keystore state is behind, every required configuration change must be transported there. A chain introduced after the account was created may need the complete historical change sequence before it can recognize the current authority.

A checkpointed configuration treats account authority as a state channel. Configuration changes are authorized and ordered offchain. The latest state is supplied only when the account sends a transaction, and a new chain can initialize the account from its initial commitment plus one current snapshot. This makes frequent device rotation, session creation, application authorization, and revocation independent of the number of supported chains.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### [EIP-8130](./eip-8130.md) Inheritance

This proposal defines only the checkpointed Keystore mode. All [EIP-8130](./eip-8130.md) behavior not explicitly changed below remains unchanged, including:

- the `AA_TX_TYPE` transaction envelope and signing payload;
- call phases, batching, atomicity, and return-data behavior;
- sender-paid, dedicated payer, and sponsored gas modes;
- keyed and nonce-free transaction nonces;
- validity windows, fee accounting, receipts, and RPC methods;
- EOA transactions, delegation, and default account behavior;
- canonical authenticator selection and bounded authenticator execution; and
- transaction metadata.

Checkpoint proofs are carried inside the existing `sender_auth` and `payer_auth` byte strings. No field is added to the [EIP-8130](./eip-8130.md) transaction envelope.

### Terminology

- **Configuration**: the complete actor and policy state for one account at one checkpoint.
- **Actor root**: the Merkle commitment to every actor configuration.
- **Image hash**: the commitment to the actor root, threshold, checkpoint, and checkpointer.
- **Checkpoint**: a strictly increasing `uint56` configuration sequence number.
- **Settled configuration**: the image hash currently stored in the Keystore for an account.
- **Effective configuration**: the configuration used to validate a transaction.
- **Checkpointer**: a contract that validates a proof and returns the latest recognized snapshot for an account.
- **Snapshot**: an `(imageHash, checkpoint)` pair returned by a checkpointer.

### Checkpointed Account Mode

An account opts into checkpointed mode during creation or import. The mode is permanent. The Keystore stores:

```text
checkpointed_state[account] =
    settled_image_hash
    settled_checkpointer
    CHECKPOINTED flag
```

`settled_checkpointer` is the checkpointer committed by `settled_image_hash`. Storing it explicitly avoids reconstructing the settled actor topology on every transaction and prevents an effective configuration from selecting the contract that approves its own snapshot.

The Keystore MUST NOT store `actor_config`, `policy_manager`, or `policy_commitment` slots for a checkpointed account. Actor and policy data are supplied as witnesses and authenticated against the effective image hash.

[EIP-8130](./eip-8130.md)'s transaction nonces remain in the Nonce Manager. They are execution replay protection and are not part of the checkpointed configuration state channel.

### Actor Configuration Commitment

Checkpointed mode retains the [EIP-8130](./eip-8130.md) actor fields and meanings:

```text
CheckpointedActor {
    actor_id           bytes32
    authenticator      address
    expiry             uint48
    scope              uint16
    policy_manager     address
    policy_commitment  bytes32
}
```

When `scope & POLICY == 0`, `policy_manager` and `policy_commitment` MUST both be zero. When `scope & POLICY != 0`, both values are interpreted exactly as in [EIP-8130](./eip-8130.md).

The actor leaf is:

```text
actor_leaf = keccak256(
    "EIP-8130 checkpointed actor:\n" ||
    actor_id ||
    authenticator ||
    expiry ||
    scope ||
    policy_manager ||
    policy_commitment
)
```

The byte widths are exactly those shown above and no ABI padding is inserted. An actor topology is constructed from actor leaves and opaque node hashes. Two child nodes are combined as:

```text
node = keccak256(left || right)
```

The topology and proof encoding MUST use the checkpointed branch, node, and dynamic-size conventions defined by this specification. An implementation MAY replace any branch not required for the acting actor with its 32-byte node hash.

This proposal assigns topology flag `0x0b` to an EIP-8130 actor. Its encoded form is:

```text
0xb0 ||
actor_id (32 bytes) ||
authenticator (20 bytes) ||
expiry (6 bytes) ||
scope (2 bytes) ||
policy_manager (20 bytes) ||
policy_commitment (32 bytes) ||
actor_auth_length (3 bytes) ||
actor_auth (variable)
```

The leaf contributes weight `1` only when `actor_auth` authenticates the enclosing payload and returns the encoded `actor_id`. Otherwise it contributes weight `0`. The signature bytes do not participate in `actor_leaf`.

The actor topology root is `actors_root`.

### Image Hash

Checkpointed [EIP-8130](./eip-8130.md) configurations use the following outer configuration commitment with a fixed threshold of `1`:

```text
image_hash = keccak256(
    keccak256(
        keccak256(
            actors_root || bytes32(uint256(1))
        ) || bytes32(uint256(checkpoint))
    ) || bytes32(uint256(uint160(checkpointer)))
)
```

`checkpoint` MUST fit in `uint56`. `checkpointer` MAY be `address(0)`, which disables snapshot validation for that configuration.

The fixed threshold preserves [EIP-8130](./eip-8130.md)'s rule that one live actor authorizes one operation. Multisignature or weighted authentication remains expressible through an [EIP-8130](./eip-8130.md) authenticator and does not change [EIP-8130](./eip-8130.md) scope semantics.

### Checkpointer Interface

The checkpointer ABI is:

```solidity
struct Snapshot {
    bytes32 imageHash;
    uint256 checkpoint;
}

interface ICheckpointer {
    function snapshotFor(
        address wallet,
        bytes calldata proof
    ) external view returns (Snapshot memory snapshot);
}
```

This ABI is compatible with existing checkpointed-wallet checkpointer implementations.

The `wallet` argument MUST be the [EIP-8130](./eip-8130.md) account being authenticated. The `proof` format is defined by the checkpointer implementation and is opaque to the Keystore. A checkpointer MAY validate a trusted signature, threshold signature, Merkle proof, trusted execution environment attestation, validity proof, or rollup state proof.

The returned snapshot MUST satisfy:

```text
snapshot.checkpoint <= uint56.max
```

A snapshot with `snapshot.imageHash == bytes32(0)` disables the checkpointer for that validation and activates the chained fallback path described below.

### Snapshot Semantics

The checkpointer used for validation MUST be `settled_checkpointer` from Keystore storage. A checkpointer address supplied only by a newer, unsettled configuration MUST NOT be invoked. This prevents a proposed state from selecting a checkpointer that authorizes itself.

When `settled_checkpointer` is non-zero, checkpointed authorization includes checkpointer data and invokes:

```text
snapshot = ICheckpointer(settled_checkpointer).snapshotFor(account, proof)
```

The call MUST use `STATICCALL`. Its calldata, execution, return-data copying, and decoding costs MUST be charged within the authentication-gas bound.

If `snapshot.imageHash != bytes32(0)`, the effective configuration is valid when either:

1. its `image_hash` equals `snapshot.imageHash` and its checkpoint equals `snapshot.checkpoint`; or
2. it is connected to `snapshot.imageHash` by a valid chained configuration proof and its checkpoint is greater than `snapshot.checkpoint`.

An effective configuration whose checkpoint is less than the snapshot checkpoint is invalid. A configuration at the same checkpoint with a different image hash is invalid.

The checkpointer is responsible for defining what makes a snapshot current and authorized. A signature-only checkpointer is a trusted authority for snapshot correctness. A proof-verifying checkpointer MAY provide the same interface without that trust assumption.

Unlike validation designs that use a snapshot only as a freshness constraint while retaining the onchain image hash as the final authority anchor, checkpointed Keystore mode accepts the returned snapshot as an authority anchor. This deliberate semantic change is what permits a chain at checkpoint `0` to use checkpoint `100` without replaying the intervening configuration chain. The ABI, snapshot value, proof bytes, and configuration encoding remain compatible.

### Configuration Updates

A configuration update authorizes one successor image hash using the following [EIP-712](./eip-712.md) payload:

```text
ConfigUpdate(bytes32 imageHash,address[] wallets)
```

For a non-nested [EIP-8130](./eip-8130.md) account, `imageHash` is the successor image hash and `wallets` is the empty array. The struct hash is:

```text
keccak256(abi.encode(
    keccak256("ConfigUpdate(bytes32 imageHash,address[] wallets)"),
    next_image_hash,
    keccak256(abi.encodePacked(wallets))
))
```

The domain separator is:

```text
keccak256(abi.encode(
    keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
    keccak256("Checkpointed Keystore"),
    keccak256("1"),
    uint256(0),
    account
))
```

The update digest is `keccak256("\x19\x01" || domain_separator || struct_hash)`. The account is therefore bound as `verifyingContract`, and `chainId = 0` makes the signature chain-independent. Implementations supporting nested wallets MAY populate `wallets`; otherwise they MUST require it to be empty.

The update is authenticated by an admin actor from the current configuration. The successor checkpoint MUST be strictly greater than the current checkpoint. The signature and actor witness are evaluated using the current configuration, not the successor configuration.

This encoding allows the same configuration update signature and state channel to be used across every [EIP-8130](./eip-8130.md) chain.

### Chained Configuration Proofs

A chained proof represents configurations `C[0] -> C[1] -> ... -> C[n]`, where `C[0]` is the snapshot or settled configuration and `C[n]` is the effective configuration. For every forward link `C[i] -> C[i+1]`:

1. the proof MUST reveal the `C[i]` header and the admin actor witness needed to reconstruct its image hash;
2. a live admin actor from `C[i]` MUST authenticate the image hash of `C[i+1]`;
3. the checkpoint of `C[i+1]` MUST be strictly greater than the checkpoint of `C[i]`; and
4. bind the same account.

The wire encoding is reversed: transaction authorization by `C[n]` appears first, followed by update signatures from `C[n-1]` through `C[0]`. The proof is valid only if its oldest recovered image hash equals either the active snapshot image hash or the settled image hash.

Nested chained proofs are invalid. Each link authorizes exactly one successor configuration. Chained proof elements and their three-byte length prefixes MUST use the chained-signature ordering and length encoding defined below.

### Checkpointed Authorization Encoding

For a checkpointed account, configured-actor authorization retains the [EIP-8130](./eip-8130.md) prefix:

```text
authenticator (20 bytes) || checkpointed_authorization
```

For a regular authorization, `checkpointed_authorization` contains:

```text
flags                         1 byte
checkpointer                  20 bytes, when flags & 0x40 != 0
checkpointer_data_length      3 bytes, when flags & 0x40 != 0
checkpointer_data             variable
checkpoint                    0 to 7 bytes, size in flags bits 4..2
threshold                     1 byte, MUST equal 1
actor_topology                checkpointed topology encoding containing one 0x0b actor leaf
```

When `flags & 0x01 != 0`, the bytes following the optional checkpointer data are instead a chained-signature list. Each element has a three-byte length prefix and is ordered from the transaction authorization back toward the settled configuration.

The flag assignments, variable-width checkpoint encoding, three-byte dynamic lengths, checkpointer placement, and reverse chained ordering define the checkpointed authorization encoding. Bit `0x02` MUST be set for configuration-update links because those links are chain-independent. Transaction signatures remain bound to the [EIP-8130](./eip-8130.md) transaction `chain_id` through the unchanged [EIP-8130](./eip-8130.md) sender or payer signature hash.

The actor witness MUST reconstruct the `actors_root` and reveal exactly one `CheckpointedActor` matching the authenticator prefix. The authenticator is then invoked exactly as in [EIP-8130](./eip-8130.md) over the unchanged sender or payer signature hash. The returned `actorId` MUST equal the witnessed `actor_id`.

### Transaction Validation

For a checkpointed sender, [EIP-8130](./eip-8130.md) validation replaces the `actor_config` storage read with:

1. Read the account's settled image hash, settled checkpointer, and `CHECKPOINTED` flag.
2. Decode the effective configuration header and reconstruct its image hash.
3. Recover the settled checkpointer and, when enabled, validate the supplied snapshot proof.
4. Validate the effective image hash against the snapshot or a chained configuration proof.
5. Reconstruct the actor root from the acting actor and its witness.
6. Invoke the declared [EIP-8130](./eip-8130.md) authenticator and require the returned actor ID to match the witnessed actor.
7. Apply [EIP-8130](./eip-8130.md) actor expiry and scope rules to the witnessed configuration.
8. Continue with the unchanged [EIP-8130](./eip-8130.md) nonce, payer, balance, validity-window, and mempool checks.

The same flow applies independently to a checkpointed `payer_auth`. The sender and payer MAY use different checkpointers and snapshots.

Nodes MUST bound checkpoint, witness, and chained-proof processing through the intrinsic-gas schedule. A chain MUST NOT accept proof work whose declared or measured cost exceeds its configured [EIP-8130](./eip-8130.md) authentication bound.

### Policy Enforcement

For an actor whose scope includes `POLICY`, `policy_manager` and `policy_commitment` are read from the authenticated actor witness rather than Keystore storage.

The [EIP-8130](./eip-8130.md) call gate is otherwise unchanged: every `call.to` MUST equal the snapshotted `policy_manager`. The manager interprets `policy_commitment` and enforces the application-specific policy. The actor witness used during validation MUST remain the policy source for all phases; a later settlement or configuration change in the same transaction MUST NOT retarget the gate.

### Account Creation

A checkpointed create entry replaces `initial_actors` with:

```text
checkpointed_create = rlp([
    0x00,
    user_salt,
    code,
    initial_configuration,
    initial_image_hash
])
```

`initial_configuration` MUST have checkpoint `0`, MUST reconstruct `initial_image_hash`, and MUST contain at least one admin actor. Every initial actor MUST have `expiry == 0`.

Address derivation uses `initial_image_hash` in place of [EIP-8130](./eip-8130.md)'s initial-actor commitment:

```text
effective_salt = keccak256(user_salt || initial_image_hash)
```

All remaining [EIP-8130](./eip-8130.md) CREATE2 freshness, bytecode, delegation, and initial-call rules remain unchanged. Later configuration roots do not change the account address.

### Account Import

A checkpointed import registers an existing smart account with an initial image hash rather than materializing initial actors. The import signature MUST bind the account, `chainId`, and initial image hash. The existing [EIP-8130](./eip-8130.md) `chainId == 0 || chainId == block.chainid` rule remains unchanged.

The Keystore adds:

```solidity
function importCheckpointedAccount(
    address account,
    uint256 chainId,
    bytes32 initialImageHash,
    address initialCheckpointer,
    bytes calldata initialConfiguration,
    bytes calldata signature
) external;
```

`initialConfiguration` MUST reconstruct `initialImageHash` and commit to `initialCheckpointer`.

On successful creation or import, the Keystore stores the initial image hash and initial checkpointer, sets `CHECKPOINTED` and `DEFAULT_EOA_REVOKED`, initializes the existing EIP-8130 change state, and stores no actor or policy slots. Continuing to use the address-bound secp256k1 key requires including it as an explicit checkpointed actor. It MUST NOT remain available through the implicit EOA path because that path would bypass checkpointed revocation.

### Settlement

Checkpointed mode assigns one [EIP-8130](./eip-8130.md) account change:

| Type | Name | Payload |
|---|---|---|
| `5` | `SettleImageHash` | `abi.encode(bytes32 imageHash, bytes32 actorsRoot, uint56 checkpoint, address checkpointer)` |

`SettleImageHash` is valid only when `imageHash` is the effective configuration already authenticated for the enclosing transaction or signed Keystore call. The supplied header MUST reconstruct `imageHash`. It updates `settled_image_hash` and `settled_checkpointer` atomically and emits:

```solidity
event ImageHashUpdated(address indexed account, bytes32 imageHash, uint56 checkpoint);
```

Settlement is optional. A transaction MAY authenticate and execute against a snapshot without updating the settled image hash. Wallets SHOULD settle periodically to bound fallback proof length and reduce dependence on the checkpointer.

The original [EIP-8130](./eip-8130.md) `AuthorizeActor`, `RevokeActor`, and `IncrementLocalEpoch` changes are invalid for checkpointed accounts. Their equivalent actor and policy transitions occur in the offchain configuration state channel. They remain unchanged for non-checkpointed accounts.

[EIP-8130](./eip-8130.md) lock and unlock changes remain chain-local onchain state and retain their existing semantics. They MUST NOT be represented only inside the multichain image hash. Applying `Lock` to a checkpointed account MUST first settle the effective image hash. While the lock is effective, validation MUST require `effective_image_hash == settled_image_hash`; snapshots and chained proofs cannot advance authority until the existing EIP-8130 unlock delay has completed.

### EVM Validation Interface

The Keystore exposes checkpointed authentication through ordinary EVM calls so the same account can use another transaction transport on a chain without the [EIP-8130](./eip-8130.md) transaction type:

```solidity
interface ICheckpointedKeystore {
    function validateCheckpointedSignature(
        address account,
        bytes32 hash,
        bytes calldata auth
    ) external view returns (bytes32 actorId, uint16 scope);

    function settleImageHash(
        address account,
        bytes calldata authorization
    ) external;
}
```

Contract execution and native [EIP-8130](./eip-8130.md) validation MUST produce identical image-hash, snapshot, witness, actor, expiry, and scope results.

### Gas Accounting

The [EIP-8130](./eip-8130.md) intrinsic-gas formula remains structurally unchanged. For checkpointed authentication:

- the `actor_config` SLOAD component is removed;
- calldata cost includes the checkpoint, snapshot, actor witness, and chained proof;
- authentication cost includes topology hashing, checkpointer proof verification, chained-update authentication, and actor authentication; and
- `SettleImageHash` charges the corresponding Keystore storage write.

Canonical proof operations MUST have protocol-defined constant costs under the [EIP-8130](./eip-8130.md) L1 profile. The L2 profile MAY use chain-specific costs fixed under that chain's consensus rules.

### Constants

| Name | Value | Meaning |
|---|---|---|
| `CHECKPOINTED` | `0x08` | Account uses checkpointed Keystore authority |
| `CHECKPOINTED_THRESHOLD` | `1` | Fixed outer configuration threshold |
| `CHECKPOINTER_FLAG` | `0x40` | Authorization includes checkpointer data |
| `CHAINED_FLAG` | `0x01` | Authorization includes chained configuration updates |
| `NO_CHAIN_ID_FLAG` | `0x02` | Configuration-update signature uses `chainId = 0` |
| `DISABLED_SNAPSHOT` | `bytes32(0)` | Checkpointer is disabled for this validation |
| `MAX_CHECKPOINT` | `2^56 - 1` | Largest encodable checkpoint |

## Rationale

### Why Extend [EIP-8130](./eip-8130.md)?

[EIP-8130](./eip-8130.md) already defines predictable authenticator selection, native transaction validation, sponsorship, batching, parallel nonces, policies, and portable EVM fallbacks. Replacing those surfaces would create a second account-abstraction system. This proposal changes only where actor authority is stored and how the effective state is proven.

### Why Use This Encoding?

Image hashes, monotonically ordered checkpoints, chained configuration signatures, opaque checkpointer proofs, and the `snapshotFor` interface are already implemented by checkpointed-wallet infrastructure. Matching those primitives permits existing configuration hashing, signature encoding, checkpointer, and proof infrastructure to be reused. [EIP-8130](./eip-8130.md)-specific actor leaves preserve its authenticator and scope model while remaining opaque nodes to generic topology tooling.

### Why Is Actor Expiry 48 Bits?

The `uint48` actor expiry is inherited unchanged from [EIP-8130](./eip-8130.md), where it is a Unix timestamp in seconds and occupies six bytes in the packed actor configuration. Checkpointed mode retains the width so actor witnesses use the same field semantics and representation even though they are no longer stored in Keystore actor slots.

### Why Is the Checkpoint Limited to 56 Bits?

The authorization flag reserves three bits for the byte length of the checkpoint. It can therefore encode between zero and seven bytes, making `2^56 - 1` the largest representable checkpoint. The checkpointer ABI exposes the value as `uint256`, but values above the 56-bit wire limit are invalid.

### Why Keep a Settled Image Hash?

The settled image hash is a trustless anchor and an escape path. It lets an account continue through owner-authorized chained updates when the checkpointer is unavailable or disabled. Optional settlement also bounds future proof size without requiring a standalone configuration transaction.

### Why Can a Snapshot Anchor Authority?

Treating the snapshot only as a freshness floor would still require every lagging chain to receive the signed configuration chain back to its settled image hash. Accepting the snapshot itself as an authority anchor enables lazy one-proof synchronization and constant-history initialization on newly supported chains. This gives the configured checkpointer authority over the accepted configuration, so the trust and escape-hatch requirements are explicit and the mode is opt-in.

### Why Keep Lock State Onchain?

Lock and unlock depend on local block time and intentionally restrict configuration changes on one chain. Moving them into a chain-independent offchain root would make their timing ambiguous across chains and would give the checkpointer control over the escape mechanism.

## Backwards Compatibility

Checkpointed mode is opt-in. Existing [EIP-8130](./eip-8130.md) accounts and transactions continue to use materialized actor and policy storage without modification.

The [EIP-8130](./eip-8130.md) transaction envelope is byte-for-byte unchanged. Nodes distinguish checkpointed authentication from materialized authentication through the account's Keystore mode. Wallets, relays, and payers that do not support checkpointed proofs can continue using ordinary [EIP-8130](./eip-8130.md) accounts.

Existing checkpointer contracts can be used without ABI changes. Existing configuration and signature tooling can reproduce the outer image hash, checkpoint and checkpointer headers, node and branch proofs, and chained-signature framing. It requires support for this specification's neutral [EIP-712](./eip-712.md) domain, the `0x0b` actor leaf, and snapshots being accepted as authority anchors rather than only freshness constraints.

## Security Considerations

**Checkpointer authority**: A checkpointer that can attest arbitrary snapshots can select an image hash containing attacker-controlled actors. Accounts using a signature-only checkpointer therefore trust it as configuration authority. Trust-minimized deployments SHOULD use a proof-verifying checkpointer that establishes owner-authorized transitions.

**Stale snapshots**: A stateless signature verifier cannot determine that a previously valid signed snapshot has become stale. Such a checkpointer MUST provide freshness through expiring proofs, a committed latest-state root, a rollup proof, or another mechanism that prevents acceptance of superseded snapshots.

**Checkpointer self-selection**: Validation MUST use the checkpointer committed by the settled configuration. Using a checkpointer supplied only by the proposed configuration lets that configuration authorize itself.

**Checkpointer liveness**: A mandatory unavailable checkpointer can prevent transactions. The disabled snapshot sentinel and chained fallback provide an escape path, but wallets must retain the signed update chain and configuration witnesses needed to use it.

**Data availability**: Actor and policy state is no longer reconstructible from chain storage or events alone. Wallets MUST retain configuration data and signed transitions in redundant storage. Losing this data can make an otherwise valid account unusable.

**Checkpoint monotonicity**: Every successor checkpoint must be strictly greater. Reusing a checkpoint for a different image hash creates an equivocation that validators MUST reject when both states are visible.

**Proof bounds**: Unbounded Merkle or chained proofs create a denial-of-service surface during transaction validation. Implementations MUST charge all proof bytes and hashing work and enforce the [EIP-8130](./eip-8130.md) authentication-gas bound.

**Mempool invalidation**: A pending transaction can be invalidated by a newer checkpointer snapshot even when its nonce is unchanged. Nodes accepting checkpointed transactions MUST track the settled checkpointer and revalidate affected transactions when the checkpointer's latest-state commitment changes.

**Policy witness stability**: The policy manager and commitment used for execution must come from the actor witness authenticated during validation. They must not be re-resolved after calls or settlement because doing so could change the gate within one transaction.

**Settlement rollback**: Settling an older image hash can resurrect revoked actors. `SettleImageHash` MUST accept only the effective image hash already authenticated for the same operation and MUST reject a checkpoint below the currently settled checkpoint.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
