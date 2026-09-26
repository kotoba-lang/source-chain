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

`kotoba/source_chain.kotoba` exports five functions. All take granted
regions (base address and length handed in by the loader) and answer an i64.

| export | answers |
|---|---|
| `verify-chain base len` | number of entries in a valid chain (0 for an empty region), or a negative reason |
| `head-tag base len` | first 15 hex digits of the head entry's hash as a non-negative i64; `-7` for an empty chain; a negative reason when the chain does not verify |
| `fork-evidence a-base a-len b-base b-len` | `1` when two validly signed entries by the same agent share a seq but differ; `0` when they do not; a negative reason when either does not verify |
| `verify-warrant base len` | `1` when a warrant verifies together with the fork it carries as evidence, or a negative reason |
| `replay-balance base len n credit-limit` | an agent's mutual-credit balance **plus 2^50** (so every balance is a non-negative answer), or a negative reason |

Negative reasons:

| code | reason |
|---|---|
| -1 | seq mismatch |
| -2 | prev mismatch (including a genesis entry that names a prev) |
| -3 | invalid signature |
| -4 | malformed message (not printable ASCII, or not the v1 layout) |
| -5 | agent mismatch (key bytes, the message's agent line, and the chain's first agent must agree) |
| -6 | region is not a whole number of entries (or records) |
| -8 | a warrant's evidence is not a fork |
| -9 | a warrant does not name its evidence (accused, seq, or either hash) |
| -10 | the balance fell below `-credit-limit` at some transfer |
| -11 | a transfer is referenced by no entry of the chain |
| -12 | a transfer is referenced by more than one entry (it would count twice) |
| -13 | the chain's agent is neither the spender nor the receiver |
| -14 | transfers are not in the order of the entries that reference them |

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

## Warrant (v1)

A warrant is an issuer's signed accusation, and it carries its own evidence
in one region of 1336 bytes:

```
[issuer key 32][signature 64][warrant message 332][entry A 454][entry B 454]
```

```
kotoba.warrant/v1
issuer:<64 hex: the same 32-byte key>
accused:<64 hex>
rule:fork
seq:<20 decimal digits, first two 0>
a:<64 hex: sha256 of entry A's message>
b:<64 hex: sha256 of entry B's message>
```

A warrant verifies only when all three of these hold:

- the issuer signed it;
- the two entries are a fork (the `fork-evidence` judgement);
- it names exactly that fork: the accused, the seq, and both hashes in order.

So a warrant cannot accuse on hearsay, and a valid signature cannot be
re-pointed at different evidence. Spreading warrants to neighbours is stage
D; this only judges one.

## Mutual-credit transfer (v1)

A transfer is countersigned by both parties, 462 bytes:

```
[spender key 32][receiver key 32][spender sig 64][receiver sig 64][message 270]
```

```
kotoba.engi.transfer/v1
spender:<64 hex>
receiver:<64 hex>
amount:<20 decimal digits, first eight 0, not all 0>
nonce:<64 hex>
```

Each party appends an entry whose `content` is the transfer id (the sha256
of the message) to its own chain. `replay-balance` takes one region: the
agent's chain of `n` entries, followed by the transfers it references, in
chain order. It verifies the chain first, then folds the transfers:

- the spender loses the amount and the receiver gains it;
- the credit limit is checked at every step, not only at the end.

There is no global ledger and nothing is issued, so balances are net zero
across agents; the qualification checks this on two chains. Two limits
apply:

- A double spend is not visible inside one chain. It shows up as a fork,
  which is what `fork-evidence` and `verify-warrant` are for.
- Completeness is the caller's job: the replay counts only the transfers
  it is given.

## Qualification

```bash
kbb --backend sci qualification/native.cljk [--amu ../amu]
```

This compiles the guest for the host's native target, extracts the exports,
builds `amu/tools/kexe_loader.c`, and signs vectors on the host. Signing is
the key holder's job, so the guest only verifies. Every row pins the exact
answer. Two rows pin that the guest traps (SIGTRAP) without the verify or
sha256 grant, rather than answering. A guest with the prev check, the
warrant's evidence-hash check, or the double-count guard removed fails
exactly the row that names it.

Exit codes:

- `0`: every row passed.
- `1`: a row failed.
- `3`: the qualification could not run. This covers a missing toolchain, a
  failed build, and bad arguments. It is never reported as a pass.

Inside the superproject, run it through `scripts/resource-guard.mjs run build
--wait -- …`.

## Not here yet

The following parts of ADR-2609261900 are not done yet:

- DNA validation rules: which content an entry may carry. That decides
  whether a transfer id refers to a real transfer.
- Warrant tally across distinct issuers (K/2), and gossip. These are stages
  C and D.
- Networking, which is stage B.
- Running on x86_64 and Linux static ELF, which has not been measured.
