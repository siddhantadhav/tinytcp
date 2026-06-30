# tinytcp
Building a TCP/IP stack from scratch in Go

# TCP/IP Stack in Go — Learning Plan

Goal: user-space TCP/IP stack from raw Ethernet up to TCP. Ping, UDP echo, real TCP transfer.
Timeline: 6–12 weeks, few hrs/week. TCP (phase 6) ~half the time.
Rule: don't move to next phase until current one actually works against real traffic, not just self-tests.

Repo: `tinytcp` (or similar). MIT license.

**Structure plan:** CLI first, package later — no public API design until the stack actually works.

```
tinytcp/
  cmd/tinytcp/main.go      # CLI entrypoint
  internal/tap/            # phase 1
  internal/ethernet/       # phase 2
  internal/arp/            # phase 3
  internal/ipv4/           # phase 4
  internal/icmp/           # phase 4
  internal/udp/            # phase 5
  internal/tcp/            # phase 6
```

- Phases 1–6: everything lives under `internal/`, not importable outside the module. `main.go` wires it together, exposes subcommands (`tinytcp ping`, `tinytcp tcpserver`, etc.) — same shape as `gotcp`.
- After phase 6 exit checks pass: refactor. Move stable types/functions from `internal/` to a root or `pkg/` package with an exported API (`Stack`, `Conn`, `Dial`, `Listen`...). CLI in `cmd/tinytcp` becomes a thin consumer of that package instead of owning the logic directly. Tag a release (`v0.1.0`) once it's `go get`-able.
- Don't do this refactor early — exported APIs are a commitment, and the design isn't known until TCP actually works end to end.

---

## Phase 0 — Read packets manually

- Capture traffic (Wireshark / `tcpdump -XX`)
- Ping something, decode one ICMP packet byte-by-byte by hand (Ethernet → IP → ICMP)
- Capture a TCP handshake (`curl`), find SYN / SYN-ACK / ACK, decode seq/ack/flags manually

**Resources:** Beej's Guide to Network Programming (concepts only, code is C) · RFC 791 (IPv4) · RFC 792 (ICMP) · RFC 793 / 9293 (TCP, keep for phase 6)

**Exit check:** can ID MAC, EtherType, IP version/IHL/TTL/protocol, TCP flags+seq from a raw hex dump, no reference.

---

## Phase 1 — TAP device

- Create TAP interface (`/dev/net/tun`, `IFF_TAP`) — TAP not TUN, need L2 frames
- Go program reads raw bytes off TAP, writes back, confirm via `tcpdump -i <iface>`
- Needs root. Linux only — use a VM/Docker (`--privileged --cap-add=NET_ADMIN`) if on macOS

**Resources:** `songgao/water` (Go TAP/TUN wrapper, fine to use as-is) · `terassyi/gotcp` (reference impl, read device setup) · `tuntap(4)` man page

**Exit check:** frames read as `[]byte`, written frames show up in `tcpdump`.

---

## Phase 2 — Ethernet parsing

- Struct: dst MAC (6B), src MAC (6B), EtherType (2B)
- Parse with `encoding/binary` (big-endian)
- Dispatch: `0x0806`→ARP, `0x0800`→IPv4, else drop
- Write matching construct/serialize function

**Resources:** Ethernet frame format (Wikipedia is accurate here) · `encoding/binary` docs

**Exit check:** EtherType printed for every frame matches Wireshark.

---

## Phase 3 — ARP

- Parse/construct ARP (hw type, proto type, opcode, sender/target MAC+IP)
- ARP cache: `map[string]net.HardwareAddr`, mutex-protected
- Respond to requests for own IP
- Send requests to resolve unknown IPs (needed for phase 4)

**Resources:** RFC 826 (short, read full) · `gotcp` ARP code

**Exit check:** stack shows in `arp -a` on another box; resolves a real gateway MAC.

---

## Phase 4 — IPv4 + ICMP

- Parse IPv4 header: version, IHL, total length, TTL, protocol, checksum, addrs
- Checksum (one's complement sum) — write a unit test against a known-good capture, this is the #1 silent-bug spot
- Route to own IP only, dispatch by protocol field (no real forwarding table needed)
- ICMP echo request/reply

**Resources:** RFC 791, RFC 792 (read properly now) · `gotcp` ping/IP layer · `google/netstack` IP layer (gvisor.dev/gvisor/pkg/tcpip, read after own version works)

**Exit check:** another machine can `ping` the stack and get correct-latency replies; `tcpdump` shows valid checksums.

---

## Phase 5 — UDP

- Parse/construct: src port, dst port, length, checksum
- Checksum uses pseudo-header (IP fields + UDP segment) — not identical to IP checksum
- UDP echo server

**Resources:** RFC 768 (short, read full)

**Exit check:** `nc -u` against the stack echoes correctly.

---

## Phase 6 — TCP

### 6a. State machine
- Go type for states: LISTEN, SYN_SENT, SYN_RECEIVED, ESTABLISHED, FIN_WAIT_1, FIN_WAIT_2, CLOSE_WAIT, CLOSING, LAST_ACK, TIME_WAIT, CLOSED
- Write transition table on paper first

### 6b. Handshake
- LISTEN → recv SYN → send SYN-ACK → recv ACK → ESTABLISHED
- Exit check: `nc <ip> <port>` connects (no data yet)

### 6c. Data transfer
- Seq/ack tracking
- Segmentation out, reassembly (incl. out-of-order) in
- Send/recv buffers, unacked-data window

### 6d. Retransmission
- RTO timer, resend on timeout
- Fixed or basic RTT-estimated timeout fine — skip Karn's algorithm for now

### 6e. Flow control + teardown
- Receive window advertisement
- Four-way close (FIN/ACK both directions), TIME_WAIT

### Stopping point
No congestion control (slow start / Reno / Cubic) unless extending later. Reliable flow-controlled transfer = done.

**Resources:** RFC 9293 (primary reference, supersedes 793) · `google/netstack` tcpip pkg (best reference once stuck) · `hsheth2/gonet` + its paper ("An Implementation and Analysis of a Kernel Network Stack in Go with the CSP Style") — Go concurrency patterns for timers/retransmission · *TCP/IP Illustrated Vol. 1* (Stevens) — why, not just what

**Exit check:**
- `nc <ip> <port>` connects, interactive typing works
- multi-MB file transfers byte-identical
- connection closes clean, no stuck state

**→ Once this passes: do the CLI-to-package refactor (see top of doc). Stack is stable enough to commit to an API.**

---

## Reference repos, kept open throughout

| Repo | Use for |
|---|---|
| `terassyi/gotcp` | Phases 1–4 structure |
| `hsheth2/gonet` | Go concurrency patterns, phase 6 |
| `google/netstack` (gvisor tcpip) | Production-grade comparison |
| `songgao/water` | TAP/TUN setup, phase 1 |

## Books

- *TCP/IP Illustrated, Vol. 1* — W. Richard Stevens
- *Network Programming with Go* — Adam Woodbeck (net-package level, useful for Go idioms around concurrency/connections)

## Testing rule

Test against real tools (`ping`, `nc`, `curl`, Wireshark) at every phase, not just self-to-self. Sender/receiver agreeing on the same wrong checksum is invisible in self-tests.
