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
capital markets. Low-latency trading systems and blockchain are two different
rails money moves on today; I wanted to build both from scratch before
trying to work in either.

No alpha, no strategy secrets in either repo — this is the infrastructure
layer, which is exactly the part that generalizes across firms and across
rails.

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
auction, same input). The order book itself lives in a companion repository,
[onchain-orderbook](https://github.com/ITJHIT/onchain-orderbook), built around
a determinism budget (no floats, no map iteration on any output path, no wall
clock) and proven, not asserted: an adversarial test computes the state root
the naive way alongside the real one, and shows the naive version producing
eight different answers over two hundred runs of *identical* input.

**What integrating the two actually found:** the chain's consensus path
(mining and block validation) was calling a stale version of the state
transition function, one that didn't know about order identity — caught by a
regression test that drove the real `Chain.AddBlock` path instead of testing
the exchange package in isolation, exactly the kind of bug that isolated
unit tests are structurally unable to see.

---

Both repos follow the same rule: a bug found by a real CI run and root-caused
is worth more in a portfolio than a benchmark that was never allowed to fail.
