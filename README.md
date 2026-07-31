# ITJHIT

I build the infrastructure layer of trading and settlement systems — the parts
that have to stay correct under concurrency, partial failure, and adversarial
input, not just fast in the common case. Two repositories anchor this
profile, each built to demonstrate a different half of that discipline:
**low-latency systems programming in C++**, the OMS/exchange infrastructure
real-time trading firms run on, and **deterministic distributed state in
Go**, the same correctness discipline applied to blockchain settlement.

Both are the same bet: understanding markets from the infrastructure layer up
— not just the strategy layer — is what I'm building toward a career in
capital markets. Low-latency trading systems and blockchain are two
different rails money moves on today, and Korean 증권사 are now building the
infrastructure that has to run both at once — regulated security-token
(STO, 토큰증권) trading and clearing, where a matching engine needs
exchange-grade correctness *and* every party has to independently
re-derive the same settlement state. I wanted to build both rails from
scratch, separately, before working on where they intersect.

No alpha, no strategy secrets in either repo — this is the infrastructure
layer, which is exactly the part that generalizes across firms, across
rails, and into where the two are starting to merge.

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

That combination — matching and clearing (청산) coming out of one state
every node independently re-derives, with fairness enforced against the
party running the sequencer itself, not just against other traders — is the
settlement shape a regulated STO exchange has to have. It's the same
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
machine — the guarantee a regulated STO 청산기관 needs when no single
party's ledger is allowed to be the ground truth.

---

All three repos follow the same rule: a bug found by a real CI run and
root-caused is worth more in a portfolio than a benchmark that was never
allowed to fail.
