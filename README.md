# ITJHIT

A number is only as trustworthy as the most serious attempt to break it.

That holds for a matching engine under concurrent load, and it holds for a power plant's debt-service schedule over twenty years. In both, the dangerous output is not the one that looks wrong. It is the one that looks right because nobody tried to break it: the assumption no one traced, the base year no one checked, the repayment structure taken on faith. So the work here is built around adversarial checks first, and output second.

---

## [infra-im-harness](https://github.com/ITJHIT/infra-im-harness): verification gates for infrastructure project finance

Seven gates that attack an investment memo before anyone relies on it. Who actually pays this return, and why? Is every input traced to a primary source and a base year? Was the base case chosen because it looked good? Does an independent re-computation give the same answer? Who carries each risk, and which risks have no owner?

The sample is a hypothetical 1 MW solar plant built only from public data: a national research institute's cost tables, a bank's actual loan terms, and grid-policy announcements. The gates caught real mistakes while it was being built:

- **A stale base year.** The capital cost figure came from the right report, but from its 2023 column. The 2024 column moved equity IRR by four points.
- **A search summary that was wrong.** It dated an event two years late. The primary article settled it.
- **Missing cost lines.** Land lease and corporate tax were absent from the first cash flow.
- **A risk in the wrong year.** The model put the tightest debt-service year near year 15. The actual bank product allows only equal-principal repayment, which moves it to **year 1**.

Every gate documents the failure it exists to catch, and each of those failures actually happened. Spreadsheet models are built by hand. The code only audits them.

---

## Earlier work: the same discipline, applied to systems

### [lowlat-oms-core](https://github.com/ITJHIT/lowlat-oms-core): low-latency OMS core, C++17

[![CI](https://github.com/ITJHIT/lowlat-oms-core/actions/workflows/ci.yml/badge.svg)](https://github.com/ITJHIT/lowlat-oms-core/actions/workflows/ci.yml)

A lock-free ring buffer, a binary wire protocol, a price-time-priority matching engine, a pre-trade risk gate, and two feed handlers (`epoll` and `io_uring`) over an identical pipeline. A concurrent-load benchmark with a provable expected fill count surfaced five genuine shutdown and receive-path bugs in the `io_uring` handler. Each one was root-caused from a failing CI run, not guessed at.

### [l1chain](https://github.com/ITJHIT/l1chain_JIHO): a from-scratch Layer-1 blockchain, Go

[![CI](https://github.com/ITJHIT/l1chain_JIHO/actions/workflows/ci.yml/badge.svg)](https://github.com/ITJHIT/l1chain_JIHO/actions/workflows/ci.yml)

Proof-of-Work consensus, libp2p networking, a stack VM plus an embedded EVM, a Merkle Patricia Trie with independently verified state proofs, and a block explorer. An integration test driven through the real block-validation path caught consensus calling a stale state-transition function, a bug that isolated unit tests could not see.

### [onchain-orderbook](https://github.com/ITJHIT/onchain-orderbook): a deterministic matching engine, Go

[![CI](https://github.com/ITJHIT/onchain-orderbook/actions/workflows/ci.yml/badge.svg)](https://github.com/ITJHIT/onchain-orderbook/actions/workflows/ci.yml)

A matching engine that has to agree byte-for-byte across every node. An adversarial test computes the state root the naive way next to the real one. Over two hundred runs of identical input, the naive version produced eight different answers and the real one produced one.

---

Across all of it, one rule holds. A mistake caught and documented is worth more than a result that was never allowed to fail.
