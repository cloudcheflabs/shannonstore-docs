# Network & Internal Communication

ShannonStore nodes communicate via a custom binary NIO RPC protocol.

- A lightweight binary protocol with request-reply correlation enables efficient concurrent operations between nodes.
- Supports transparent network compression for payloads above a configurable threshold.
- TCP socket parameters are tunable for optimal throughput and low latency.
