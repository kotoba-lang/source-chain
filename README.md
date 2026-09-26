# source-chain

A per-agent, signed, append-only **source chain**, verified inside a Kotoba
guest compiled by [amu](https://github.com/kotoba-lang/amu) to native code
and run through amu's kexe loader. It is stage A of root ADR-2609261900
(agent-centric write plane): the write path where an agent writes only its
own chain, and no global consensus, gas or native token is involved.

The retired Rust `kotoba-dht` crate
(`kotoba-server/workspace/crates/kotoba-dht/src/source_chain.rs`) is the
reference for what is checked. It is a specification here, not a wire format
to stay compatible with.

## What the guest answers

`kotoba/source_chain.kotoba` exports three functions. All take granted
regions (base address and length handed in by the loader) and answer an i64.

| export | answers |
|---|---|
| `verify-chain base len` | number of entries in a valid chain (0 for an empty region), or a negative reason |
| `head-tag base len` | first 15 hex digits of the head entry's hash as a non-negative i64; `-7` for an empty chain; a negative reason when the chain does not verify |
| `fork-evidence a-base a-len b-base b-len` | `1` when two validly signed entries by the same agent share a seq but differ; `0` when they do not; a negative reason when either does not verify |

Negative reasons:

| code | reason |
|---|---|
| -1 | seq mismatch |
| -2 | prev mismatch (including a genesis entry that names a prev) |
| -3 | invalid signature |
| -4 | malformed message (not printable ASCII, or not the v1 layout) |
| -5 | agent mismatch (key bytes, the message's agent line, and the chain's first agent must agree) |
| -6 | region is not a whole number of entries |

Checks run in this order: malformed, agent, signature, seq, prev. An
untrusted entry is authenticated before its position is believed.

## Wire format (v1)

Entries are laid out back to back, 454 bytes each:

```
[agent Ed25519 key 32][signature 64][signed message 358]
```

The signed message is fixed-width ASCII, with every line ending in LF:

```
kotoba.source-chain/v1
agent:<64 hex: the same 32-byte key>
seq:<20 decimal digits, first two 0>
prev:<64 hex: sha256 of the previous message>   (64 '-' at seq 0)
ts:<20 decimal digits, first two 0>
policy:<64 hex: sha256 of the data-policy document>
content:<64 hex: sha256 of the content>
```

The entry hash, which the next entry's `prev` names, is the SHA-256 of the
message.

The message is text for a measured reason. On the kexe loader,
`:hash/sha256` digests a string, a string must be valid UTF-8, and
`:identity/verify` reads a granted byte span. An ASCII message is the one
byte sequence both can take unchanged, so the bytes that are verified are
exactly the bytes that are hashed.

Every identifying field is inside the signed message. `policy` is one of
them, so downgrading an entry's data policy without re-signing it is
detected. `seq` and `ts` are fixed width, so they cannot collide at a digit
boundary (the collision the Rust `entry_cid` fixed).

## Qualification

```bash
kbb --backend sci qualification/native.cljk [--amu ../amu]
```

This compiles the guest for the host's native target, extracts the exports,
builds `amu/tools/kexe_loader.c`, and signs vectors on the host. Signing is
the key holder's job, so the guest only verifies. Every row pins the exact
answer. Two rows pin that the guest traps (SIGTRAP) without the verify or
sha256 grant, rather than answering.

Exit codes:

- `0`: every row passed.
- `1`: a row failed.
- `3`: the qualification could not run. This covers a missing toolchain, a
  failed build, and bad arguments. It is never reported as a pass.

Inside the superproject, run it through `scripts/resource-guard.mjs run build
--wait -- …`.

## Not here yet

This covers only the source chain and fork evidence of ADR-2609261900
stage A. The following are not done:

- DNA validation rules.
- The warrant record itself.
- The engi mutual-credit replay.
- Networking, which is stage B.

`fork-evidence` produces the fact a warrant would carry; it does not publish
it.
