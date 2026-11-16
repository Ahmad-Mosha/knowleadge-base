# 01 — Computer Foundations

[Home](../README.md) › Computer Foundations

Everything a backend does ends as bits in memory and instructions on a CPU. This short section covers the two facts about the machine that explain the most behavior higher up: **how data is represented**, and **how different the speeds of hardware components are**.

## Pages

| # | Page | What you will understand |
|---|---|---|
| 1 | [Data Representation](01-data-representation.md) | Bits, integers, floating point, text encodings, and why `0.1 + 0.2 !== 0.3` |
| 2 | [Hardware and Latency](02-hardware-and-latency.md) | CPU, caches, RAM, disks, and the latency numbers that drive every design decision |

## Why this comes first

- JavaScript's `number` type, `Buffer`, database column types, and wire protocols are all direct uses of [data representation](01-data-representation.md).
- Caching, indexes, batching, and async I/O all exist because of the [latency gaps](02-hardware-and-latency.md) between CPU, memory, disk and network.

## Where this leads

- [02 Operating Systems](../02-operating-systems/README.md) shows how the OS shares this hardware between programs.
- [04 JavaScript: Types and Coercion](../04-javascript/01-types-and-coercion.md) builds on number and string representation.
- [06 Node.js: Buffers and Streams](../06-nodejs/03-buffers-and-streams.md) works with raw bytes directly.

## Primary sources

- Bryant and O'Hallaron, *Computer Systems: A Programmer's Perspective* (chapters 2 and 6).
- IEEE 754 standard for floating-point arithmetic.
- The Unicode Standard and RFC 3629 (UTF-8).

---

← [The Big Picture](../00-start-here/big-picture.md) | Next: [Data Representation →](01-data-representation.md)
