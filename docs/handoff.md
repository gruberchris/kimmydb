# Handoff — where development stands

[← Documentation index](README.md)

A running note for picking work back up. Updated at the end of each branch.

---

## As of 2026-08-09 — M6 in progress: webhooks

**M0–M5 are complete.** Their record lives in [Decisions](decisions.md) and
[Deviations](deviations.md); nothing about them is needed to continue here.

**M6 delivers webhooks end to end and is not finished.** This section is written
for someone starting cold.

### State of the branches

| | |
|---|---|
| `main` | PRs #16–#30 merged. Registry, dispatcher, ownership, delivery, signing |
| `m6-webhook-failure-handling` | **Pushed, not merged.** Per-subscription backoff, invalidation past retention, no history replay, webhook metrics |

Start by merging or rebasing that branch — everything below assumes it.
**Gate on it:** 748 tests · fmt · clippy `-D warnings` · `./scripts/check-native-deps.sh`.

### What webhooks are, in one paragraph

A client registers a URL against a collection; the cluster pushes change events
to it. Same events as a [change stream](change-streams.md), opposite direction —
for consumers that cannot hold a WebSocket. [Webhooks](webhooks.md) is the user
documentation and is accurate; read it before the code.

### The design, so it is not re-derived

Read [ADR-045](decisions.md) for the full reasoning. The three load-bearing
ideas:

1. **The dispatcher is an ordinary oplog consumer**, like the embedding worker.
   Nothing about it is special machinery.
2. **Delivery progress is replicated state.** Each node writes *only its own*
   record — `__kimmy.__webhook_progress`, `_id = {subscription}:{node}`, holding
   a `VersionVector` — so there are no write conflicts, and any node reads the
   union. This is what makes a node dying survivable.
3. **Ownership is derived, not elected.** `owner = rendezvous_hash(subscription,
   live SWIM members)`, a pure function every node computes independently. No
   vote, no term, no coordinator. When the owner dies it leaves the live set,
   survivors recompute, and one resumes from the union of progress.

Two earlier designs were tried on paper and rejected, so do not re-propose them:
the *originating* node delivering (loses events when that node dies) and *every*
node delivering (N duplicates per write). Both fix which node delivers in
advance; replicated progress is what removes that constraint.

### Decisions already taken — do not re-litigate

| | |
|---|---|
| Delivery guarantee | **At-least-once.** Exactly-once is not achievable; every event carries a stable `X-Kimmy-Event-Id` so receivers deduplicate |
| Who may register | The **`webhook`** action, independent of `watch`. Only `admin` implies it |
| Egress | Private ranges blocked by default, `webhooks.allowed_hosts` to override. Resolved address checked, every address, redirects refused |
| Where a subscription starts | **Now**, not the beginning of the oplog. Seeded at registration |
| Signing | `HMAC-SHA256(secret, timestamp + "." + body)`, timestamp inside the signature |

### Where the code is

| | |
|---|---|
| `kimmy-api/src/webhooks.rs` | Registry, registration API, the `__kimmy.__webhooks` collection |
| `kimmy-api/src/dispatch.rs` | The worker, progress, delivery, signing, backoff, invalidation |
| `kimmy-api/src/ownership.rs` | Rendezvous hashing over the SWIM member set |
| `kimmy-api/src/egress.rs` | Which URLs may be dialled |
| `kimmy-api/tests/webhooks.rs` | Delivery against a receiver on a real socket |
| `kimmyd/src/node.rs` | Spawns the dispatcher with the live `Members` handle |

### What is left

Small, and none of it blocking. The roadmap's M6 table has the full list; these
are the ones worth doing:

| Item | Note |
|---|---|
| **Backlog age metric** | The one an operator would actually alert on — "how far behind is this subscription". Neither it nor a live-subscriptions gauge is built; task 11 is 🚧 for this reason |
| **Per-node delivery cap** | The only remaining item that is load-bearing: one webhook on a hot collection can currently saturate a node's outbound connections |
| **Payload size cap** | `fullDocument` on large documents, batched, can produce a very large POST. Undecided what happens to a single event that exceeds it |
| Test gaps | Batching behaviour under load, and that deleting a subscription stops it mid-flight. Task 12 is 🚧 |
| MCP tool | Deliberately not built. Registering an egress path is not a reading act; revisit only if asked |

### How this work has been verified

Beyond the suite: a real Python receiver on a socket, verifying the HMAC. That
found the useful things — per-subscription secrets being genuinely distinct
(one subscription's delivery read INVALID at a receiver holding another's
secret), and the history-replay behaviour that became the "start from now" fix.

**Mutation-test anything touching delivery or egress.** Seventeen mutations
have been run across the three M6 branches — six on the registry, six on the
dispatcher, five on failure handling. **One escaped, and it was in the egress
check**: only the first resolved address was being tested, so a host answering
`[public, metadata]` would have been let through. Nothing covered it because a
unit test cannot make DNS return a chosen pair; the fix was to make the rule
testable (`permits_addrs` takes the list) rather than to test around it.

Two habits that came out of this, both worth keeping:

- **Run a no-op control first.** A mutation that fails to *compile* produces no
  `test result:` line and reads exactly like an escape.
- **Check the mutation actually applied** before concluding a test missed it.

### Carried debt, none blocking

One 🔴 in [Deviations](deviations.md): index ranges use only one bound —
correct but less selective; closing it means tracking multikey-ness per index on
the write path. Then `$in` not using an index, descending-field ranges, HNSW
snapshot persistence, SRV discovery (needs a DNS resolver crate), and a vector
reindex operation.

### Invariants a change must not break

- **The transport moves bytes and nothing else.** Convergence is tested without
  a network in `kimmy-storage/src/sync.rs`; keep it that way, or a merge bug and
  a dropped packet become indistinguishable.
- **The version vector is authoritative, not derived.** Never reintroduce a
  rebuild that lowers it — a snapshot grants coverage the oplog never held.
- **Applying replicated DDL must not log**, and must carry the originating stamp
  into any tombstone it records. Both were bugs; both have tests.
- **Retention never collects the newest oplog entry** — the clock resumes from
  it (ADR-028).
- **Anti-entropy excludes `OpKind::UniqueViolation`** — a node's own observation.
- **Collection and index ids are derived from names**, which is what lets a
  replicated entry address the same thing on every node.
- **`kimmy_api::exec` is the single authorization point** for anything a
  principal asked for. Replication goes through `apply_remote`, not `exec`.
- **The login rate limit is consulted before the password is verified**, or it
  stops bounding the Argon2 work that is half its purpose (ADR-038).
- **Both serving paths use `into_make_service_with_connect_info`.** Without it
  there is no peer address, and every caller silently shares one rate-limit
  bucket. There are now two stacks — `axum::serve` for plaintext, `axum-server`
  for TLS — and a new one is how this would regress.
- **Certificates are read before the socket is bound**, so a bad one stops the
  node rather than failing for whoever connects first (ADR-039).
- **Do not add a second native crypto stack.** `ring` is already in the build;
  anything selecting `aws-lc-rs` adds CMake for the same primitives.
- **A type that crosses a format boundary needs a chosen representation, not an
  inherited one.** `NodeId` and `CollectionId` have both cost a replication
  outage by deriving serde and letting BSON decide. Anything new on the wire
  gets the same scrutiny — particularly a `u64`, which BSON cannot hold above
  `i64::MAX`.
- **A fixture that is a hash is a sample, not a constant.** Test both halves of
  the range, and assert the fixture still has the property the test needs.
- **Webhook progress is recorded only after an endpoint accepts.** Recording
  first turns a failed delivery into a silently skipped event.
- **A node writes only its own progress record.** The moment two nodes write
  one record, last-writer-wins starts discarding delivery history.
- **The egress policy is checked before *every* delivery**, not only at
  registration, and every resolved address is checked rather than the first. A
  hostname is not a destination.
- **Webhook ownership hashes with FNV-1a, never `DefaultHasher`**, which is not
  stable between Rust versions — ownership shifting under a compiler upgrade
  would reshuffle every subscription on a rolling restart.

## Conventions for this file

Replace the section above when a branch lands; keep only the current state.
The historical record lives in [Deviations](deviations.md) and
[Decisions](decisions.md), which are append-mostly by design.
