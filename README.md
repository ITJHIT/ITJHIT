# ITJHIT

I build the infrastructure layer of trading and settlement systems — the parts
that have to stay correct under concurrency, partial failure, and adversarial
input, not just fast in the common case. Two repositories anchor this
profile, each built to demonstrate a different half of that discipline:
**low-latency systems programming in C++**, the OMS/exchange infrastructure
real-time trading firms run on, and **deterministic distributed state in
Go**, the same correctness discipline applied to blockchain settlement.

Both are the same conviction, tested under two different constraints: a
market is only as trustworthy as the infrastructure underneath it, and that
infrastructure has to be provably correct before it's allowed to be fast. I
didn't want to take either half on faith — not the low-latency systems
real trading firms already run on, and not the blockchain settlement
everyone assumes is trustless by default — so I built both from the ground
up and made each one fail in a real CI run before calling it done.

No alpha, no strategy secrets in either repo — this is the infrastructure
layer, the part that holds regardless of which rail the money is moving on,
or which firm is running the exchange.

---

## [lowlat-oms-core](https://github.com/ITJHIT/lowlat-oms-core) — low-latency OMS core, C++17

[![CI](https://github.com/ITJHIT/lowlat-oms-core/actions/workflows/ci.yml/badge.svg)](https://github.com/ITJHIT/lowlat-oms-core/actions/workflows/ci.yml)

The building blocks a real-time OMS is made of: a lock-free SPSC ring buffer
(in-process **and** across a process boundary over shared memory), an
exchange-style binary wire protocol with incremental decode and gap
detection, a price-time-priority matching engine, a pre-trade risk gate, and
two feed handlers — `epoll` and `io_uring` — over the identical pipeline, so
the network I/O model is the only variable between them.

**What building both feed handlers actually found:** a concurrent-load
benchmark (50 connections, tens of thousands of frames, exact expected fill
count provable from TCP's own ordering guarantee) run against real CI
surfaced five independent, genuine bugs in the io_uring handler's shutdown
and recv path — an unread-kernel-bytes race, a consumer-exit race, an
accept-backlog gap, a transient `EAGAIN` completion misread as a dead
connection, and a syscall-bound reaping pattern that fell behind under burst
load. All five were root-caused from failing CI runs, not guessed at. The
README documents the fix for each one, and is equally honest about what
*didn't* work: two different attempts to make the regression check airtight
against noise on shared CI runners, and why neither one held up when
re-checked against the actual data.

## [l1chain](https://github.com/ITJHIT/l1chain_JIHO) — a from-scratch Layer-1 blockchain, Go

[![CI](https://github.com/ITJHIT/l1chain_JIHO/actions/workflows/ci.yml/badge.svg)](https://github.com/ITJHIT/l1chain_JIHO/actions/workflows/ci.yml)

Proof-of-Work consensus, an account-based native coin, real libp2p
peer-to-peer networking, a custom stack VM plus an embedded EVM running
standard ERC-20 contracts, a JSON-RPC node, and a Next.js block explorer with
browser-side transaction signing that a Go node accepts verbatim. Every
component is built from scratch, not forked.

Layered on top: an **on-chain limit order book** — order-as-transaction,
book state folded automatically into the chain's state root, and a batch-auction
matching mode as a structural defense against block-producer MEV (demonstrated
head-to-head against continuous matching: 10/0 fills favoring the
front-runner under continuous matching, 5/5 regardless of order under batch
auction, same input).

That combination — matching and clearing coming out of one state every node
independently re-derives, with fairness enforced against the party running
the sequencer itself, not just against other traders — is the settlement
shape a regulated market for security tokens has to have. It's the same
problem `lowlat-oms-core` solves for matching speed, applied instead to
matching that has to be provably fair and byte-identical across every
validator.

**What integrating the two actually found:** the chain's consensus path
(mining and block validation) was calling a stale version of the state
transition function, one that didn't know about order identity — caught by a
regression test that drove the real `Chain.AddBlock` path instead of testing
the exchange package in isolation, exactly the kind of bug that isolated
unit tests are structurally unable to see.

The state root itself was a flat hash fold at first — a placeholder, not a
real trie. Replaced it with a from-scratch SHA-256 Merkle Patricia Trie
(two-level: world trie + per-account storage tries), then built the thing
that trie exists to enable: account/storage Merkle proofs over RPC, verified
independently by the CLI against a PoW-checked header rather than just
trusted from the node. Proved the network layer the same way — a
`ConnectionManager` and an inbound-stream cap now bound eclipse-style sybil
connections and sync-flood DoS, with adversarial tests confirmed to fail
without either defense before they were allowed to pass with it. Real 3-node
convergence demonstrated over actual Docker containers discovering each
other via libp2p, not an in-process harness handing them each other's
addresses.

### [onchain-orderbook](https://github.com/ITJHIT/onchain-orderbook) — the matching engine underneath, Go

[![CI](https://github.com/ITJHIT/onchain-orderbook/actions/workflows/ci.yml/badge.svg)](https://github.com/ITJHIT/onchain-orderbook/actions/workflows/ci.yml)

The engine l1chain's on-chain exchange is built on, standalone: a
determinism budget (no floats, no map iteration on any output path, no wall
clock) and proof instead of assertion — an adversarial test computes the
state root the naive way (walking a map) alongside the real one, and shows
the naive version producing eight different answers over two hundred runs of
*identical* input, the real one producing one. Same price-time-priority
matching discipline as `lowlat-oms-core`, opposite constraint: this one has
to agree byte-for-byte across every validator, not just be fast on one
machine — the guarantee a regulated clearing venue needs when no single
party's ledger is allowed to be the ground truth.

---

All three repos follow the same rule: a bug found by a real CI run and
root-caused is worth more in a portfolio than a benchmark that was never
allowed to fail.
