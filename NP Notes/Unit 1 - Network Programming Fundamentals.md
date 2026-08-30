# Unit 1 — Network Programming Fundamentals

**Subject:** Network Programming (CMP 380) · **Unit 1** | **Priority: HIGH**
**Number of teaching hours in syllabus:** 5

> **How to use this note:** this unit is written to give you *complete understanding*, not memorising. Read each section until you can explain it to a friend in your own words and draw the diagrams by hand. If you can do that, an examiner will believe you understand — and give full marks.

---

## 1.1 What is a Computer Network?

A **computer network** = a group of computers (and other devices) connected together so they can **share data and resources**.

- They are connected by cables (Ethernet) or wirelessly (Wi-Fi).
- Each device talks to others using agreed-upon rules called **protocols**.
- The most important family of protocols is **TCP/IP** (Transmission Control Protocol / Internet Protocol).

### Why do we need a network?
1. **Resource sharing** — many users can use one printer, one disk, one server.
2. **Communication** — email, chat, video calls, web browsing.
3. **Centralised management** — data stored on servers, not every PC.

### What is a protocol?
A **protocol** is simply **a set of rules** that two communicating parties agree to follow, so that both sides understand each other. Examples: TCP, UDP, SCTP, IP, HTTP, FTP, SMTP.

---

## 1.2 What is Network Programming?

**Network programming** = **writing programs that let processes talk to each other** — across a network or even on the same machine.

- A **process** is a running program (e.g., a web server, a chat app, a browser).
- The most fundamental tool a network programmer uses is the **socket**.

### Two sides of a conversation: client and server
Almost all network communication follows the **client–server model**:

- **Server** = a process that **waits** for requests, listens on a well-known address (IP + port), and is usually long-running.
- **Client** = a process that **takes the initiative** and sends a request.

```
  CLIENT                                   SERVER
┌──────────┐   sends request ─────────▶  ┌──────────┐
│ process  │                            │ process  │
│ (starts) │  ◀────────── reply ────────  │ (waits)  │
└──────────┘                            └──────────┘
```

**A classic textbook example:** a client sends a **pathname** (a filename) in a request; the server opens that file, and returns the **file's contents** — or an **error** if it cannot open it.

Servers are typed by how they handle many clients:
- **Iterative server** — handles one client at a time; when that client finishes, it moves to the next. Simple but slow.
- **Concurrent server** — forks (creates) another process/thread for each client so many clients are served at once. (More in Unit 3.)

---

## 1.3 Interprocess Communication (IPC)

**IPC** = Interprocess Communication = the different **ways processes exchange messages**.

### Evolution of UNIX IPC (memorise this timeline)
```
Pipes ──▶ Named pipes/FIFOs ──▶ System V msg queues ──▶ POSIX msg queues ──▶ RPC
(only parent/   (any process)    (early 1980s)          (POSIX real-time)    (mid-1980s,
 child)                                                                      call a function
                                                                             on another host)
```

### The 3 ways two UNIX processes share information
1. **Through a file** in the filesystem — each process reads/writes via the kernel. Needs **synchronization**.
2. **Through the kernel** — pipes, System V message queues, semaphores. Every operation is a **system call** (costs time).
3. **Through a shared region of memory** — both processes attach the same memory region and access it directly, **no kernel involvement** after setup. Fastest, but still needs **synchronization**.

```
Diagram - three ways:
process A --file--> [kernel] <--file-- process B          (way 1)
process A --pipes/queues--> [kernel] <-- process B        (way 2)
process A <----shared memory region----> process B        (way 3, direct)
```

### Threads vs Processes
- A **process** has its **own address space**; processes are isolated, heavier to create.
- **Threads** are lightweight and **share the same address space** (global variables are automatically shared between threads of one process).
- For IPC purposes, threads of the same process "share" memory automatically — no kernel message passing needed between them.
- POSIX.1 threads standard: **1995**.

### Semaphore (Dijkstra, late 1960s)
A **semaphore** = a counter/integer used to **control access to a shared resource**.

- **Analogy:** a single-track railway crossing. The semaphore allows only **one train (process)** to enter at a time. Others wait until the track is free.
- Operations: a process **waits** (decrements / blocks if 0) and **signals/post** (increments) when done.
- Used to protect shared memory and other shared resources from being corrupted by simultaneous access.

### POSIX (Portable Operating System Interface)
- A **family of standards** from IEEE (also ISO/IEC 9945) — *not* one standard.
- Three main parts:
  - **Part 1**: System API (C language interfaces)
  - **Part 2**: Shell & utilities
  - **Part 3**: System administration
- **POSIX IPC** = POSIX message queues + POSIX semaphores + POSIX shared memory.

---

## 1.4 RPC and BSD History (short notes / background)

### Remote Procedure Call (RPC)
- **Local procedure call** = function and caller in the **same process** (normal C function call).
- **RPC (same host)** = caller (client) and called procedure (server) are in **different processes** on the same machine.
- **RPC over network** = the client on one machine calls a procedure that runs **on another machine** over the network.

```
LOCAL                 REMOTE (same host)        REMOTE (network)
┌────────┐ ┌─────┐    ┌────────┐ ┌────────┐     host1 ──network── host2
│ caller │→│ fn  │    │ client │ │ server │     [client]  call  [server]
└────────┘ └─────┘    └────────┘ └────────┘     [client]◀─return─[server]
```

### The BSD Networking History (know the timeline)
- **1982**: 4.2BSD introduces sockets (Berkley Software Distribution, at UC Berkeley).
- Socket = a generalised IPC mechanism for both **local** (Unix domain) and **network** communication.
- **1983**: 4.3BSD refines sockets.
- **1990**: 4.3BSD Reno adds ISO/OSI sockets + SCTP later; POSIX.1g standardises sockets.
- The socket API became the **de-facto standard** for network programming; Windows later copied it as **Winsock**.

### Importance of networking & network programming (asked in Gandaki paper)
- Networking lets machines work together, share resources and data.
- Network programming is what *builds* the applications (web servers, messengers, databases) on top of networks.
- Without network programming, the network exists but does nothing useful for users.

---

## 1.5 The TCP/IP Protocol Suite and the communication protocols

The protocols we use in network programming are Transport-layer protocols. Three matter: **TCP, UDP, SCTP**. They sit **above IP** and **below the application**.

```
┌─────────────────────── Application (HTTP, FTP, SMTP, ...) ───────────────────────┐
│        TCP               UDP                SCTP        ← transport layer       │
│                    IP (IPv4 / IPv6)                            ← network layer   │
│                 Link (Ethernet, Wi-Fi)                         ← data link       │
└──────────────────────────────────────────────────────────────────────────────────┘
```

### TCP — Transmission Control Protocol
- **Connection-oriented**: a connection must be established before data flows.
- **Reliable**: uses **acknowledgments (ACKs)** and **retransmission** — lost data is resent.
- **Ordered**: ensures bytes arrive in order.
- **Byte-stream**: no message boundaries — you read/write a stream of bytes.
- **Flow control**: receiver advertises a **window** so sender doesn't overflow it.
- **Congestion control**: adapts sending rate to network load (slow start, congestion avoidance).
- Used by: HTTP, FTP, SMTP, SSH — anything needing reliability.

### UDP — User Datagram Protocol
- **Connectionless**: no handshake; just send datagrams.
- **Unreliable**: no ACK, no retransmission — packets may be lost, duplicated, or reordered.
- **Message-oriented**: preserves **message boundaries** (each send = one datagram).
- **Low overhead, low latency**.
- Used by: DNS, NFS, SNMP, streaming, real-time apps where a little loss is acceptable.

### SCTP — Stream Control Transmission Protocol (NEWER, telephony-focused)
- **Connection-oriented**, but the connection is called an **association**.
- **Reliable** (like TCP).
- **Message-oriented** (preserves boundaries, unlike TCP) AND supports **multistreaming** (independent ordered streams inside one association — avoids head-of-line blocking).
- **Multi-homing**: one association can use multiple IP addresses at each end for fault tolerance.
- **Four-way handshake with a COOKIE** to resist SYN-flood DoS attacks (TCP is vulnerable).
- **No half-open state** (unlike TCP which allows half-closed connections).
- Used by: telephony signaling (SIGTRAN), Diameter, etc.

### TCP vs UDP vs SCTP — comparison table (asked EVERY year)
| Feature | TCP | UDP | SCTP |
|---|---|---|---|
| Connection | Oriented | **Connectionless** | Oriented (association) |
| Reliability | Reliable | **Unreliable** | Reliable |
| Ordering | Byte stream, ordered | No ordering | Ordered per stream |
| Message boundaries | **No** (stream) | **Yes** | **Yes** |
| Multi-homing | No | No | **Yes** |
| Multistreaming | No | No | **Yes** |
| Handshake | 3-way | none | **4-way + cookie** |
| Half-open support | Yes | n/a | **No** |
| Anti-SYN-flood | Weak | n/a | **Strong (cookie)** |
| Common use | HTTP, FTP, SMTP | DNS, NFS, SNMP | Telephony signaling |

---

## 1.6 TCP Connection Establishment — The Three-Way Handshake (HIGH)

### Terms
- **Active open** = the client initiating the connection (calls `connect`).
- **Passive open** = the server waiting for connections (calls `listen`; enters LISTEN state).

### The handshake (3 segments)
```
CLIENT (active)                             SERVER (passive)
   │  SYN (seq = x, SYN flag set)                │
   │ ────────────────────────────────────▶       │  (client asks to open)
   │  SYN+ACK (seq = y, ack = x+1)               │
   │ ◀────────────────────────────────────       │  (server agrees + invites)
   │  ACK (ack = y+1)                            │
   │ ────────────────────────────────────▶       │  (client confirms)
   │          CONNECTION ESTABLISHED             │
```
Explanation:
1. Client sends **SYN** with an initial sequence number **x**.
2. Server responds **SYN+ACK** — its own sequence number **y**, and acknowledges `x+1`.
3. Client sends **ACK** acknowledging `y+1`.

> Why 3, not 2? Because both sequence numbers must be synchronised **in both directions**. It also lets the server confirm the client can actually receive (connection is bidirectional).

### Why the initial sequence number (ISN) should NOT start from 0 (HIGH — asked directly)
- If ISN always started at 0, an **old, delayed segment** from a previous, now-closed connection could arrive with a sequence number that **overlaps** the new connection's data. The receiver would accept it as *valid new data* → data corruption (the "wandering duplicate" / "lost duplicate" problem).
- Starting from an **unpredictable/random ISN** (RFC 6528 recommends randomising it) makes it very unlikely that a stray old segment's number falls inside the current connection's window.
- Also, a predictable ISN is a **security risk** (session hijacking).
- The **TIME_WAIT** state (2× MSL) gives additional protection by letting old duplicates **expire** in the network before the port is reused.

---

## 1.7 The TCP State-Transition Diagram (HIGH — asked EVERY year, often 8 marks)

TCP defines **11 states**. `CLOSED` is *fictional* (represents "no connection"). The states are the same names shown by the `netstat` / `ss` command.

### The 11 states
| State | Meaning (one line) |
|---|---|
| **CLOSED** | No connection exists (start/end point). |
| **LISTEN** | Server waiting for a connection request (passive open). |
| **SYN_SENT** | Client sent SYN, waiting for SYN+ACK (active open). |
| **SYN_RCVD** | Server got SYN, sent SYN+ACK, waiting for final ACK. |
| **ESTABLISHED** | Connection open — normal data-transfer state. |
| **FIN_WAIT_1** | Sent FIN, waiting for ACK (or FIN+ACK). |
| **FIN_WAIT_2** | Got ACK of FIN; waiting for other side's FIN. |
| **CLOSE_WAIT** | Got other side's FIN, sent ACK; app hasn't closed yet. |
| **CLOSING** | Both sides sent FIN simultaneously (rare). |
| **LAST_ACK** | Sent our FIN, waiting for final ACK. |
| **TIME_WAIT** | Sent final ACK; waiting 2×MSL for old segments to expire. |

### Establishment path (draw this)
```
                  (passive open = listen)
 CLOSED ───────────────────────────────▶ LISTEN
   │                                        │  (recv SYN, send SYN+ACK)
   │ (active open = connect; send SYN)      ▼
   │                                      SYN_RCVD
   ▼                                        │  (recv ACK)
 SYN_SENT ──────── (recv SYN+ACK,          ▼
   │                send ACK) ──────▶  ESTABLISHED
   │                                     │  ◀── data transfer happens here
   └─────── (simultaneous open via       │
            SYN_SENT → SYN_RCVD) ────────┘
```

### Termination path (the "four-way handshake")
```
ESTABLISHED           (active close)            ESTABLISHED   (passive close)
   │ ①FIN ─────────────────────────────────────▶ │
FIN_WAIT_1                                         │  (recv FIN)
   │◀───────────────────────── ②ACK ───────────── CLOSE_WAIT   (app hasn't closed)
FIN_WAIT_2                                         │  (app calls close())
   │◀───────────────────────── ③FIN ───────────── LAST_ACK
   │ ④ACK ─────────────────────────────────────▶  │
TIME_WAIT (2×MSL)                                  │  (recv ACK)
   ▼                                               ▼
 CLOSED                                          CLOSED
```
- The side that **initiates** the close (calls `close` first) is the **active closer** → goes through `FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT`.
- The other side is the **passive closer** → goes through `CLOSE_WAIT → LAST_ACK`.
- Both sides must each send a FIN and receive the other's FIN + ACK = 4 segments (FIN/ACK in both directions).

### TIME_WAIT — why it exists (and why 2×MSL)
**MSL** = Maximum Segment Lifetime = the longest time a segment can live in the network (RFC 1122 recommends 2 minutes; Berkeley uses 30 seconds). Because every IP datagram has a max **TTL/hop-limit** (255), its lifetime is bounded.

After the **active close** sends its final ACK, TCP stays in **TIME_WAIT for 2×MSL**. Two reasons:
1. **Reliable full-duplex termination** — if the final ACK is lost, the other side retransmits its FIN, and TIME_WAIT lets us resend the ACK.
2. **Allow old duplicate segments to expire** — any wandering/lost duplicates from this connection die out in the network, so they can't contaminate a *future* connection reusing the same port.

> Common exam pitfall: TIME_WAIT is **normal and correct**, not a bug. It's what protects a restarted server's port from stale data.

### CLOSING state
Both sides try to close at the **same time** — each sends a FIN and enters FIN_WAIT_1; each receives the other's FIN and moves to **CLOSING** instead of FIN_WAIT_2; each sends an ACK, then goes to TIME_WAIT.

### Connection reset (RST)
If something goes wrong (port not listening, crash, protocol error), a **RST (reset)** segment aborts the connection immediately — no graceful close.

---

## 1.8 TCP Reliability, Flow & Congestion Control (deep understanding)

### Sequence numbers & ACKs
- Every byte in the stream has a **sequence number**.
- The **ACK number** tells the sender: "I've received all bytes up to (ack-1); please send from ack onward."
- This makes TCP **reliable** — missing bytes are retransmitted.

### The advertised (receive) window — Flow Control
- Each side tells the other how much **receive buffer** space is free (the **advertised window**).
- The sender must not send more than the window → prevents **overflowing** the receiver's buffer.
- The window changes dynamically: shrinks as data arrives, grows as the app reads.

### Sliding window
The sender keeps track of a window of unacknowledged bytes. As ACKs arrive, the window **slides forward**, allowing new data to be sent. This lets TCP send many bytes in flight (pipelining) instead of one-at-a-time.

### TCP flags (mentioned in the PPTX, good for understanding)
- **SYN** — synchronise (connection request).
- **ACK** — acknowledgment.
- **FIN** — finish (close).
- **RST** — reset (abort).
- **PSH** — push: deliver data to application immediately.
- **URG** — urgent data (rarely used today).
- **ECE / CWR** — for Explicit Congestion Notification (ECN).

### Congestion control (TCP adapts to network load)
- **Slow start**: begin with a small congestion window (cwnd); double it each RTT until a threshold (ssthresh) — grows quickly.
- **Congestion avoidance**: after ssthresh, grow cwnd slowly (roughly +1 per RTT) to probe for available capacity.
- On **packet loss**, TCP reduces cwnd (halves it, or drops to 1 in slow start) — this throttles the sender so it doesn't overrun the network.

### Nagle's algorithm & delayed ACK (worth knowing)
- **Nagle's algorithm**: don't send small segments if there is already unacknowledged data in flight — coalesce small writes into bigger packets. Good for throughput, can add latency for tiny interactive messages.
- **TCP_NODELAY** socket option disables Nagle — used for low-latency interactive apps (e.g., SSH).
- **Delayed ACK**: the receiver doesn't ACK every segment immediately; it waits a little (up to ~200 ms) hoping to bundle the ACK with response data.
- These two interacting can cause a performance deadlock for latency-sensitive apps — hence TCP_NODELAY.

---

## 1.9 The Port Numbers and the Socket Pair

### Port numbers (ranges)
- **0–1023**: **well-known** ports (controlled by IANA), e.g., 80=HTTP, 443=HTTPS, 21=FTP, 25=SMTP, 53=DNS.
- **1024–49151**: **registered** ports (IANA-registered, e.g., 1433=SQL Server, 3306=MySQL).
- **49152–65535**: **dynamic / private / ephemeral** ports — automatically assigned to client sockets.

### What is a socket?
A **socket** = an **endpoint for communication**. At the TCP/UDP level, it is identified by the **IP address + port number**.

- **Socket address** = (IP address, port number).
- When two sockets exchange data, the pair forms a connection.

### Socket pair (HIGH)
For a TCP stream, the **socket pair** is a **4-tuple**:
```
(local IP, local port, foreign IP, foreign port)
```
This 4-tuple **uniquely identifies every TCP connection on the Internet.** Two sockets in the pair are called the **local socket** and the **foreign (remote) socket**.

### Types of sockets
| Socket type | Protocol | Characteristics |
|---|---|---|
| **SOCK_STREAM** | TCP | Connection-oriented, reliable, ordered **byte stream** (no boundaries) |
| **SOCK_DGRAM** | UDP | Connectionless, unreliable, preserves **message/datagram boundaries** |
| **SOCK_SEQPACKET** | SCTP | Connection-oriented, reliable, ordered **messages** |
| **SOCK_RAW** | IP | Direct access to IP packets (low-level tools like ping) |

---

## Quick Memory Sheet (Unit 1)
- **Network programming** = processes talking via sockets. **Client** initiates; **server** waits.
- **IPC timeline**: Pipes → FIFOs → SysV msg queues → POSIX msg queues → RPC.
- **3 share ways**: file / kernel (pipes-queues-semaphores) / shared memory.
- **TCP**=reliable stream, **UDP**=unreliable datagram, **SCTP**=reliable message + multistreaming + **multi-homing** + 4-way cookie handshake.
- **3-way handshake**: SYN → SYN+ACK → ACK (synchronises ISNs both ways).
- **ISN should be random, not 0** — else old duplicate segments corrupt new connections.
- **11 TCP states**; **TIME_WAIT (2×MSL)** protects from old duplicates + reliable close.
- **Socket pair** = (local IP, local port, foreign IP, foreign port).
- **Ports**: 0–1023 well-known, 1024–49151 registered, 49152–65535 ephemeral.
