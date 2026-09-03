# Network Programming — Exam Questions & Answers

**CMP 380 Network Programming (Pokhara University)**
Compiled from the **actual NCIT Spring 2025** and **Gandaki College 2025** question papers (extracted from screenshots), the official syllabus, and the full study notes.

## How to use this guide
- Each question has a **priority rating** based on how often it appeared in real past papers:
  - 🔴 **HIGH** — appears in nearly every paper / asked repeatedly (study first, be able to write full 8-mark answer).
  - 🟡 **MEDIUM** — appears often, usually as 5–7 mark (know well).
  - 🟢 **LOW** — appears occasionally / as a short note.
- Questions asked almost **word-for-word** in the captured papers are marked **★ (asked in real paper)**.
- Every answer is written out as **detailed bullet points** and ends with a **"Marking scheme (how to get full marks)"** box that tells you exactly which parts earn which marks — so you can allocate your exam time to the highest-value points.

---

## Unit 1 — Fundamentals

### Q1 🔴★ Compare and contrast TCP, UDP, and SCTP. (asked: NCIT 2025 Q1a)
All three are **transport-layer protocols** (Layer 4 of the OSI model) that move application data between two hosts. They differ in **connection**, **reliability**, **ordering**, and **message handling**.

**Transport layer context:**
- Sits between the application (HTTP, DNS, video…) and the network layer (IP).
- Responsible for **end-to-end delivery**, **multiplexing** (port numbers), and often **reliability**.

**Point-by-point comparison:**

| Feature | TCP | UDP | SCTP |
|---|---|---|---|
| Connection | Connection-**oriented** (3-way handshake first) | **Connectionless** (send whenever) | Connection-oriented (4-way handshake) |
| Reliability | Reliable via ACK + retransmission | Unreliable (best effort) | Reliable (ACK + retransmit) |
| Ordering | Byte stream, **sequenced** | No ordering | **Message-ordered** |
| Message boundaries | **No** (pure byte stream) | **Yes** (datagrams) | **Yes** (messages) |
| Flow/congestion control | Yes | No | Yes |
| Multi-homing | No (one IP at a time) | No | **Yes** (multiple IPs per endpoint) |
| Data transfer | Stream | Datagram | **Message + stream** |
| Handshake | 3-way (SYN/SYN+ACK/ACK) | none | **4-way** (INIT/INIT-ACK/COOKIE) |
| Protection against SYN flood | Partial | n/a | **4-way handshake w/ cookie** |
| Usage | HTTP, FTP, SMTP, SSH | DNS, NFS, SNMP, VoIP, streaming | Telephony/Signaling (SIGTRAN), IP telephony |

**Why these differences matter:**
- **Reliability** costs overhead (headers, ACKs) — TCP is right when data must arrive intact (files, web).
- **No reliability** gives low latency and tiny headers — UDP is right for DNS (one query/response), live audio/video and games where a late packet matters more than a missing one.
- **SCTP chooses the best of both:** reliable AND preserves message boundaries AND survives a network-interface failure (multi-homing). That is why it is used in telephone signalling — a call must not drop just because one link fails.

**What an examiner looks for:** a table (bulk of marks), a "why each is used" line, and one real example per protocol. Draw a small diagram of the TCP vs UDP packet if you have time.

---
**Marking scheme (8 marks):** table = **4**, context sentece = **1**, why-differences-matter = **2**, examples = **1**.

---

### Q2 🔴★ Explain the TCP three-way handshake. (asked in multiple papers)
The **three-way handshake** is the process by which a **client (active open)** and a **server (passive open)** establish a reliable TCP connection by exchanging three segments before any data flows.

```
CLIENT                                  SERVER
   │                                      │
   │  1. SYN  (seq = x)                   │
   │ ──────────────────────────────────▶  │  "I want to connect; my start is x"
   │                                      │
   │  2. SYN+ACK (seq = y, ack = x+1)     │
   │ ◀──────────────────────────────────  │  "OK, I agree; my start is y"
   │                                      │
   │  3. ACK (ack = y+1)                  │
   │ ──────────────────────────────────▶  │  "I received your start"
   │                                      │
   │      CONNECTION ESTABLISHED          │
```

**Step-by-step explanation:**
1. **Client → SYN (seq = x):** the client sends a segment with the **SYN** flag set and an initial sequence number **x**. This is the *active open*.
2. **Server → SYN+ACK (seq = y, ack = x+1):** the server acknowledges the client's sequence (ack = x+1) and sends its **own** initial sequence number **y**. This is the *passive open*.
3. **Client → ACK (ack = y+1):** the client acknowledges the server's sequence number. Now **both** sides know the other is alive and ready to send.

**Why called "three-way":** because exactly **three segments/packets** are exchanged (the SYN and ACK in step 2 are combined into one segment — this is why it is not two).
**What two-way handshake would fail:** the server could never be sure the client actually received its SYN — so the server would "assume" the connection even if the client never agreed (classic two-army problem).

**Purpose / guarantees:**
- Verifies both sides are **reachable** and **ready**.
- Exchanges each side's **initial sequence numbers**, so subsequent data segments can be ordered and duplicates detected.
- Negotiates TCP options (MSS, window scaling, timestamps) during the SYN segments.

---
**Marking scheme (7–8 marks):** correct diagram with all seq/ack labels = **3**, each step explained = **3**, "why three-way not two-way" + purpose = **2**.

---

### Q3 🔴★ Why should the initial sequence number NOT start from 0? Explain TCP state transition diagram. (asked: NCIT 2025 Q1a — 7 marks)

**Part A — Why ISN should not start from 0:**

- TCP data is identified by **32-bit sequence numbers**. If every connection started at **0**, two different connections (one after the other, or on the same port) would use **overlapping sequence numbers**.
- An **old, delayed segment** from a previous connection still floating in the network could carry a sequence number that **collides** with the new connection's numbers. The receiver would mistake this stale packet for **valid new data** — corruption without error.
- If ISN = 0 always, then an attacker can **guess/forge** sequence numbers and inject fake data (sequence-number prediction attack).
- Solution: each side chooses a **random / unpredictable ISN** AND uses **TIME_WAIT** so old duplicates expire (after 2×MSL) before the numbers can wrap or be reused. The classic rule is the ISN incrementing by a clock roughly every 4 µs, making prediction hard.

Takeaway line: *"A non-zero, unpredictable ISN + TIME_WAIT prevents a stale segment from being mistaken for new data, and defeats sequence-number guessing."*

**Part B — TCP state-transition diagram (draw this, it is the single most-asked diagram):**

```
           active open
   CLOSED ─────────────▶ SYN_SENT
      ▲                    │
      │ (close)            │ recv SYN+ACK, send ACK
      │                    ▼
      │                ESTABLISHED ────────────── passive open path
      │                      │                      ▲
      │  FIN_WAIT_1          │                      │
      │    │                 │ (recv SYN) → SYN_RCVD │
      │    │  send FIN       │        (send SYN+ACK) │
      │    ▼                 │                       │
      │ FIN_WAIT_2          │ (from LISTEN)          │
      │    │                 ▼                       │
      │    │             ESTABLISHED ◀───────────────┘
      │    │  (recv ACK)      │
      │    ▼                  │  send FIN
      │  TIME_WAIT            ▼
      │  (2×MSL)          CLOSE_WAIT
      │                      │  send FIN
      │                      ▼
      │                   LAST_ACK
      │                      │
      └───────────────────── │ (recv ACK of FIN)
                           CLOSED

 Server path:  CLOSED ─▶ LISTEN ─(recv SYN)▶ SYN_RCVD ─(send SYN+ACK)▶ ESTABLISHED
```

**The 11 states (brief):** `CLOSED`, `LISTEN`, `SYN_SENT`, `SYN_RCVD`, `ESTABLISHED`, `FIN_WAIT_1`, `FIN_WAIT_2`, `CLOSE_WAIT`, `LAST_ACK`, `TIME_WAIT`, `CLOSING`.
- Server: `CLOSED → LISTEN → SYN_RCVD → ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED`.
- Client: `CLOSED → SYN_SENT → ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED`.

**Which marks come from which:** drawing the state diagram is worth most (it is proof of understanding). Then write one sentence per key state showing *which side* is in it and *what event* moves it.

---
**Marking scheme (7 marks):** ISN part = **3** (problem, TIME_WAIT, security), state diagram = **3**, one sentence per key state = **1**.

---

### Q4 🟡★ What is a socket? What are the types of sockets? (asked: NCIT 2025 Q2 — differentiate TCP vs UDP socket)
A **socket** is the **programming endpoint** of a two-way communication link — a **combination of IP address + port number** used to uniquely identify the end of a connection. It is the interface an application uses to send/receive data over the network.

**Types of sockets:**
1. **Stream socket (SOCK_STREAM)** → uses **TCP** — connection-oriented, reliable, ordered **byte stream** (no message boundaries).
2. **Datagram socket (SOCK_DGRAM)** → uses **UDP** — connectionless, unreliable, preserves **message (datagram) boundaries**.
3. **Raw socket (SOCK_RAW)** → direct access to **IP packets** (build your own headers) — for low-level tools like `ping`, packet sniffers, routing protocols.
4. (Unix also has **sequenced-packet** sockets `SOCK_SEQPACKET` — reliable, ordered *messages*; used with SCTP/Unix domain.)

**TCP socket vs UDP socket (asked directly):**

| | TCP socket | UDP socket |
|---|---|---|
| Type constant | `SOCK_STREAM` | `SOCK_DGRAM` |
| Connection | must `connect` first | connectionless |
| Reliable | yes (ack/retransmit) | no |
| Ordering | byte stream (no boundaries) | preserves datagram boundaries |
| Server calls | `bind → listen → accept` | `bind → recvfrom` only (no listen/accept) |
| Client send | `send`/`write` | `sendto` |
| Data recv | `recv`/`read` (stream, may be partial) | `recvfrom` (one whole datagram) |

**How a socket is created (both):** `socket(family, type, protocol)` e.g. `socket(AF_INET, SOCK_STREAM, 0)`.

---
**Marking scheme (6–8 marks):** definition of socket = **2**, three socket types = **3**, TCP-vs-UDP table = **3**.

---

### Q5 🟢 What is IPC? List the evolution of UNIX IPC mechanisms.
**IPC (Interprocess Communication)** = the mechanisms that let **two or more processes exchange data** and synchronize with each other. Since processes have **separate address spaces**, they need a common channel (file, kernel object, or shared memory) to talk.

**Evolution (timeline):**
```
Pipes ─▶ Named pipes (FIFOs) ─▶ System V message queues ─▶ POSIX msg queues ─▶ RPC
                                                                         ─▶ Sockets/Networks
```

**Each in one line:**
1. **Pipes** — unidirectional byte stream, parent↔child only, no name.
2. **FIFOs (named pipes)** — same but have a pathname, so unrelated processes can open them.
3. **System V message queues** — kernel-managed, message-addressed, key-based.
4. **POSIX message queues** — modern replacement, namespaced (`/mq`).
5. **RPC / sockets** — let processes on *different* machines communicate (the basis of network programming).
6. **Shared memory** — fastest (no kernel copy) but needs synchronization (semaphores).

**Why this matters for a network course:** most IPC is *local* (same machine); **sockets** extend IPC to *remote* machines — which is exactly what network programming (this course) is about.

---
**Marking scheme (4–5 marks):** definition = **1**, correct chronological list = **2**, one line per mechanism = **2**.

---

### Q6 🟢 What are the three ways two UNIX processes can share info?
UNIX provides exactly **three** classes of info-sharing between processes:

1. **Through a shared file** in the filesystem — both processes read/write the same file through the kernel. Simple but needs **synchronization** (locking) and is **slow** (disk I/O).
2. **Through the kernel** — pipes, FIFOs, message queues, semaphores. Each operation is a **system call** into the kernel which holds/stores the data. Needs synchronization (esp. for semaphores).
3. **Through shared memory** — both map the **same physical memory** into their address space. **Fastest** (no kernel copy per access) but **must synchronize** manually (semaphores/mutexes) and has no kernel mediation.

**Why three and not one:** each is a speed-vs-simplicity trade-off — files are easiest but slowest; shared memory fastest but most error-prone; kernel objects are the middle ground.

---
**Marking scheme (4–5 marks):** each way = **1**, plus **1–2** for the trade-off/why-all-three explanation.

---

### Q7 🟡★ Define the port number ranges & socket pair.
A **port number** is a 16-bit value (0–65535) that identifies a specific application on a host, letting many applications share one IP.

**The three ranges:**

| Range | Name | Notes / examples |
|---|---|---|
| **0–1023** | **Well-known** | Standard services; e.g. 80 HTTP, 443 HTTPS, 21 FTP, 25 SMTP, 53 DNS |
| **1024–49151** | **Registered** | Applications register these with IANA; e.g. 1433 SQL Server, 3306 MySQL |
| **49152–65535** | **Dynamic / private (ephemeral)** | Auto-assigned by the OS to clients for the **local** end of a connection |

**Socket pair (TCP):** a TCP connection is uniquely identified by the **4-tuple**:
```
(local IP, local port, foreign IP, foreign port)
```
- **local IP & port** = your side, **foreign IP & port** = the peer side.
- The **socket pair** is what allows two hosts to run many simultaneous connections (they all differ in at least one of the four values).
- Compare: one *socket* = (local IP, local port); the *socket pair* = the full 4-tuple.

---
**Marking scheme (5 marks):** three ranges = **3**, well-known examples = **1**, socket-pair 4-tuple + why unique = **1–2**.

---

### Q8 🟡★ How is a TCP connection terminated? Why is TIME_WAIT needed?
**Graceful close uses four segments** — a full-duplex FIN/ACK exchange (each direction closes independently):
```
A ── FIN ──▶ B      (A says: I have no more data to send)
A ◀── ACK ── B      (B acknowledges A's FIN)
A ◀── FIN ── B      (B also finishes its side)
A ── ACK ──▶ B      (A acknowledges B's FIN)  → connection fully closed
```

**Who does what:** the side that calls `close()` first initiates the FIN; both directions are closed independently, so a connection can be **half-closed** (one side done sending, still receiving).

**Why TIME_WAIT is needed (lasts 2×MSL, Max Segment Lifetime — typically ~2 minutes):**
1. **Reliable full-duplex close** — if the **final ACK is lost**, the peer will re-transmit its FIN; TIME_WAIT lets the closing side re-send the ACK. Without it, the peer would be stuck in LAST_ACK.
2. **Let old duplicate segments expire** — a delayed packet from *this* connection could linger in the network. TIME_WAIT holds the 4-tuple long enough (2×MSL) that any duplicate either arrives (and is ignored) or dies, so it **cannot contaminate a new connection** that reuses the same local IP/port.

**Trade-off:** TIME_WAIT is why a server restarting sometimes gets **"address already in use"** — solved by `SO_REUSEADDR`.

---
**Marking scheme (5–6 marks):** 4-segment diagram = **2**, half-close idea = **1**, the two TIME_WAIT reasons = **2–3**.

---
## Unit 2 — Unix Basics

### Q9 🔴★ Explain value-result arguments in socket programming. Why are they needed? (asked: NCIT 2025 Q3a, Gandaki 2025 Q2b)
A **value-result argument** is an argument that is used **two ways**: its *value* is read on input (to pass info to the kernel), and the kernel *writes a result back* into the same variable before returning — so after the call the variable holds new data.

**Two directions of length passing in socket calls:**
- **Process → kernel** (`bind`, `connect`, `sendto`): you pass the size **by value** — the kernel only *reads* it to know how many bytes to copy from the caller's buffer.
- **Kernel → process** (`accept`, `recvfrom`, `getsockname`, `getpeername`): you pass a **pointer to the size** (`socklen_t *`). On input the *value* tells the kernel the buffer's size (so it never writes past the end); on output the kernel *updates* it to the actual number of bytes it stored. This is the **value-result** pattern.

```c
struct sockaddr_in cli;
socklen_t len = sizeof(cli);          /* value: how big my buffer is */
int cfd = accept(listenfd, (SA*)&cli, &len);  /* kernel returns real size in len → result */
printf("peer stored %d bytes in cli\n", len); /* now len = actual length */
```

**Why value-result is needed:**
1. **Safety vs overflow:** the kernel must know the buffer size or it could overwrite memory (buffer overflow).
2. **Variable data size:** different address families have different lengths (IPv4 = 16 B, IPv6 = 28 B). The caller sets input size; the kernel reports the actual size on output, so the caller can tell which family was received.

**Common mistake:** passing `&len` uninitialized — the kernel reads garbage → undefined/unpredictable behaviour. Always set `len = sizeof(buffer)` first.

---
**Marking scheme (8 marks):** definition = **2**, two directions table = **2**, code with value+result explained = **2**, why-needed = **2**.

---

### Q10 🔴★ Ways to pass the length of a socket structure — with function prototypes. (asked: NCIT 2025 Q2a)
There are exactly **two** ways to pass a socket-address length, matching the two groups of callers above.

**Group 1 — length passed BY VALUE (process → kernel, one-way):**
```c
int bind(int socket, const struct sockaddr *address, socklen_t address_len);
int connect(int socket, const struct sockaddr *address, socklen_t address_len);
ssize_t sendto(int socket, const void *buffer, size_t length, int flags,
               const struct sockaddr *dest_addr, socklen_t dest_len);
```
Here `address_len` is a plain value the kernel reads (tells it how many bytes of `address` to copy).

**Group 2 — length passed BY REFERENCE, value-result (kernel → process, two-way):**
```c
int accept(int socket, struct sockaddr *address, socklen_t *address_len);
ssize_t recvfrom(int socket, void *buffer, size_t length, int flags,
                 struct sockaddr *address, socklen_t *address_len);
int getsockname(int socket, struct sockaddr *address, socklen_t *address_len);
int getpeername(int socket, struct sockaddr *address, socklen_t *address_len);
```
In group 2, `address_len` is a **pointer (`&len`)**. The caller must set `*address_len` to the buffer size *before* the call; the kernel writes back the **actual** size after. That is value-result.

**How to remember which group:**
- If the *kernel produces* the address (accept/recvfrom/getsockname/getpeername) → the length is **value-result** (pointer).
- If the *we supply* the address (bind/connect/sendto) → the length is **by value**.

**Common exam trap:** forgetting to initialise `*address_len` before an `accept`/`recvfrom` call → garbage length → overflow or wrong data. Always do `socklen_t len = sizeof(addr);`.

---
**Marking scheme (8 marks):** stating the two ways = **2**, correct prototypes for each group = **4** (2 each), how-to-remember rule = **1**, common trap = **1**.

---

### Q11 🔴★ Explain the socket address structures (sockaddr, sockaddr_in, sockaddr_in6, sockaddr_storage). (asked: NCIT 2025 Q2b, Gandaki 2025 Q2a/Q2b)

**What they are:** socket address structures hold the **family** (AF_INET/AF_INET6/AF_UNIX) + **port** + **address** of an endpoint, and are passed to every socket call.

| Structure | Family | Size | Key fields | Purpose |
|---|---|---|---|---|
| `struct sockaddr` | generic | 16 B | `sa_family`, `sa_data[14]` | generic/old casting form used by all socket functions |
| `struct sockaddr_in` | AF_INET | 16 B | `sin_family`, `sin_port`, `sin_addr.s_addr`, `sin_zero[8]` | IPv4 addresses |
| `struct sockaddr_in6` | AF_INET6 | 28 B | `sin6_family`, `sin6_port`, `sin6_flowinfo`, `sin6_addr`, `sin6_scope_id` | IPv6 addresses |
| `struct sockaddr_storage` | generic | ≥128 B | opaque; aligned for any family | can hold **any** family safely |

**Field-by-field (sockaddr_in):**
- `sin_family` — always `AF_INET` (2 bytes).
- `sin_port` — 16-bit port in **network byte order** → set with `htons()`.
- `sin_addr.s_addr` — 32-bit IPv4 address in **network byte order** → set with `htonl()` or `inet_pton()`.
- `sin_zero[8]` — padding to match `sockaddr` size; you must **zero** it (usually `bzero()` the whole struct).

**Why every call takes `struct sockaddr *`:**
All socket functions take a **generic pointer** `(const struct sockaddr *)`. You *declare* a concrete struct (`sockaddr_in`) but *cast* it to `(struct sockaddr *)` when calling `bind/connect/accept`. This is the classic **polymorphism via casting** trick in C:
```c
struct sockaddr_in serv;
bind(sockfd, (struct sockaddr *)&serv, sizeof(serv));
```

**Why `sockaddr_storage` is significant (asked):**
- It is **big enough (≥128 B)** to hold the **largest** socket-address type the system supports (IPv4 *or* IPv6).
- It has the **strictest alignment**, so you can declare it, pass it to `accept`/`recvfrom`, and inspect `ss_family` afterwards to learn which family actually arrived — safely, without knowing it in advance. This is how you write **family-neutral** servers.

**Important field-naming note:** the generic struct's fields are `sa_family`, `sa_data`; the IPv4 struct's are `sin_*`; the IPv6 struct's are `sin6_*`; storage's is `ss_family`. Don't confuse them.

---
**Marking scheme (8 marks):** table of 4 structs = **4**, one-field explanation (sockaddr_in) = **1**, generic-pointer casting = **1**, sockaddr_storage significance = **2**.

---

### Q12 🔴★ Byte ordering & manipulation functions. (asked: NCIT 2025 Q2a)

**Problem:** different CPUs store multi-byte integers differently:
- **Big-endian:** most-significant byte first (e.g. `0x1234` → `12 34`).
- **Little-endian** (Intel): least-significant byte first (`0x1234` → `34 12`).
- TCP/IP protocols mandate **network byte order = big-endian**. If we sent raw host values, a little-endian machine and a big-endian machine would read the port/address differently.

**Solution — the four conversion functions:**
```c
uint16_t htons(uint16_t hostshort);  /* Host → Network, Short (16-bit, ports)  */
uint32_t htonl(uint32_t hostlong);   /* Host → Network, Long  (32-bit, addresses) */
uint16_t ntohs(uint16_t netshort);   /* Network → Host, Short */
uint32_t ntohl(uint32_t netlong);    /* Network → Host, Long  */
```
- `htons`/`htonl` **before sending** (build `sin_port` with `htons(...)`, `sin_addr.s_addr` with `htonl(...)` or `inet_pton`).
- `ntohs`/`ntohl` **after receiving** (read `sin_port`, `sin_addr`).
- On a big-endian machine these are no-ops; on a little-endian machine they byte-swap — so **always use them**, never assume.

**Manipulation functions** — for binary data (not C strings):
- BSD: `bzero(ptr, n)`, `bcopy(src,dst,n)`, `bcmp(...)`.
- ANSI (modern, preferred): `memset(ptr, 0, n)`, `memcpy(dst,src,n)`, `memcmp(a,b,n)`.
- Use them to **zero/initialise** address structures (e.g. `bzero(&serv, sizeof(serv))` or `memset(&serv, 0, sizeof(serv))`) before filling the fields — this clears `sin_zero` padding and prevents leaving uninitialised data.

---
**Marking scheme (8 marks):** endianness concept = **2**, four functions = **2**, when to apply each = **2**, manipulation functions = **2**.

---

### Q13 🟡★ inet_aton / inet_addr / inet_ntoa / inet_pton / inet_ntop.
These convert between **presentation format** (a human dotted string like `"192.168.1.1"`) and **numeric binary** form (the 32-bit value stored in `sin_addr`).

- **`inet_aton("1.2.3.4", &addr)`** — ASCII → binary. Returns non-zero on success, 0 on failure. **Preferred** for IPv4 (better error reporting). `addr` is a `struct in_addr`.
- **`inet_addr("1.2.3.4")`** — like `inet_aton` but returns the address **by value** (`in_addr_t`); returns `INADDR_NONE` on error. **Problem:** `255.255.255.255` also equals `INADDR_NONE`, so it can't be distinguished from an error — avoid it.
- **`inet_ntoa(addr)`** — binary → ASCII string (returns pointer to a **static** buffer). **Not thread-safe** (overwritten by next call). Deprecated by POSIX.
- **`inet_pton(family, src, dst)`** — **presentation → numeric for BOTH IPv4 and IPv6** (`AF_INET` or `AF_INET6`). Returns 1 on success, 0 (invalid) or -1 (bad family).
- **`inet_ntop(family, src, dst, size)`** — **numeric → presentation for BOTH IPv4 and IPv6**, into a caller-supplied buffer. Thread-safe and preferred.

**Recommendation (modern):** use `inet_pton`/`inet_ntop` — they handle IPv4 *and* IPv6, are thread-safe, and give clean error codes. `inet_aton` is fine for IPv4-only code; avoid `inet_addr` and prefer not to use `inet_ntoa`.

---
**Marking scheme (6 marks):** purpose (two-way conversion) = **2**, each function's job = **2**, modern recommendation/limits = **2**.

---

### Q14 🔴★ Write the TCP server & client system-call sequence. (asked multiple times)

**Server (passive open):**
```
socket() → bind() → listen() → accept() → read()/write() → close()
```
**Client (active open):**
```
socket() → connect() → read()/write() → close()     (bind optional)
```

**What each call does:**
- `socket(family,type,proto)` — create the endpoint.
- `bind(sock, &addr, len)` — **server**: attach a specific IP/port to the socket (client *may* skip this).
- `listen(sock, backlog)` — **server only**: mark as passive, allow incoming connections (backlog = pending queue size).
- `accept(sock, &cli, &len)` — **server only**: block until a client connects, then return a **new** connected socket for that client.
- `connect(sock, &addr, len)` — **client only**: initiate the connection.
- `read()/write()` or `send()/recv()` — exchange data.
- `close(sock)` — release the socket; also `shutdown()` for a graceful half-close.

```
SERVER                              CLIENT
socket()  ───────── course of time ─► socket()
bind()  ────────── (client may also bind a specific source port) ─►
listen()
accept() ◀── 3-way handshake ─────── connect()
         ◀── data: read()/write()──────▶
close()                                 close()
```

**Gotchas:** the server blocks in `accept` until a client arrives; `accept` returns a *new* fd (the listener is kept for more connections); a real server loops `accept → fork/thread` to serve many clients.

---
**Marking scheme (8 marks):** correct server sequence = **2**, client sequence = **2**, diagram with both sides = **2**, one-line job of each call = **2**.

---

### Q15 🔴 What happens if you call bind() in a TCP client? (asked: NCIT 2025 Q3b)
A TCP client normally does **NOT** call `bind()`. When the client calls `connect()`, the kernel automatically assigns a free **ephemeral port** and uses the local machine's address as the source — this is called an **implicit bind**.

**If you DO call `bind()` in a client:**
- You force a **specific local IP/port** as the source instead of the automatic ephemeral one.
- If that port is **already in use**, `bind()` fails with **`EADDRINUSE`**.
- Binds to a specific source IP so traffic leaves via a **particular interface**.
- It is only needed in special cases, e.g. an **FTP active mode** client that must report to the server: "connect back to me on port 20000" — the client must bind to 20000 first.

**Also asked (send/recv in UDP, sendto/recvfrom in TCP):**
- **TCP** normally uses `write()`/`read()` or `send()`/`recv()` — the connection already identifies both ends, so no per-datagram address is needed.
- **UDP** uses `sendto()`/`recvfrom()` because each datagram may have a **different peer** — you must specify (or learn) the peer address on *every* datagram.
- Technically you *could* use `sendto`/`recvfrom` on a connected TCP socket (the address is ignored), but it is unnecessary and confusing.

---
**Marking scheme (6 marks):** why client skips bind (ephemeral) = **2**, consequences of explicit bind + EADDRINUSE = **2**, TCP-vs-UDP send functions = **2**.

---

### Q16 🟡★ What is a daemon? How do you daemonize a process in UNIX? (asked: NCIT 2025 Q4a)
A **daemon** is a **long-running background process** that:
- has **no controlling terminal** (cannot read from keyboard),
- usually **runs with special privileges** (may need to bind to well-known ports),
- **starts at boot / login** and runs until shutdown,
- writes output to a **log file or syslog** (not the screen).

Examples: `sshd`, `httpd`, `syslogd`, `crond`.

**To daemonize (standard steps + code):**
```c
fork();            /* 1. parent exits, child becomes the daemon  */
setsid();          /* 2. new session, detach from controlling terminal */
chdir("/");        /* 3. safe working directory (don't lock a mounted fs) */
umask(0);          /* 4. clear file-mode mask so files aren't too restrictive */
/* 5. redirect stdin/stdout/stderr to /dev/null */
```

**Why each step:**
1. **`fork()`** — so the child is not a session leader (required for `setsid` to work); parent exits immediately so the shell prompt returns.
2. **`setsid()`** — creates a **new session**; the process has no controlling terminal. If this succeeds, the process is a **session leader** with no tty.
3. **`chdir("/")`** — avoid keeping a busy/locked filesystem as the current directory.
4. **`umask(0)`** — ensures the daemon can create files with full intended permissions.
5. **Redirection** — `open("/dev/null"); dup2(0,1); dup2(0,2);` sends stdin/stdout/stderr to the null device so no terminal interaction is possible and no stray output appears.

**Optional but common:** a **second `fork()`** (double-fork) so the daemon is *not* a session leader and can never re-acquire a controlling terminal.

```c
if (fork() > 0) exit(0);   /* 1 */
setsid();                  /* 2 */
chdir("/");                /* 3 */
umask(0);                  /* 4 */
open("/dev/null"); dup2(0,1); dup2(0,2);  /* 5 */
```

---
**Marking scheme (7 marks):** daemon definition = **2**, code/5 steps = **3** (1 each), why-each-step = **2**.

---

### Q17 🟡★ signal() and sigaction(), and signal handling in UNIX. (asked: Gandaki 2025 Q3a)
**Signals** are **asynchronous notifications** that tell a process an event happened (SIGINT = Ctrl-C, SIGIO, SIGCHLD, SIGTERM, SIGHUP). A process can **ignore, default-handle, or catch** a signal with a handler function `void handler(int)`.

**`signal(signum, handler)`** — simple but **not portable/robust**:
- Behavior differs across UNIX variants.
- It may **reset the handler to default** after catching once (BSD fixes this).
- Does not let you **block other signals** during handling.

**`sigaction(signum, &act, &old)`** — the **modern, powerful, recommended** form:
```c
struct sigaction act;
act.sa_handler = my_handler;      /* or sa_sigaction for extra args */
sigemptyset(&act.sa_mask);        /* which signals to block during handler */
act.sa_flags = SA_RESTART;        /* auto-restart interrupted syscalls */
sigaction(SIGIO, &act, NULL);
```
Advantages over `signal`:
- **`sa_flags`** — e.g. `SA_RESTART` auto-restarts a syscall interrupted by the signal (avoids `EINTR` errors).
- **`sa_mask`** — block other signals **while this handler runs** (prevents re-entrant races).
- **Portable** behaviour across all UNIX systems.

**Why it matters in network programming:**
- **`SIGIO`** — signals "socket is ready" → basis of **signal-driven I/O**.
- **`SIGCHLD`** — sent to parent when a child exits → used to **reap zombies** in a fork-based server (`waitpid` in the handler).
- **`EINTR`** — an interrupted `accept`/`read` returns -1 with `errno == EINTR`; either loop-and-retry or use `SA_RESTART`.

---
**Marking scheme (7 marks):** what signals are = **1**, signal() form = **1**, sigaction() form = **2**, why sigaction better = **2**, network-programming uses = **1**.

---

### Q18 🟡★ fork() and exec() — creating processes. (asked: NCIT 2025 Q4a)

**`fork()`** — creates a new **child process** that is an **exact copy** of the parent:
- Child gets its **own PID**; both run the *same* code from the point of `fork` onwards.
- Return values: **0** in the child, **child's PID** in the parent, **-1** on failure.
- Used after `accept()` so **each client gets its own server process**.

```c
pid_t pid = fork();
if (pid == 0) {           /* child */
    handle_client(connfd);
    exit(0);
} else if (pid > 0) {     /* parent */
    close(connfd);        /* parent doesn't need the per-client socket */
}
```

**`exec()`** — **replaces the current process image** with an entirely new program and runs it from its entry point:
- On success it **never returns**; the PID stays the same, the code/data/stack are new.
- Variants: `execl`, `execlp`, `execle`, `execv`, `execvp`, `execve` — differ in how args and the environment are passed.

**How they combine (classic pattern):**
```c
fork();   /* create a child that is a copy of parent */
exec();   /* in the child, replace with a new program (e.g. /bin/ls) */
```
- **fork + exec together = run a *different* program.** `fork` gives you the *process*, `exec` gives you the *program*.
- In a network server: `fork()` after `accept` to service many clients in parallel; a client may then `exec()` a shell to let the user issue commands (`telnet`-style).

**Zombies:** if a child exits and the parent doesn't `wait()`/`waitpid()` on it, it stays as a **zombie** (dead but still in the process table). A server must handle `SIGCHLD` and `waitpid` to **reap** children, else it leaks resources.

---
**Marking scheme (6–7 marks):** fork() = **2**, exec() = **2**, fork+exec pattern and use in servers = **2**, zombies = **1**.

---

### Q19 🟢★ UNIX domain sockets & socketpair. (asked: short note)
**UNIX domain sockets (`AF_UNIX` / `AF_LOCAL`)** are the network-programming model adapted for **same-host IPC** — two processes on the *same* machine communicate using socket API, but **no network stack / IP** is involved (addresses are **pathnames** in the filesystem, e.g. `"/tmp/foo.sock"`).

**Features:**
- **Faster** than TCP/IP on the same host (no protocol headers, no routing, kernel-internal copy).
- Supports **stream (`SOCK_STREAM`)**, **datagram (`SOCK_DGRAM`)**, and **sequenced-packet** forms (the last preserves message boundaries and is reliable).
- Can **pass file descriptors** between processes via `sendmsg()`/`recvmsg()` (with a `SCM_RIGHTS` control message) — the receiver gets a *new* fd referring to the same open file. This is used by `X Window`, systemd, and daemons.
- Used widely: `X11`, `PostgreSQL` (local connections), `systemd`, Docker socket.

**`socketpair()`** — creates **two connected sockets at once**, already linked to each other:
```c
int fds[2];
socketpair(AF_UNIX, SOCK_STREAM, 0, fds);
/* fds[0] and fds[1] are connected; write on fds[0], read on fds[1] */
```
- Gives a **bidirectional** pipe (like an fd-pair but connection-oriented) — handy for **parent↔child** IPC without a filesystem path.
- Faster/cleaner than a named socket for two related processes.

**vs TCP loopback:** don't use TCP (`AF_INET` 127.0.0.1) for same-host IPC when you can use a Unix socket — the latter avoids port conflicts, routing, and firewall overhead.

---
**Marking scheme (5–6 marks):** what it is (pathname, same-host) = **2**, stream/datagram forms = **1**, fd passing = **1**, socketpair = **1–2**.

---

### Q19b 🟡 Hostname & service name resolution (gethostbyname / getservbyname / getaddrinfo).
Computers communicate with **IP addresses and port numbers**, but humans (and programs) use **names** (hostnames, service names). **Name resolution** converts between them.

**`gethostbyname(name)`** → returns `struct hostent *` for a hostname:
```c
struct hostent {
    char *h_name;            /* official host name */
    char **h_aliases;        /* alternate names */
    int   h_addrtype;        /* AF_INET (IPv4) / AF_INET6 */
    int   h_length;          /* length of address in bytes */
    char **h_addr_list;      /* list of IP addresses (network byte order) */
};
```
- Reads **DNS** to find addresses.
- **Limits:** blocks (synchronous DNS), **IPv4 only** (no IPv6), and returns a pointer to **static data** → **not thread-safe**. Deprecated.

**`gethostbyaddr(addr, len, type)`** → the **reverse**: IP address → official hostname (a DNS PTR lookup).

**`getservbyname(name, proto)`** → returns `struct servent *` giving the **port** for a well-known service:
```c
struct servent *s = getservbyname("http", "tcp");
/* s->s_port = 80 (network byte order) */
```
Reads the system **`services` database** (`/etc/services`).
**`getservbyport(port, proto)`** → the **reverse**: port number → service name.

**Modern alternative — `getaddrinfo()` (preferred):**
- One function gets name + service and returns a **linked list of `struct addrinfo`**, each already filled with a usable `sockaddr` for **IPv4 or IPv6** (`AF_UNSPEC` lets the system pick).
- **Thread-safe**, family-neutral, and is the canonical new API (used by the `getaddrinfo` examples in Stevens).

```c
struct addrinfo hints, *res;
memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;            /* IPv4 or IPv6 */
hints.ai_socktype = SOCK_STREAM;        /* TCP */
getaddrinfo("www.example.com", "http", &hints, &res);
/* walk res; each node has res->ai_addr, res->ai_addrlen */
freeaddrinfo(res);
```

**Which to use:** for new code always `getaddrinfo` (IPv6-ready, thread-safe). Know `gethostbyname` because it is still in old textbooks/exams, but state its limits.

---
**Marking scheme (7 marks):** what resolution is = **1**, gethostbyname + hostent = **2**, getservbyname = **1**, getaddrinfo = **2**, limits/recommendation = **1**.

---
## Unit 3 — Advanced Unix

### Q20 🔴★ The five I/O models; which are synchronous? (asked: Gandaki 2025 Q4a)
Every network read involves **two phases**:
1. **Wait for data to be ready** — the kernel waits for a packet to arrive (from the network card).
2. **Copy data from kernel to process** — once ready, the kernel copies data into the user's buffer (never instant, even with zero-copy tricks).

The five I/O models differ in *what the process does* during these two phases:

| # | Model | Phase 1 (wait) | Phase 2 (copy) | Blocking? |
|---|---|---|---|---|
| 1 | **Blocking I/O** | Process **sleeps** (blocks in `recvfrom`) | Process **sleeps** while kernel copies | Yes — blocks on recvfrom |
| 2 | **Non-blocking I/O** | Process **polls** (`EWOULDBLOCK` until ready) | Kernel copies once ready | Yes — poll+recvfrom both block/sleep |
| 3 | **I/O multiplexing** (`select`/`poll`) | Process **blocks in select()** until one fd is ready | Process does `recvfrom` (kernel copies) | Yes — blocks in select, then blocks in recvfrom |
| 4 | **Signal-driven I/O** (`SIGIO`) | Kernel **sends SIGIO** when fd ready (process runs handler) | Process does `recvfrom` (kernel copies) | Yes — recvfrom blocks |
| 5 | **Asynchronous I/O** (`aio_read`, POSIX AIO) | Kernel waits for data | Kernel **copies + notifies process** (via signal/callback) | **No** — fully async |

**Which are synchronous?**
A POSIX "synchronous" I/O operation is one that **blocks the process until the operation itself completes**. Models 1–4 are all **synchronous**: the actual `recvfrom()` call **blocks** until data is copied. Only model **5 (asynchronous I/O)** is truly asynchronous — the process is never blocked waiting for data, because the *kernel does everything* and notifies you later.

**Which to use for many clients:** models 3 or 4 are the standard server models. Model 5 (POSIX AIO) exists on paper but the real-world server winner is **model 3 (select/poll/epoll)** because it is portable, mature, and allows multiplexing thousands of sockets.

**Diagram (the single most valuable thing to draw):**
```
         Phase 1              Phase 2
         ─────────            ─────────
Model 1: [BLOCK: sleep]──────[BLOCK: kernel copies] ──▶ return
Model 2: [poll: EWOULDBLOCK]─[BLOCK: kernel copies] ──▶ return
Model 3: [BLOCK: select()]───[BLOCK: kernel copies] ──▶ return
Model 4: [SIGIO handler runs][BLOCK: kernel copies] ──▶ return
Model 5: [Kernel does everything]───────────────────return (never blocks user)
```

---
**Marking scheme (8 marks):** diagram = **3**, table of 5 = **2**, sync vs async explanation = **2**, one example each = **1**.

---

### Q21 🟡★ Blocking vs Non-blocking I/O. (asked: Gandaki 2025 Q4a)

**Blocking I/O (`SOCK_STREAM` default):**
- The `recvfrom()` call **sleeps** until data arrives AND is copied into the user buffer. The process does nothing — it simply stops until the kernel has data ready.
- **Advantage:** simplest code, no polling logic.
- **Disadvantage:** **ties up the thread entirely** — one thread can only handle *one* client at a time. A server using blocking I/O needs a thread per client (or `fork`) — scales poorly.
- Example: `read(fd, buf, n)` on a blocking socket returns `n` bytes read, or 0 if the peer closed, or -1 on error. Never returns with "no data yet" — that can't happen.

**Non-blocking I/O (`O_NONBLOCK` or `FIONBIO`):**
- Set the socket non-blocking: `fcntl(fd, F_SETFL, O_NONBLOCK)`.
- Now `recvfrom()` **returns immediately** — if data is not ready, it returns **-1** with `errno = EWOULDBLOCK` (or `EAGAIN`). If data *is* ready, it copies and returns the count.
- **Advantage:** the process can work on other things between polls.
- **Disadvantage:** you must **poll repeatedly** (busy-wait) until data arrives → **wastes CPU**. A server polling thousands of sockets in a tight loop burns 100% CPU doing nothing useful.

**The solution — combine with `select()`:**
The real reason non-blocking sockets exist is **not** to poll manually — it is to use them **with `select`/`poll`**. `select` blocks efficiently until *one or more* sockets are ready, then you do non-blocking reads on the ready ones. This gives you the best of both worlds:
- `select` tells you which sockets are ready (efficient, no busy-wait).
- Non-blocking read on each ready socket returns immediately with data (or an unexpected `EWOULDBLOCK` if you got the readiness wrong — a "spurious readiness").

**Summary table:**

| | Blocking | Non-blocking |
|---|---|---|
| `recv` when no data | Blocks (sleeps) | Returns `EWOULDBLOCK` |
| CPU usage | Low (sleep) | High (polling) if busy-waiting |
| Thread per client? | Required | Not required |
| Use alone? | Simple, one client | Poor (wastes CPU) |
| Use with select? | Not needed | **Best combination** |

---
**Marking scheme (6 marks):** blocking explanation = **2**, non-blocking explanation = **2**, why-combine-with-select = **2**.

---

### Q22 🔴★ Explain signal-driven I/O, compare with I/O multiplexing. (asked: Gandaki 2025 Q4a, NCIT 2025 alternative)

**Signal-driven I/O:**
- The process enables the socket for `SIGIO`, installs a signal handler using `sigaction`, and goes about its business.
- When a **datagram arrives** (data is now ready in the kernel), the kernel sends **SIGIO** to the process. The **signal handler** runs `recvfrom()` to read the data.
- This is like an **interrupt model** — the kernel *tells you* when I/O is ready rather than you asking.
- Real implementation (Linux): uses `F_SETOWN` + `F_SETFL` with `O_ASYNC`:
  ```c
  int flags = 1;
  ioctl(fd, FIOASYNC, &flags);        /* enable async */
  fcntl(fd, F_SETOWN, getpid());      /* deliver SIGIO to this process */
  signal(SIGIO, my_handler);
  ```

**I/O Multiplexing (select/poll):**
- The process calls `select()` or `poll()` with a set of file descriptors. It **blocks** inside `select` until one or more fds are ready.
- Then the process does `recvfrom` on each ready fd (this `recv` returns immediately since data is there).
- No signals involved; it is purely a **blocking loop**:
  ```c
  for (;;) {
      FD_SET(...);
      select(maxfd+1, &readset, NULL, NULL, NULL);  /* blocks */
      for (fd = 0; fd < maxfd+1; fd++)
          if (FD_ISSET(fd, &readset)) read(fd, buf, n);
  }
  ```

**Head-to-head comparison:**

| Feature | Signal-driven I/O | I/O Multiplexing (select) |
|---|---|---|
| Notification mechanism | **Kernel sends SIGIO** | **select() returns** ready set |
| How process learns readiness | Signal handler runs | select() unblocks |
| Does the process block? | **No** (handler fires asynchronously) | **Yes** (blocks in select) |
| Which fds are ready? | **Only the one that triggered SIGIO** (need `F_GETOWN` to find out) | **A bit-field** showing all ready fds at once |
| Multi-fd handling | One signal per fd; many signals may arrive together → missed events | One select returns all ready fds in one shot |
| Best for | Many low-rate fds (few events/sec) | High-throughput servers, many fds |

**Why `select` is generally preferred:**
- With signal-driven I/O, if many signals fire simultaneously, the kernel may **merge/miss** them (signals are not queued on standard UNIX) — you don't know which fd was actually ready.
- `select()` clearly returns a **set** of all ready fds at once — much more reliable.
- However, signal-driven I/O has **lower latency** when data arrives and the process is sleeping — you don't have to be blocked in `select` to learn about it.

**Both are synchronous:** in both cases, the actual `recvfrom()` call in the handler or after select **blocks** (synchronously copies data). Neither is asynchronous (model 5).

---
**Marking scheme (8 marks):** signal-driven explanation = **2**, select explanation = **2**, comparison table = **2**, which preferred/why = **1**, both-synchronous note = **1**.

---

### Q23 🔴★ I/O multiplexing & select(). (asked: NCIT 2025 Q4b)
**I/O multiplexing** means one process can watch **many** file descriptors (sockets, stdin, etc.) simultaneously and be told which ones are ready for reading/writing, using a **single blocking call**.

**`select()` prototype:**
```c
int select(int maxfdp1, fd_set *readset, fd_set *writeset,
           fd_set *exceptset, const struct timeval *timeout);
// returns: number of ready fds, 0 on timeout, -1 on error
```

**Parameters explained:**
- `maxfdp1` — one more than the **highest-numbered** fd you are watching (e.g. if fds are 0, 3, 5 → pass 6).
- `readset` — set of fds to watch for **read readiness** (data available, peer closed, or listening socket with pending connection).
- `writeset` — set of fds to watch for **write readiness** (buffer space available to write without blocking).
- `exceptset` — set of fds to watch for **exceptional conditions** (e.g. out-of-band TCP data).
- `timeout` — how long to wait: `NULL` = wait forever; `tv.tv_sec=5, tv.tv_usec=0` = wait 5 seconds; zero = poll (return immediately).

**Macros (all operate on `fd_set`):**
```c
FD_ZERO(&set);          /* clear the set */
FD_SET(fd, &set);       /* add fd to the set */
FD_CLR(fd, &set);       /* remove fd from the set */
FD_ISSET(fd, &set);     /* is fd in the set? (after select returns) */
```

**Typical pattern:**
```c
fd_set readset;
int maxfd = listenfd;
for (;;) {
    FD_ZERO(&readset);
    FD_SET(listenfd, &readset);
    FD_SET(stdin_fd, &readset);
    select(maxfd + 1, &readset, NULL, NULL, NULL);

    if (FD_ISSET(listenfd, &readset)) {
        /* new connection arrived */
    }
    if (FD_ISSET(stdin_fd, &readset)) {
        /* user typed something */
    }
}
```

**Use cases:**
- A **single-threaded client** watching both **stdin** and its **socket** simultaneously (e.g. a telnet client).
- A **single-threaded server** handling many sockets in one thread (simple chat server).
- Combine with non-blocking sockets: do non-blocking reads on ready fds to avoid a single read blocking the whole select.

**Limits of select (and why `poll`/`epoll` exist):**
- `fd_set` has a fixed maximum size (typically `FD_SETSIZE = 1024`).
- It must be **re-initialized** before every `select` call (select overwrites it).
- `maxfdp1` can be no larger than `FD_SETSIZE`.
- For very large numbers of sockets, `poll` or Linux `epoll` is more scalable.

---
**Marking scheme (8 marks):** what multiplexing is = **1**, select prototype + params = **2**, macros = **1**, code pattern = **2**, use cases + limits = **2**.

---

### Q24 🔴★ Mechanisms to handle multiple clients in UNIX — with code. (asked: Gandaki 2025 Q3b, NCIT 2025 Q5a)
A server must handle **many clients simultaneously**. There are three standard approaches, each with a different concurrency model:

**Approach 1 — `fork()` per client (process-per-connection):**
- After `accept()`, the server `fork()`s a child process for each client. The child handles the client; the parent goes back to `accept()`.
- **Advantage:** simple, each client is isolated (one crash doesn't kill the server).
- **Disadvantage:** one process per client uses lots of memory, does not scale well to thousands of clients.
- Must **reap zombies** (`waitpid` in a `SIGCHLD` handler).

```c
for (;;) {
    clilen = sizeof(cliaddr);
    connfd = accept(listenfd, (SA*)&cliaddr, &clilen);
    if ((pid = fork()) == 0) {       /* child */
        close(listenfd);             /* child doesn't need the listener */
        doit(connfd);                /* service this one client */
        close(connfd);
        exit(0);
    }
    close(connfd);                   /* parent closes the connected socket */
}
```

**Approach 2 — `select()`/`poll` multiplexing (single-threaded, event-driven):**
- One process/thread watches *all* sockets with `select()` or `poll()`. When data arrives on any socket, read it. No fork, no threads.
- **Advantage:** low memory (one process), good for thousands of idle connections.
- **Disadvantage:** complex code, one slow handler blocks everything.
- **Add non-blocking sockets** so a read on a ready socket never blocks unexpectedly.

**Approach 3 — threads (`pthread_create`) per client:**
- Like fork, but lighter-weight: threads share the same address space (no `fork`/`exec` overhead, no zombie issues).
- **Advantage:** lighter than processes, good concurrency.
- **Disadvantage:** shared memory → **race conditions**; need mutexes/locks. A thread crash can kill all threads.

**Comparison:**

| Mechanism | Pros | Cons | Best for |
|---|---|---|---|
| fork() | Isolation, simple | Slow, zombie issues, 1-conn/memory | Small servers |
| select() | Scalable, low memory | Complex, single-point-of-failure | Thousands of clients (chat, proxy) |
| pthreads | Lightweight, shared memory | Race conditions, need locks | Moderate concurrency, shared data |

**Code example (fork):** already shown above.

---
**Marking scheme (8 marks):** 3 approaches explained = **3**, code example = **2**, comparison table = **2**, zombies note = **1**.

---

### Q25 🟡★ Broadcast vs multicast. (asked: NCIT 2025 Q4a)
Both are one-to-many communication methods (send one packet, many hosts receive it), but differ in *scope* and *efficiency*:

**Broadcast (UDP only):**
- Sends a datagram to **every host on the local subnet** (e.g. `255.255.255.255` = limited broadcast).
- Must set `setsockopt(s, SOL_SOCKET, SO_BROADCAST, &on, sizeof(on))` — the socket must explicitly enable broadcast.
- **Routers do NOT forward** broadcast packets → it is **local-subnet only**.
- **Wastes resources:** every host on the subnet must process the packet even if it is not interested (CPU overhead, interrupt storms).
- Use case: **network-wide discovery/announce** (e.g. finding a printer on the LAN, DHCP discovery).

**Multicast:**
- Sends to a **group of hosts** that have "subscribed" (joined) to a multicast group address.
- **Group address:** Class D IP range `224.0.0.0` to `239.255.255.255`. A multicast address is like a "virtual club" — you join it, you receive its packets.
- Hosts join a group by calling `setsockopt(s, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, ...)`. They can leave with `IP_DROP_MEMBERSHIP`.
- `IP_MULTICAST_TTL` controls how far packets go (how many routers forward them).
- **NIC hardware filtering:** network cards are programmed to receive only packets for their multicast group, so non-members never see the packet — **far more efficient** than broadcast.
- Use case: **video conferencing**, streaming, live stock tickers, multiplayer games — any scenario where only *interested* hosts should receive the data.

**Summary:**
| Feature | Broadcast | Multicast |
|---|---|---|
| Scope | All hosts on subnet | Only interested hosts |
| Efficiency | Poor (all hosts process) | Better (NIC filters, group-only) |
| Setup | `SO_BROADCAST` | `IP_ADD_MEMBERSHIP` |
| Routing | Routers don't forward | Routers forward (with PIM) |
| Typical use | LAN discovery, DHCP | Video conferencing, live streaming |

---
**Marking scheme (6 marks):** broadcast definition + setup = **2**, multicast definition + setup = **2**, comparison table = **2**.

---

### Q26 🟡★ Socket options: setsockopt / getsockopt + SO_REUSEADDR, SO_BROADCAST, SO_KEEPALIVE, SO_LINGER. (asked: NCIT 2025 Q5b, Gandaki 2025 Q4b)

**`setsockopt` / `getsockopt`** get and set socket options at various protocol levels:

```c
int setsockopt(int s, int level, int optname, const void *optval, socklen_t optlen);
int getsockopt(int s, int level, int optname, void *optval, socklen_t *optlen);
```
- `level` — protocol layer: `SOL_SOCKET` (general), `IPPROTO_TCP`, `IPPROTO_IP`, `IPPROTO_IPV6`.
- `optname` — the specific option to get/set.
- `optval` — a pointer to the value (typically an `int`).

**The four key options:**

**1. `SO_REUSEADDR` — reuse a local address/port:**
- **Problem:** after a server closes, its address enters `TIME_WAIT` (2×MSL, ~2 minutes). If you restart immediately, `bind()` fails with `EADDRINUSE` — the port is still "in use" by old orphaned packets.
- `SO_REUSEADDR` allows binding to an address even if it is in TIME_WAIT. It is **essential for servers** that restart frequently (HTTP servers, database servers).
- Example: `setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));`

**2. `SO_BROADCAST` — allow sending broadcast messages:**
- **Problem:** by default, a socket **rejects** broadcast sends (`EACCES`).
- Setting `SO_BROADCAST` allows `sendto()` to a broadcast address (e.g. `255.255.255.255`).
- Used with UDP only; routers do not forward broadcast (local-subnet only).
- Required before sending a broadcast; else you get `EACCES` on `sendto`.

**3. `SO_KEEPALIVE` — TCP keepalive probes:**
- After **2 hours of inactivity** (no data sent), the OS sends a TCP keepalive **probe** to check if the peer is still alive.
- If the peer responds with ACK → connection alive, reset the timer, wait another 2 hours.
- If the peer has crashed → RST is received → `ECONNRESET`.
- If no response → up to **8 probes**, **75 seconds apart** (~10 minutes) → `ETIMEDOUT` (or `EHOSTUNREACH` if ICMP unreachable).
- Useful for long-lived connections (SSH, database, VPN) to detect dead peers and free resources.
- Timers are often tunable via system-wide `sysctl`.

**4. `SO_LINGER` — control close() behaviour (see Q27 for full detail):**
- Controls what `close()` does with unsent data.
- Three modes: default (immediate return), abort (RST, discard data), linger (wait up to `l_linger` seconds for data to be acked).

---
**Marking scheme (8 marks):** setsockopt/getsockopt = **2**, SO_REUSEADDR = **2**, SO_BROADCAST = **1**, SO_KEEPALIVE = **2**, SO_LINGER = **1**.

---

### Q27 🟡 SO_LINGER in detail.
`SO_LINGER` controls what happens to **unsent data** when a TCP socket is `close()`d. It is configured using:
```c
struct linger {
    int l_onoff;    /* 0 = off (default), non-zero = on */
    int l_linger;   /* seconds to wait (0 = immediate) */
};
```

**Three modes:**

| `l_onoff` | `l_linger` | Behaviour | Data |
|---|---|---|---|
| **0** (off) | any | `close()` **returns immediately**; OS sends data in background, silently discards if connection fails | Sent when possible |
| non-zero | **0** | `close()` returns immediately; TCP sends **RST** (abort), all **unsent data is discarded** | Lost |
| non-zero | **> 0** | `close()` **blocks** up to `l_linger` seconds, waiting for sent data to be acknowledged; if timeout expires before ACK, the connection is **aborted (RST)** and `close()` returns | Best-effort delivery |

**When to use each:**
1. **Default (on=0):** everyday programming — you don't want `close()` to block. The OS handles delivery in the background.
2. **Linger with timeout (on=1, linger=5):** you **must** ensure data is delivered before the process exits — e.g. a database flush, a financial transaction. You are willing to wait 5 seconds for the peer to ACK.
3. **Abort (on=1, linger=0):** you want to **terminate immediately** and discard unsent data — e.g. cancelling an operation, sending an error to the peer via RST.

**Common error:** setting linger without understanding it → calling `close()` on a socket with unsent data and long linger time can **block the process** for several seconds, which is catastrophic in a high-performance server.

**Note on TCP RST:** when the linger timeout expires (or linger=0), the OS sends a TCP **RST** (reset) segment. This tells the peer to abort immediately. The peer sees `ECONNRESET` on its next read/write.

---
**Marking scheme (5–6 marks):** struct definition = **1**, three modes = **3**, when to use each = **1–2**.

---

### Q28 🟢 SO_KEEPALIVE in detail.
`SO_KEEPALIVE` enables **TCP keepalive probes** — periodic "are you still alive?" messages sent on idle connections.

**Default timers (Linux):**
1. After **2 hours** of inactivity → send first keepalive **probe**.
2. Peer ACK → connection alive, **reset 2-hour timer**.
3. No ACK → wait **75 seconds**, send another probe. Repeat up to **8 times**.
4. After 8 failed probes (~10 minutes total) → report **`ETIMEDOUT`** to the application, or **`EHOSTUNREACH`** if an ICMP unreachable was received.

**What the peer sees:**
- The peer sees a **regular TCP segment** with `ACK` set and the **same sequence number** as the last data (a zero-length segment). If the peer is alive, it ACKs immediately.
- If the peer has **crashed and rebooted**: it replies with `RST` → `ECONNRESET`.
- If the peer is **gone** (machine powered off): no response → 8 probes → timeout.

**Why it matters:**
- Detects **dead connections** (zombie connections left open).
- Frees server resources (memory, fd, port) faster.
- Prevents TCP **half-open connections** that waste resources.
- Used by **SSH**, **databases**, **VPNs**, **long-lived HTTP keep-alive**.

**System tuning (Linux):**
```bash
sysctl -w net.ipv4.tcp_keepalive_time=60    # first probe after 60s (not 2h)
sysctl -w net.ipv4.tcp_keepalive_intvl=30   # probe every 30s
sysctl -w net.ipv4.tcp_keepalive_probes=5   # 5 probes before giving up
```
These system-wide settings apply to all sockets with `SO_KEEPALIVE` enabled.

**Warning:** `SO_KEEPALIVE` adds a tiny overhead (one extra byte per probe) but can cause problems with NAT devices or firewalls that time out idle connections before the TCP keepalive timer fires — use shorter app-level heartbeats (ping/pong) in those cases.

---
**Marking scheme (4–5 marks):** what it is = **1**, timers = **1**, what peer sees = **1**, why-useful = **1**, tuning (bonus) = **1**.

---

### Q29 🟡★ Syslog & logging from network applications. (asked: NCIT 2025 Q4b — with block diagram)
**Syslog** is a **centralised logging facility** on UNIX/Linux that lets daemons and network applications report errors, events, and status messages **without needing a terminal** (since daemons have no controlling terminal).

**Architecture (block diagram — draw this):**
```
     ┌─────────────────────────────────────────┐
     │            Clients (daemons)             │
     │  openlog("sshd", LOG_PID, LOG_AUTH)      │
     │  syslog(LOG_WARNING, "bad login from %s") │
     └────────────────────┬────────────────────┘
                          │  (socket: UDP 514 or /dev/log)
                          ▼
              ┌──────────────────────┐
              │     syslogd daemon   │
              │  (central logging)   │
              ├──────────────────────┤
              │  reads /etc/syslog.conf │
              │  classifies by facility + priority │
              │  writes to log files │
              └──────┬──────┬────────┘
                     │      │
              /var/log/auth.log
              /var/log/syslog
              /var/log/messages
              /dev/console (emergency)
```

**Functions:**
- `openlog(ident, options, facility)` — open the connection to syslog; `ident` is the program name, `facility` is the category (LOG_AUTH, LOG_DAEMON, LOG_LOCAL0-7, etc.).
- `syslog(priority, format, ...)` — log a message; `priority` combines `facility | level` (LOG_ERR, LOG_WARNING, LOG_INFO, etc.).
- `closelog()` — close the connection.

**Priorities (from most to least severe):**
`LOG_EMERG` > `LOG_ALERT` > `LOG_CRIT` > `LOG_ERR` > `LOG_WARNING` > `LOG_NOTICE` > `LOG_INFO` > `LOG_DEBUG`

**How it helps network applications:**
- A daemon cannot write to `stdout`/`stderr` (no terminal) — syslog is its **output channel**.
- `/etc/syslog.conf` controls where messages go (which file, which host) — no code change needed to redirect logs.
- Centralised → one daemon collects logs from all processes; logs are **timestamped**, **facility-tagged**, and can be **rotated**.

**Network option:** a client can send syslog messages to a **remote server** (UDP port 514) — so multiple machines share one log server, which is invaluable for distributed systems.

---
**Marking scheme (7 marks):** block diagram = **3**, openlog/syslog/closelog = **2**, priorities/facilities = **1**, why-network-daemons-need-syslog = **1**.

---

### Q30 🟡★ How to secure a network application. (asked: short note "Wrapper function…")
Securing a network application requires **layers** of protection — access control, authentication, and encryption. The question specifically asks about **by hostname, by IP number, and by wrapper program**:

**1. By hostname/domain:**
- Allow connections only from trusted **hostnames** — resolve the client's IP to a hostname using `gethostbyaddr()`, then check against an allow-list.
- **Limitation:** DNS can be **spoofed** (attacker forges a DNS reply to match a trusted hostname) — this alone is weak. Use it as part of a layered defence, not as the only control.

**2. By IP number:**
- Restrict connections by source IP address using **`/etc/hosts.allow` + `/etc/etc.deny`** (TCP wrappers) or **firewall rules** (`iptables`, `nftables`, `pf`).
- Simpler and stronger than hostname-only — IP spoofing is harder than DNS spoofing.
- But: IPs can still be spoofed; a VPN or NAT makes IP-based control less reliable.

**3. Wrapper program (TCP wrappers concept):**
- A **wrapper program** is a small front-end that intercepts the incoming connection *before* the real service starts.
- It checks the client against a policy (hosts.allow/deny, custom rules) and **only if allowed**, it launches the real service and relays the connection to it.
- This implements access control **without modifying the server application itself** — the server binary is unchanged; the wrapper does the gating.
- Classic example: `inetd` + TCP wrappers (`in.tcpd`) — `inetd` accepts the connection, runs `tcpd`, `tcpd` checks allow/deny, then runs the real `telnetd`, `ftpd`, etc.
- **Modern equivalent:** systemd socket activation + firewall rules.

**4. Encryption — TLS/SSL:**
- **Encrypts** all data in transit (AES, ChaCha20 — symmetric ciphers).
- **Authenticates** the server via **certificates** signed by a trusted CA.
- **Detects tampering** via message authentication codes (HMAC).
- Without TLS, access control only restricts *who* can connect — data is still in the clear for eavesdropping and modification.

**Best practice:** combine all layers — firewall/WTCP wrappers for access control + TLS for confidentiality/integrity. Access control gates connections; TLS protects data in transit.

---
**Marking scheme (6–8 marks):** hostname = **2**, IP = **1**, wrapper program (concept + example) = **2**, TLS/SSL = **2**, best practice layering = **1**.

---## Units 4 & 5 — Winsock

### Q31 🔴★ How is Winsock different from UNIX sockets? + static vs dynamic linking. (asked: NCIT 2025 Q6a — 7 marks)

**Winsock = the Windows implementation of the BSD/Berkeley socket API.** It provides the same socket programming model but adapted for the Windows environment — its own setup, handle types, error handling, and extra I/O models.

**Comprehensive comparison table (memorise this):**

| Feature | Unix / Berkeley | Winsock |
|---|---|---|
| Socket type | `int` (file descriptor: 0, 1, 2, 3...) | `SOCKET` (opaque handle, not an fd) |
| Close a socket | `close(fd)` | `closesocket(s)` |
| Read/write | `read()`/`write()` (also `send`/`recv`) | `send()`/`recv()` only (no `read`/`write`) |
| Error reporting | global variable `errno` | `WSAGetLastError()` function |
| Setup before use? | **None needed** — `socket()` just works | **Must call `WSAStartup()` first**, and `WSACleanup()` when done |
| Address structure | `struct sockaddr_in` (same semantics) | `SOCKADDR_IN` (same semantics, different name) |
| Include header | `<sys/socket.h>`, `<netinet/in.h>` | `<winsock2.h>` + link `ws2_32.lib` |
| Async I/O models | `select`, `poll`, `epoll`, `kqueue` | **WSAAsyncSelect, WSAEventSelect, overlapped, IOCP** |
| Network init | no DLL loading | loads `ws2_32.dll` (Winsock DLL) |

**Why `WSAStartup()`? — DLLs (Dynamic Link Libraries):**
- Windows loads network protocol support as **DLLs** (shared code libraries). A program cannot call socket functions directly in the kernel; it must first load `ws2_32.dll`.
- `WSAStartup()` loads the DLL and negotiates a Winsock version (e.g. 2.2).
- Unix uses kernel system calls — the kernel always provides the network API, so no DLL loading is needed.

**Static vs Dynamic linking:**

| | Dynamic linking (DLL) | Static linking |
|---|---|---|
| Where is the code? | Separate `.dll` file, loaded at run time | Copied into the `.exe` at compile time |
| Executable size | **Smaller** (code not embedded) | **Larger** (code embedded) |
| Updating | **Easy** — replace the DLL, all programs pick up the fix | **Hard** — must recompile and redistribute the `.exe` |
| Dependency | Needs the DLL present, correct version ("**DLL hell**" if wrong version) | **No external dependency** — always runs |
| Code sharing | Multiple programs share one DLL | Each program has its own copy |

**Winsock example (showing the differences):**
```c
// WINSTOCK
#include <winsock2.h>          // must include winsock2.h
#pragma comment(lib, "ws2_32.lib")  // link to ws2_32.dll

WSADATA wsaData;
WSAStartup(MAKEWORD(2,2), &wsaData);   // MUST call before any socket function
SOCKET s = socket(AF_INET, SOCK_STREAM, 0);
// ... use socket ...
closesocket(s);     // not close()
WSACleanup();       // match every WSAStartup
```

---
**Marking scheme (7 marks):** comparison table = **3**, DLL concept + WSAStartup = **2**, static-vs-dynamic = **2**.

---

### Q32 🔴★ WSAStartup / WSACleanup; role of setup(), cleanup(). (asked: NCIT Q6b, Gandaki Q5a)

**`WSAStartup(MAKEWORD(2,2), &wsadata)`** — the **setup function** (called once before any socket function):
- `MAKEWORD(2,2)` requests Winsock **version 2.2**.
- The OS loads `ws2_32.dll` and negotiates the version.
- `wsadata` (a `WSADATA` struct) returns: the loaded version, the description, and status.
- **Must succeed** (return 0) — if it fails, no network functions will work.
- You can call it **multiple times** if needed (it is reference-counted).

**`WSACleanup()`** — the **cleanup function** (must match every `WSAStartup`):
- Decrements the Winsock **reference count**.
- When the count reaches 0, the DLL is **unloaded** and all Winsock resources are freed.
- Called **once** when the program is shutting down.
- If you forget it, resources leak (minor in practice since the OS cleans up at process exit).

**"setup() / cleanup()" in the exam paper = `WSAStartup()` and `WSACleanup()`.** They are the Windows equivalent of "no init needed" on Unix — Unix provides the socket API in the kernel at all times; Windows loads it on demand.

**Reference counting:** if two DLLs in the same process both call `WSAStartup`, the DLL is loaded once (count=2). Each `WSACleanup` decrements the count; when it reaches 0, the DLL is truly unloaded.

**Error on missing WSAStartup:** every Winsock call returns **`WSANOTINITIALISED`** if called before `WSAStartup` — a very common beginner mistake on Windows.

---
**Marking scheme (6–7 marks):** WSAStartup explained = **3**, WSACleanup explained = **2**, reference counting + error = **1–2**.

---

### Q33 🔴★ Major DLLs needed for a Winsock app. (asked: NCIT Q6b — 8 marks)
Windows organises the network stack as a hierarchy of **DLLs** (Dynamic Link Libraries). A Winsock application depends on several:

| DLL | Role | When is it loaded? |
|---|---|---|
| `ws2_32.dll` | **Main Winsock 2.0 API** — the primary DLL you link (`#pragma comment(lib, "ws2_32.lib")`) | Loaded by `WSAStartup` |
| `wsock32.dll` | Winsock 1.1 32-bit API (legacy, backward-compatible) | Loaded if old app calls WSA 1.1 functions |
| `winsock.dll` | Winsock 1.1 **16-bit** API (very old, Windows 3.1) | Only for 16-bit apps |
| `mswsock.dll` | Microsoft-specific extensions: `AcceptEx`, `TransmitFile`, `WSASendDisconnect`, etc. | On demand when extensions are used |
| `wshtcpip.dll` | **TCP/IP helper** — routines for TCP/IP-specific operations (e.g. `GetAddressByName`) | On demand by helper functions |
| `msafd.dll` | **Winsock ↔ kernel interface** — the layer that translates Winsock calls into kernel network calls | Internally by the stack |
| `wship6.dll` | IPv6 helper — `WSAAddressToString`, `WSAStringToAddress` for IPv6 | On demand for IPv6 operations |

**How DLLs work (brief):**
- **Dynamic linking:** the library code lives in a separate `.dll` file loaded at run time. The `.exe` does not contain the library code — it calls into the DLL.
- **Advantages:** smaller executable, easy to update (replace the DLL, all programs pick up the fix), code is shared across programs.
- **Disadvantages:** if the DLL is missing, wrong version, or corrupted, the program may fail to start ("dependency problem" / "**DLL hell**").

**Static vs dynamic linking (also asked):**
- **Dynamic (DLL):** code in a shared `.dll`, loaded at run time. Smaller `.exe`, easier updates, but requires the DLL present.
- **Static:** code copied into `.exe` at compile time. No external dependency, always runs, but larger executable and must recompile to update.

---
**Marking scheme (8 marks):** table of DLLs = **4**, ws2_32 = **1**, mswsock extensions = **1**, DLL concept (pros/cons) = **1**, static-vs-dynamic = **1**.

---

### Q34 🔴★ Winsock TCP & UDP client-server sequences with code. (asked: Gandaki Q5b — 8 marks)

**TCP server (full sequence + code):**
```
WSAStartup → socket → bind → listen → accept → recv/send → closesocket → WSACleanup
```

```c
// Winsock TCP server
#include <winsock2.h>
#pragma comment(lib, "ws2_32.lib")

int main() {
    WSADATA w; WSAStartup(MAKEWORD(2,2), &w);        // 1. init

    SOCKET s = socket(AF_INET, SOCK_STREAM, 0);        // 2. create

    SOCKADDR_IN sa;
    sa.sin_family = AF_INET;
    sa.sin_port = htons(5150);
    sa.sin_addr.s_addr = htonl(INADDR_ANY);            // 3. bind
    bind(s, (SOCKADDR*)&sa, sizeof(sa));

    listen(s, 5);                                       // 4. listen (backlog = 5)

    SOCKADDR_IN cli; int clen = sizeof(cli);
    SOCKET cs = accept(s, (SOCKADDR*)&cli, &clen);     // 5. accept (blocks until client)

    char buf[1024]; int n = recv(cs, buf, sizeof(buf), 0); // 6. recv
    send(cs, buf, n, 0);                               // 7. send

    closesocket(cs); closesocket(s);                   // 8. close
    WSACleanup();                                      // 9. cleanup
}
```

**TCP client (full sequence):**
```
WSAStartup → socket → connect → send/recv → closesocket → WSACleanup
```

```c
SOCKET s = socket(AF_INET, SOCK_STREAM, 0);
SOCKADDR_IN sa; sa.sin_family = AF_INET; sa.sin_port = htons(5150);
inet_pton(AF_INET, "127.0.0.1", &sa.sin_addr);
connect(s, (SOCKADDR*)&sa, sizeof(sa));                 // no bind needed (kernel assigns ephemeral)
send(s, "hello", 5, 0);
recv(s, buf, sizeof(buf), 0);
closesocket(s);
```

**UDP server:**
```
WSAStartup → socket → bind → recvfrom → closesocket → WSACleanup
```
(No `listen`/`accept` — UDP is connectionless.)

**UDP client:**
```
WSAStartup → socket → sendto → closesocket → WSACleanup
```
(No `bind` needed, no `connect` — you specify the peer with each `sendto`.)

**Key difference between TCP and UDP:**
- TCP: `listen()` and `accept()` are required on the server — they establish the connection.
- UDP: no `listen`/`accept` — the server just binds and waits for datagrams with `recvfrom`.

---
**Marking scheme (8 marks):** TCP server code = **3**, TCP client = **2**, UDP server sequence = **2**, TCP-vs-UDP difference = **1**.

---

### Q35 🔴★ What is overlapped I/O in Winsock? How does it support async? (asked: NCIT 2025 Q7a — 7 marks)

**Overlapped I/O** is Winsock's model for issuing **multiple I/O operations simultaneously** and letting the kernel handle them in the background. Instead of blocking on each read/write, the process **continues working** and is notified when each operation completes.

**How it works:**
1. Create the socket as **overlapped**: `WSASocket(..., WSA_FLAG_OVERLAPPED)` — this is required; without it, overlapped calls fail.
2. Issue I/O calls using the overlapped functions: `WSASend`, `WSARecv`, `WSARecvFrom`, `WSAIoctl`, `AcceptEx`.
3. Each call takes a **`WSAOVERLAPPED` structure** (contains a manual-reset event, status, and offset).
4. If the operation **completes immediately** → the function returns TRUE. The kernel did all the work.
5. If the operation **returns `SOCKET_ERROR` + `WSA_IO_PENDING`** → the call was **queued** (this is NOT an error). Completion is signaled later via one of two mechanisms:
   - **Event object:** the `hEvent` field in `WSAOVERLAPPED` is a Win32 event; the kernel signals it when the operation completes.
   - **Completion routine (callback):** pass a callback function in the overlapped struct; the kernel calls it when the operation completes.

**Why it supports async:**
- The thread **does not block** on each I/O operation — it can issue 10 `WSASend` calls at once and continue doing other work.
- When each completes, the event or callback fires — the thread handles results one at a time, concurrently.
- One thread can manage **hundreds of outstanding I/O operations** — no thread-per-connection needed.
- **Best throughput** of all Winsock I/O models.

**Overlapped completion detection (Win32):**
- `WaitForSingleObject(event, timeout)` — wait for one overlapped to complete.
- `WaitForMultipleObjects(count, events, ...)` — wait for any of several to complete.
- When signaled, check with `WSAGetOverlappedResult()` to learn how many bytes were transferred.

**Relationship to Windows IOCP (I/O Completion Ports):**
- Overlapped + IOCP is the **most scalable** model on Windows — the OS manages a thread pool and dispatches completions to waiting threads, efficiently handling thousands of connections (NT/2000+).

---
**Marking scheme (7 marks):** what overlapped I/O is = **2**, WSA_FLAG_OVERLAPPED + WSAOVERLAPPED = **2**, completion via event/callback = **2**, advantage (thread doesn't block) = **1**.

---

### Q36 🟡★ Event-driven programming & WSAEventSelect. (asked: NCIT Q7a alt, Gandaki Q6b)

**Event-driven programming** = the flow of the program is controlled by **events** (input, I/O readiness, messages) rather than a linear sequence. A loop detects events, and the appropriate handler is dispatched when an event fires. This is the dominant pattern for network servers and GUIs.

**WSAEventSelect** enables event-driven I/O in Winsock by associating a socket with a **Win32 event object**:

```c
WSAEVENT hEvent = WSACreateEvent();   // 1. create event
WSAEventSelect(s, hEvent, FD_READ | FD_WRITE | FD_CLOSE);  // 2. bind events to socket
```

**Usage pattern (loop):**
1. Create events for each socket.
2. Call `WSAWaitForMultipleEvents(count, events, ...)` — blocks until **one or more** events are signaled.
3. When woken, call `WSAEnumNetworkEvents(socket, event, &networkEvents)` to find out *what happened* (FD_READ, FD_WRITE, FD_CLOSE, etc.).
4. Handle the event (e.g. `recv` for FD_READ, `send` for FD_WRITE).

**Key features:**
- **No window needed** (unlike `WSAAsyncSelect`) — works in console apps and services.
- Can wait on up to **64 events per thread** (use `WSAWaitForMultipleEvents`'s limit).
- Socket becomes **non-blocking** automatically after `WSAEventSelect`.

**When to use:**
- Console servers, background services, or any Windows app that doesn't have a message loop/window.
- Good for a moderate number of sockets (up to 64 per thread).

---
**Marking scheme (6 marks):** event-driven concept = **2**, WSAEventSelect mechanism = **2**, code pattern = **1**, advantage (no window) = **1**.

---

### Q37 🟡★ WSAAsyncSelect vs WSAEventSelect. (asked: NCIT alt)
Both are "event notification" I/O models that tell you when a socket is ready — but they differ in **how** the notification is delivered:

| Feature | WSAAsyncSelect | WSAEventSelect |
|---|---|---|
| Notification mechanism | **Windows messages** to a **window procedure** (WndProc) | **Event object** is signaled (Win32 event) |
| Requires a window? | **Yes** — needs a message loop and a window handle (HWND) | **No** — works in console/service apps |
| Target app type | GUI applications with a message loop | Console apps, services, background daemons |
| How to check readiness | `case WM_SOCKET: ...` in WndProc | `WSAWaitForMultipleEvents` + `WSAEnumNetworkEvents` |
| Socket mode | Becomes non-blocking | Becomes non-blocking |
| Scalability | Limited by Windows message queue (many messages → performance) | Up to 64 events per thread, lighter overhead |
| Portability | Windows only | Windows only |

**WSAAsyncSelect is the older model (Winsock 1.1):**
- Notifications are delivered as `WM_SOCKET` messages to a window — you handle them in the `WndProc`.
- Simple for GUI apps that already have a message loop.

**WSAEventSelect is the newer model (Winsock 2.0):**
- No window required — events are Win32 kernel objects (`WSAEVENT`).
- Suitable for background services and console apps (the most common Winsock server pattern).

**Key point for exams:** both are Windows-only; both make sockets non-blocking; the only real difference is **how** you are notified (window message vs event object).

---
**Marking scheme (5 marks):** what both are = **1**, table of differences = **3**, when to use each = **1**.

---

### Q38 🟡★ WSAPoll vs select. (asked: Gandaki Q6b alt)
Both are **I/O multiplexing** functions that check which sockets are ready — but differ in the data structures they use:

| Feature | `select` | `WSAPoll` |
|---|---|---|
| Data structure | `fd_set` — a fixed bitmap (typically `FD_SETSIZE = 64` on Windows) | **Array of `WSAPOLLFD`** structs — no fixed limit |
| Size limit | Up to `FD_SETSIZE` (64 on Windows) | **No fixed limit** — dynamically sized array |
| Sets modified? | **Yes** — select overwrites the fd_sets; must reinitialize before every call | **No** — the array of `WSAPOLLFD` structs is preserved, just update `revents` |
| Result format | Bit field (`FD_ISSET`) — which fds are ready | Event bitmask in `revents` field of each `WSAPOLLFD` |
| Portability | Portable (Unix and Windows) | Windows-only (but mirrors Unix `poll`) |
| Setup per call | Must call `FD_ZERO`, `FD_SET` before each call | Just pass the array — much simpler for many sockets |

**Why WSAPoll is better for many sockets:**
- With `select`, if you have 100 sockets but `FD_SETSIZE = 64`, you **can't use it** — you need multiple threads or a different model.
- With `WSAPoll`, you create an array of 100 `WSAPOLLFD` structs and pass it — the OS checks all of them in one call.
- No need to reinitialize before each call (unlike `select`, which overwrites the fd_sets).

**`WSAPOLLFD` structure:**
```c
typedef struct pollfd {
    SOCKET fd;       // socket to check
    SHORT  events;   // events interested in (POLLIN, POLLOUT)
    SHORT  revents;  // events that actually occurred (returned by WSAPoll)
} WSAPOLLFD;
```

**When to use:**
- `WSAPoll` for many sockets on Windows (better than `select` at scale).
- `select` for simple cases or when portability to Unix is required (both systems have `select`).

---
**Marking scheme (5–6 marks):** what both are = **1**, comparison table = **3**, WSAPollFD structure = **1**, when to use = **0–1**.

---

### Q39 🟡 Graceful close in Winsock.
A **graceful close** ensures all data is delivered before the connection is shut down. In Winsock:

```c
shutdown(s, SD_SEND);   // "I am done sending" — sends TCP FIN to peer
// ... do any final recv from peer if needed ...
closesocket(s);         // fully releases the socket
```

**Why `shutdown` before `close`:**
- `shutdown(SD_SEND)` sends a **TCP FIN** to the peer — this is the **half-close** signal. The peer knows no more data will come from you, but can still send.
- `closesocket(s)` immediately **releases** the socket handle and any buffered data. If there is still data in the send buffer, it may be **discarded** (or sent with RST if linger is set).
- By calling `shutdown` first, you ensure the FIN is sent **and the peer gets it**, then you call `closesocket` to clean up.

**`shutdown` parameters:**
- `SD_SEND` — stop sending (send FIN).
- `SD_RECEIVE` — stop receiving (sends RST to peer if data arrives).
- `SD_BOTH` — stop both (close both directions).

**Graceful close sequence:**
1. Server has finished responding to client.
2. Server calls `shutdown(s, SD_SEND)` — sends FIN.
3. Server calls `closesocket(s)` — releases the socket.
4. Client gets EOF on next `recv` — knows the server is done.
5. Client sends its own FIN (via `shutdown` + `closesocket`).

**vs abrupt close:**
- Just calling `closesocket(s)` without `shutdown` is an **abrupt close** — the OS may send a **RST** instead of a FIN, which tells the peer to abort immediately (peer sees `ECONNRESET`).
- Use `shutdown` + graceful linger (`SO_LINGER`) to ensure data is delivered before closing.

---
**Marking scheme (5 marks):** shutdown explained = **2**, closesocket vs shutdown = **1**, half-close concept = **1**, graceful vs abrupt = **1**.

---

### Q40 🟢 WSAEnumProtocols / WSAAccept / WSAConnect (Winsock extensions).
These are **Winsock-specific extensions** (not in standard Berkeley sockets) that add functionality:

**`WSAEnumProtocols`** — list all installed network protocols and their capabilities:
```c
WSAEnumProtocols(lpiProtocols, lpProtocolBuffer, lpdwBufferLength);
```
Returns an array of `WSAPROTOCOL_INFO` structs — each describes a protocol (TCP, UDP, etc.), its address family, socket type, capabilities (`dwServiceFlags1`), and name. Used to discover what protocols are available and select the right one.

**`WSAAccept`** — accept with a **condition function** (reject or defer connections:
```c
WSAAccept(s, addr, addrlen, conditionFunction, callbackData);
```
The `conditionFunction` is called **before** the connection is accepted. It can:
- Return `CF_ACCEPT` — accept the connection.
- Return `CF_REJECT` — reject it (send RST to client).
- Return `CF_DEFER` — defer (accept later with `WSAEventSelect` + `WSAGetOverlappedResult`).
This is useful for rate-limiting or filtering clients before accepting them.

**`WSAConnect`** — connect with **caller data** and **QoS** (Quality of Service):
```c
WSAConnect(s, name, namelen, lpCallerData, lpCalleeData, lpSQOS, lpGQOS);
```
- `lpCallerData`/`lpCalleeData` — send/receive data during the connection setup (not supported by most providers).
- `lpSQOS`/`lpGQOS` — specify QoS parameters (bandwidth, latency) for the connection. Useful for multimedia or real-time apps.

**When to use:** WSAEnumProtocols for protocol discovery; WSAAccept for connection filtering; WSAConnect for QoS-aware connections. Most Winsock apps use the standard `accept`/`connect` — these extensions are for advanced cases.

---
**Marking scheme (4–5 marks):** WSAEnumProtocols = **1**, WSAAccept = **2**, WSAConnect = **1**, when to use = **0–1**.

---

### Q41 🟡★ 5 Winsock I/O models.
Winsock offers **five** distinct I/O models for handling asynchronous network operations. Each model represents a different approach to the question: "how do I know when a socket is ready for reading/writing?"

1. **select (select/poll)** — the classic cross-platform model. Pass a set of sockets (`fd_set` or `WSAPOLLFD` array); `select`/`WSAPoll` blocks until one or more are ready. Simple, works everywhere, but limited to `FD_SETSIZE` sockets.

2. **WSAAsyncSelect** — Windows message-based model. Bind a socket to a **window** (`HWND`); when events occur (FD_READ, FD_WRITE, etc.), the OS sends a `WM_SOCKET` message to the window's message procedure. Requires a GUI/message loop.

3. **WSAEventSelect** — event-object model. Bind a socket to a **Win32 event** (`WSAEVENT`). When events occur, the event is signaled; use `WSAWaitForMultipleEvents` + `WSAEnumNetworkEvents` to detect which sockets fired. **No window needed**; up to 64 events/thread.

4. **Overlapped I/O** — the most advanced model. Issue multiple `WSASend`/`WSARecv` calls at once using `WSAOVERLAPPED` structures. The kernel runs them in the background; completions are signaled via **events** or **completion routines (callbacks)**. **Best throughput**; one thread manages many outstanding operations.

5. **I/O Completion Ports (IOCP)** — the most scalable model. Create a completion port, associate sockets with it, and let the OS manage a **thread pool** that processes completions as they arrive. Handles **hundreds of thousands** of connections efficiently. Available on NT/2000+ only.

**Which model for which scenario:**
- Simple app, few sockets: **select** or **WSAEventSelect**.
- GUI app: **WSAAsyncSelect**.
- High-throughput server (hundreds of connections): **overlapped I/O**.
- Very high scalability (thousands of connections): **IOCP**.

---
**Marking scheme (7–8 marks):** each model = **1** (5 marks), which-to-use = **2–3**.

---

### Q42 🟡★ Is a common Unix+Windows app possible? How? (asked: NCIT Q5b alt)

**Yes — you can write a single codebase that compiles and runs on both Unix and Windows** by abstracting the platform-specific parts behind `#ifdef` preprocessor guards.

**What differs between platforms:**
- Header files: `<sys/socket.h>` (Unix) vs `<winsock2.h>` (Windows).
- Socket type: `int` (Unix) vs `SOCKET` (Windows).
- Close function: `close()` (Unix) vs `closesocket()` (Windows).
- Error reporting: `errno` (Unix) vs `WSAGetLastError()` (Windows).
- Init/cleanup: no init (Unix) vs `WSAStartup`/`WSACleanup` (Windows).

**Cross-platform wrapper pattern:**
```c
#ifdef _WIN32
  #include <winsock2.h>
  #pragma comment(lib, "ws2_32.lib")
  #define close_socket closesocket     // map Unix name → Windows name
  typedef int socklen_t;               // Unix has this; Windows may not
#else
  #include <sys/socket.h>
  #include <netinet/in.h>
  #include <unistd.h>
  #define close_socket close
#endif

/* --- platform-independent code --- */
int s = socket(AF_INET, SOCK_STREAM, 0);
// ... bind, listen, connect, send, recv ...
close_socket(s);

#ifdef _WIN32
  WSACleanup();   // called at program exit, guarded by _WIN32
#endif
```

**Key practices:**
1. Put all platform differences behind `#ifdef _WIN32` / `#else` at the top of the file.
2. Use **`#define close_socket closesocket`** (or a wrapper function) so the rest of the code is clean.
3. Wrap `WSAStartup`/`WSACleanup` in `#ifdef _WIN32` — they don't exist on Unix.
4. On Windows, always use `WSAGetLastError()` for socket errors (not `errno`).
5. For binary compatibility: use `SOCKET` (Windows) and `int` (Unix) behind a typedef, or just use `int` and cast.

**Limitation:** platform-specific features (IOCP on Windows, epoll on Linux) require separate implementations — only use what is common (socket API, select) in the shared code.

---
**Marking scheme (5–6 marks):** yes + why (header/type/error differences) = **2**, code wrapper pattern = **2**, key practices = **1–2**.

---
## Unit 6 — Utilities, Trends & Security

### Q43 🟡★ Name & describe network utilities. (asked: short notes — telnet, ipconfig/ifconfig, remote login, iperf, netstat)

Network utilities are **command-line tools** that help diagnose, test, and understand network connectivity and performance. Each serves a specific purpose:

**1. `ping` — ICMP reachability + round-trip time (RTT):**
- Sends **ICMP Echo Request** packets to a target host and waits for **Echo Reply**.
- Shows round-trip time (latency) in milliseconds, packet loss percentage, and TTL.
- `ping -c 4 google.com` — send 4 pings.
- Works at Layer 3 (IP); if `ping` works, IP connectivity exists.

**2. `telnet` — remote terminal login / TCP port tester:**
- Connects to a remote host on a specified port (default 23) and gives a terminal shell.
- **Modern use:** TCP port testing — `telnet host 80` sends data to port 80 and shows the response. Useful for testing HTTP, SMTP, SSH connectivity manually.
- **Limitation:** sends everything (including passwords) in **cleartext** — replaced by SSH for remote login.

**3. `ip` / `ifconfig` — configure and inspect network interfaces:**
- Shows/sets the IP address, netmask, MAC address, MTU, and interface status (up/down).
- `ifconfig eth0` (older Linux) or `ip addr show` (modern Linux); `ipconfig` (Windows).
- Used to troubleshoot: "is my interface even up? what IP do I have?"

**4. `iperf` — throughput and bandwidth measurement:**
- **Client-server tool:** runs a server (`iperf -s`) and a client (`iperf -c server_ip`) to measure TCP/UDP throughput between two hosts.
- Reports bandwidth in Mbits/sec, jitter (for UDP), and packet loss.
- Essential for testing network performance: "is the link actually delivering 100 Mbps?"

**5. `netstat` — connections, routing table, listening ports, interface stats:**
- `netstat -tlnp` — show all **TCP** ports in **listening** state (with PIDs).
- `netstat -an` — show all established connections.
- Shows routing table (`netstat -r`), interface statistics (`netstat -i`).
- Replaced by `ss` on modern Linux (`ss -tlnp` is faster).

**6. Remote login (`rlogin` / `ssh`):**
- `rlogin` — the old UNIX remote terminal tool (insecure, port 513, cleartext).
- `ssh` — **Secure Shell** (port 22) — encrypts all traffic (including passwords), authenticates via keys or passwords. The modern replacement for `rlogin`, `telnet`, and `rsh`.
- `ssh user@host` gives a secure terminal on the remote host.

---
**Marking scheme (6 marks):** each utility = **1** (6 marks).

---

### Q44 🔴★ HTTP vs WebSocket + simple server. (asked: NCIT Q7b — 8 marks)

**HTTP (Hypertext Transfer Protocol):**
- **Request-response** model: the client sends a request; the server sends a response; then the connection is typically closed (or kept alive briefly for the next request).
- **Half-duplex in practice:** the client talks first, then the server replies. The server **cannot** push data to the client unprompted.
- **Higher overhead:** each request carries full HTTP headers (cookies, User-Agent, Accept, etc.) — even if the payload is small.
- **Stateless:** each request is independent; no built-in state between requests (use cookies/sessions for state).

**WebSocket:**
- **Full-duplex:** both client and server can send messages **at any time** — true two-way communication.
- **Persistent:** a single TCP connection stays open for the duration of the session — no repeated connect/disconnect overhead.
- **Lower overhead:** once the handshake is complete, WebSocket frames are tiny (2–14 bytes header vs hundreds of bytes for HTTP headers).
- **Server push:** the server can send data to the client **without being asked** — essential for real-time applications.

**Comparison table:**

| Feature | HTTP | WebSocket |
|---|---|---|
| Communication model | Request–response | **Full-duplex** |
| Connection | Closed after each response (unless keep-alive) | **Persistent** (stays open) |
| Header overhead | Large (repeated headers per request) | **Tiny** (2–14 bytes per frame) |
| Server push | **No** (server can only respond) | **Yes** (server can send anytime) |
| Latency | Higher (new TCP connection or keep-alive overhead) | **Lower** (one connection, no headers) |
| Use cases | Web pages, REST APIs, file downloads | Chat apps, multiplayer games, live dashboards, stock tickers, IoT |

**How a WebSocket connection is established (the handshake):**
1. Client sends an HTTP request: `GET /chat HTTP/1.1` + `Upgrade: websocket` + `Sec-WebSocket-Key: <base64>`.
2. Server replies: `HTTP/1.1 101 Switching Protocols` + `Upgrade: websocket` + `Sec-WebSocket-Accept: <SHA1 hash>`.
3. After the 101 response, the connection is **upgraded to WebSocket** — both sides exchange frames, not HTTP requests.

**Simple WebSocket server (pseudo-code):**
```
socket() → bind() → listen()                  // normal TCP server setup
conn = accept()                                // client's Upgrade request arrives
read HTTP request; verify "Upgrade: websocket"
send "HTTP/1.1 101 Switching Protocols" + Sec-WebSocket-Accept header
loop {
    read frame from conn;                       // full-duplex: client can send anytime
    broadcast to all clients;                   // server can also push anytime
}
```

**WebSocket frames (brief):** each frame has: `FIN` (1 bit) + `opcode` (4 bits: 0x1=text, 0x2=binary, 0x8=close, 0x9=ping, 0xA=pong) + `MASK` (1 bit) + `payload length` + `masking key` (if masked) + `payload`.

---
**Marking scheme (8 marks):** HTTP vs WebSocket table = **3**, handshake diagram/explanation = **2**, server pseudo-code = **2**, frame structure (bonus) = **1**.

---

### Q45 🟡★ What is gRPC? (short note)
**gRPC** is a **high-performance, open-source RPC (Remote Procedure Call) framework** developed by Google, built on top of **HTTP/2** and using **Protocol Buffers** (protobuf) for binary serialization.

**Key features:**
- **Language-agnostic:** a Python client can call a C++ server seamlessly — code is generated from `.proto` service definitions.
- **Four call models:** unary (one request, one response), server streaming, client streaming, **bidirectional streaming**.
- **Binary serialization (Protocol Buffers):** much faster and smaller than JSON/XML. You define services and messages in a `.proto` file; the `protoc` compiler generates client/server stubs.
- **Built on HTTP/2:** multiplexed streams, header compression, TLS by default — fast and secure.
- **Used in:** microservices, mobile↔backend communication, Kubernetes, Google Cloud.

**How gRPC works (brief):**
1. Define the service in a `.proto` file:
   ```protobuf
   service Greeter { rpc SayHello (HelloRequest) returns (HelloReply); }
   message HelloRequest { string name = 1; }
   message HelloReply { string message = 1; }
   ```
2. Generate code with `protoc` (generates stubs for client and server).
3. Server implements the service; client calls the generated stub → transparent RPC.

**vs REST/HTTP:** gRPC is faster (binary, HTTP/2), supports streaming, generates type-safe code; REST is simpler, human-readable, and more widely supported. gRPC is preferred for internal microservice-to-microservice communication; REST for public APIs.

---
**Marking scheme (5–6 marks):** what gRPC is = **1**, HTTP/2 + protobuf = **2**, four call models = **1**, proto example = **1**, vs REST = **1**.

---

### Q46 🟡★ TLS/SSL + cryptography concepts. (short note — asked)
**TLS (Transport Layer Security)**, formerly SSL, provides **encryption + authentication + integrity** for data in transit between two endpoints (e.g. client ↔ web server). It sits between the application and TCP.

**Three pillars:**

1. **Encryption** — makes data unreadable to eavesdroppers:
   - **Symmetric encryption** (AES, ChaCha20): same key encrypts and decrypts. Fast — used for actual data transfer.
   - **Asymmetric encryption** (RSA, ECC): public key encrypts, private key decrypts (or vice versa). Slow — used **only** to exchange the symmetric key during the handshake.

2. **Authentication** (certificates):
   - The server proves its identity by presenting a **digital certificate** signed by a trusted **Certificate Authority (CA)**.
   - The client verifies the CA's signature using a built-in CA certificate store.
   - Optionally, the client also presents a certificate (mutual TLS) for server-side authentication.

3. **Integrity** (hashing / HMAC):
   - **Hash functions** (SHA-256) produce a fixed-size digest from input — used for integrity checks.
   - **HMAC** (Hash-based Message Authentication Code) = hash + secret key — ensures data hasn't been tampered with and proves it came from the expected party.

**TLS Handshake (simplified):**
1. Client → Server: "ClientHello" (supported cipher suites, TLS version).
2. Server → Client: "ServerHello" (chosen cipher suite) + **server certificate**.
3. Client verifies certificate against its CA store.
4. Client and server agree on a **session key** (via key exchange — ECDHE for forward secrecy).
5. Both sides switch to **symmetric encryption** using the session key — all subsequent data is encrypted.

**Forward secrecy:** modern TLS uses **Ephemeral Diffie-Hellman (DHE/ECDHE)** — the session key is freshly generated for each connection and **never stored on disk**, so even if the server's private key is later compromised, past sessions cannot be decrypted.

**OpenSSL example:**
```c
SSL_CTX *ctx = SSL_CTX_new(TLS_client_method());  // create context
SSL *ssl = SSL_new(ctx);                           // create SSL object
SSL_set_fd(ssl, sockfd);                           // associate with socket
SSL_connect(ssl);                                   // perform handshake
SSL_write(ssl, "hello", 5);                        // encrypted send
SSL_read(ssl, buf, sizeof(buf));                   // encrypted recv
SSL_shutdown(ssl); SSL_free(ssl); SSL_CTX_free(ctx);
```

---
**Marking scheme (6 marks):** encryption = **1**, certificates/authentication = **1**, integrity/HMAC = **1**, handshake = **1**, forward secrecy = **1**, OpenSSL code = **1**.

---

### Q47 🟡★ What is SDN? Key advantages. (asked: NCIT Q6b — 8 marks)
**SDN (Software-Defined Networking)** is a network architecture that **separates the control plane (brain) from the data plane (muscle)** and centralises network management in software.

**Traditional networking vs SDN:**
In traditional networks, each switch/router has its own control plane (routing protocols, forwarding decisions) — they are autonomous and decentralised. Changing the network requires configuring each device individually.

In SDN, the control plane is **extracted** from the switches and placed in a **centralised SDN controller** — the "brain" of the network. The switches become simple "forwarding machines" that just follow rules installed by the controller.

**The three layers:**

| Layer | What it does | Example |
|---|---|---|
| **Application layer** | Network applications and policies (firewall, load balancing, routing) | Custom Python app, network management UI |
| **Control layer** | Centralised **SDN controller** — makes all routing/policy decisions | OpenDaylight, ONOS, Floodlight |
| **Data/Infrastructure layer** | Switches just **forward packets** according to installed **flow rules** | OpenFlow switches |

**OpenFlow protocol** = the standard protocol by which the SDN controller communicates with switches:
- Controller installs **flow rules** (match fields + actions) into the switch's **flow table**.
- When a packet arrives, the switch checks its flow table:
  - **Match found →** perform the action (forward, drop, modify).
  - **No match →** send the packet to the controller (packet-in).
- Controller computes the path, installs flow rules on all relevant switches (packet-out).

**Key advantages of SDN:**
1. **Centralised control** — one controller has the entire network view; makes globally optimal decisions (unlike traditional distributed protocols that converge slowly).
2. **Programmability** — the network is controlled by software, not manual CLI configuration. Enables automation, rapid deployment of new services.
3. **Agility / rapid innovation** — new policies can be deployed in seconds (write a Python script → push rules via controller) instead of reconfiguring hundreds of devices.
4. **Better resource utilisation** — the controller sees all traffic patterns and can optimise paths globally.
5. **Vendor independence** — switches just speak OpenFlow; the controller is vendor-agnostic.

---
**Marking scheme (8 marks):** what SDN is = **1**, three layers table = **3**, OpenFlow explanation = **2**, advantages list = **2**.

---

### Q48 🟡★ OpenFlow, P4, Frenetic. (short notes — asked: Gandaki "P4 and frenetic programming")

**OpenFlow — controller ↔ switch communication protocol:**
- **Standard protocol** by which the SDN controller programs the **flow tables** of network switches.
- The controller installs rules: "if a packet matches [src IP, dst IP, port, ...], then [forward to port X / drop / modify / send to controller]."
- Switches are simple: they look up packets in their flow table and follow the rules. No routing logic in the switch.
- Used in data centres, research networks, and SDN deployments.

**P4 — programming the data plane:**
- **P4 (Programming Protocol-independent Packet Processors)** is a high-level **DSL (domain-specific language)** that lets you program **what the switch does** — not just forwarding, but custom packet parsing, processing, and modification.
- OpenFlow is limited to a fixed set of match fields (IP, MAC, port). P4 lets you define your own headers and processing logic (e.g. process custom protocols, implement new load-balancing schemes).
- You write a P4 program → compile it → deploy to a P4-programmable switch → the switch processes packets according to your custom logic.
- Key use: **custom data-plane logic** — network researchers and operators can implement protocols that OpenFlow cannot express.

**Frenetic — programming the controller:**
- **Frenetic** is a high-level **DSL** for programming the SDN controller — you write network policies in Frenetic (a functional language), and the Frenetic compiler translates them into **OpenFlow rules** and pushes them to switches.
- Advantage: the programmer writes high-level policies (e.g. "forward HTTP to the load balancer, drop all other traffic") without manually computing which OpenFlow rules to install on which switch.
- Frenetic handles the hard parts: composing multiple policies, computing the flow tables across multiple switches, and handling packet-in events.
- Part of the broader **"network programming languages"** research area (along with Pyretic, NetKAT, etc.).

**Summary:**
| Tool | Programs what? | Level |
|---|---|---|
| **OpenFlow** | Switch flow tables (match + action) | Low-level rules |
| **P4** | Switch packet processing logic | Data-plane DSL |
| **Frenetic** | Controller policies (compiles to OpenFlow) | Controller DSL |

---
**Marking scheme (5–6 marks):** OpenFlow = **2**, P4 = **2**, Frenetic = **1**, comparison table = **1**.

---

### Q49 🟡★ WebSockets short note. (asked: Gandaki short note)
**WebSocket** is a **full-duplex, persistent messaging protocol** over a single TCP connection, designed for low-latency, low-overhead two-way communication between client and server in web applications.

**Key characteristics:**
- **Full-duplex:** both client and server can send messages at any time simultaneously.
- **Persistent:** the TCP connection stays open for the session duration — no repeated connect/disconnect.
- **Low overhead:** WebSocket frames have a 2–14 byte header (vs hundreds of bytes for HTTP headers per request).
- **URL scheme:** `ws://` (unencrypted) or `wss://` (encrypted, over TLS).
- **Standardised:** RFC 6455, supported by all modern browsers.

**The handshake (HTTP → WebSocket upgrade):**
```
Client: GET /chat HTTP/1.1
        Upgrade: websocket
        Connection: Upgrade
        Sec-WebSocket-Key: dGhlIHNhbXBsZQ==

Server: HTTP/1.1 101 Switching Protocols
        Upgrade: websocket
        Connection: Upgrade
        Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
After the `101` response, the connection is upgraded — both sides exchange frames.

**WebSocket frames:** FIN (1 bit) + opcode (4 bits: 0x1 text, 0x2 binary, 0x8 close, 0x9 ping, 0xA pong) + MASK bit + payload length + masking key (4 bytes, client→server) + payload.

**Use cases:** chat applications, multiplayer games, live dashboards, stock tickers, IoT real-time push, collaborative editing (Google Docs), live sports scores.

**vs HTTP polling:** WebSocket is far more efficient — no repeated HTTP headers, no polling overhead, true real-time updates pushed from server.

---
**Marking scheme (5–6 marks):** what it is = **1**, key characteristics = **2**, handshake = **1**, frames = **1**, use cases = **1**.

---

### Q50 🟢 Daemonizing techniques in Unix.
A **daemon** is a long-running background process with no controlling terminal. To daemonize a process in Unix, follow these standard steps:

```c
if (fork() > 0) exit(0);     // 1. parent exits, child continues
setsid();                     // 2. new session, detach from controlling terminal
if (fork() > 0) exit(0);     // 3. second fork (double-fork) — not a session leader anymore
chdir("/");                   // 4. safe working directory
umask(0);                     // 5. clear file-mode mask
// 6. redirect stdin/stdout/stderr to /dev/null
open("/dev/null");
dup2(0, 1);  // stdout → /dev/null
dup2(0, 2);  // stderr → /dev/null
```

**Step-by-step explanation:**
1. **`fork()`** — create a child; parent exits so the shell prompt returns immediately.
2. **`setsid()`** — create a **new session** (new process group, no controlling terminal). The child is now a session leader.
3. **Second `fork()` (double-fork)** — after the first `setsid()`, the process is a session leader. A session leader *could* re-acquire a controlling terminal if it opens a terminal device. By forking again and having the parent exit, the grandchild is **not** a session leader — it can **never** acquire a controlling terminal.
4. **`chdir("/")`** — avoid holding a busy/mounted filesystem as the current directory (could prevent unmounting).
5. **`umask(0)`** — clear the file creation mask so the daemon can create files with full intended permissions.
6. **Redirect stdin/stdout/stderr to `/dev/null`** — since there is no controlling terminal, reading from stdin would block/error, and writing to stdout/stderr would fail. Redirecting to `/dev/null` makes them harmless (output is discarded, reads return EOF).

**Result:** a fully detached background process, with no controlling terminal, no parent (orphaned and adopted by `init`/`systemd`), writing only to logs (use `syslog`).

**Optional extras:** change the process name (`prctl(PR_SET_NAME, "mydaemon")`), write a PID file (`/var/run/mydaemon.pid`), set up signal handlers for graceful shutdown.

---
**Marking scheme (5 marks):** code/6 steps = **3** (0.5 each), double-fork explanation = **1**, redirect to /dev/null = **1**.

---

## Full 6-Part 8-Mark Model Answers

> Every answer below follows the structure **①Definition → ②Diagram → ③Full concept → ④Example/code → ⑤Common errors/limits → ⑥Conclusion**. Practise writing each one by hand with a pencil. The diagrams are worth the most — an examiner who sees a clear diagram knows you understand immediately.

> **How to get full marks (exam technique):**
> - **① (≈1 mark):** one bold definition sentence + name the category.
> - **② (≈2 marks):** biggest, clearest diagram you can draw — handshake, table, state machine, layered stack, code flow. Label every arrow.
> - **③ (≈3 marks):** the body — expand *why* and *how*, define every acronym, give the rules in a table.
> - **④ (≈1 mark):** a real C snippet or worked example (a port number, a packet format). A code block scores even if short.
> - **⑤ (≈1 mark):** a real gotcha, a deprecated feature, or a limitation.
> - **⑥ (≈0.5–1 mark):** 1–2 sentences: when to use it + what problem it solves. Don't repeat ①.
> - **Scaling rule:** for a genuine 8-mark answer, write each section 1–2 sentences longer than shown here, and always draw the biggest, clearest diagram you can — diagrams are the cheapest marks.

---

### M1. Compare TCP, UDP and SCTP  [🔴★ NCIT 2025 Q1a]

**① Definition** — TCP, UDP and SCTP are the three main **transport-layer protocols** that deliver data between applications over IP networks. Each has different trade-offs for reliability, speed and connection style.

**② Diagram — layered positions + comparison table**
```
 ┌─────────────────────────────────────────────────────┐
 │  Application (HTTP, DNS, SIP, FTP, ...)             │
 ├──────────────┬───────────────┬───────────────────────┤
 │  TCP         │  UDP          │  SCTP                │
 │  (reliable   │  (unreliable  │  (reliable +         │
 │   stream)    │   datagram)   │   multihomed +       │
 │              │               │   multistream)       │
 ├──────────────┴───────────────┴───────────────────────┤
 │              IP (Internet Protocol)                   │
 └─────────────────────────────────────────────────────┘
```

| Feature | TCP | UDP | SCTP |
|---|---|---|---|
| **Connection** | connection-oriented (3-way handshake) | **connectionless** (just send) | connection-oriented (4-way handshake + **cookie**) |
| **Reliability** | reliable: ACK, retransmission, checksum | **unreliable**: may lose, duplicate, reorder | reliable: ACK, retransmission, checksum |
| **Ordering** | ordered byte stream (sequence numbers) | **no ordering** (datagrams can arrive out of order) | ordered **per-stream** within one association |
| **Message boundaries** | **none** (pure byte stream — app must parse) | **yes** (each send = one datagram) | **yes** (each send = one message) |
| **Multi-homing** | no (one IP per end) | no | **yes** (several IPs per end for fault tolerance) |
| **Multistreaming** | no | no | **yes** (independent ordered streams avoid head-of-line blocking) |
| **SYN flood protection** | weak (SYN cookies optional) | n/a | **built-in** (cookie in handshake) |
| **Half-close** | yes (`close` sends FIN) | n/a | **no** (association is all-or-nothing) |
| **Common uses** | HTTP, FTP, SMTP, SSH, Telnet | DNS, NFS, SNMP, DHCP, TFTP, video streaming | SIP, SS7/SIGTRAN, Diameter (telephony) |

**③ Full concept**
- **TCP (Transmission Control Protocol)** is the most widely used transport protocol. It establishes a connection using a **three-way handshake** (SYN → SYN+ACK → ACK), then delivers a **reliable, ordered byte stream** of data. Every byte is given a **sequence number**; the receiver sends **ACKs** to confirm receipt; lost segments are **retransmitted**. TCP also has **flow control** (receiver advertises a window) and **congestion control** (sender adapts speed to network load). The trade-off: extra overhead and latency.
- **UDP (User Datagram Protocol)** is the simplest transport protocol. It sends **unreliable datagrams** — each send is one independent message. There is no handshake, no ACK, no retransmission, no ordering. The benefit: very **low overhead, low latency**, so it is ideal for real-time applications (video/audio, DNS lookups) where a small amount of loss is acceptable.
- **SCTP (Stream Control Transmission Protocol)** is a newer protocol designed for **telephony** (SS7 signaling). It is **reliable like TCP**, but adds three unique features: (1) **multistreaming** — multiple independent ordered streams within one connection, so a blocked stream does not block others; (2) **multi-homing** — each end can use several IP addresses simultaneously for fault tolerance; (3) a **four-way handshake with a cookie** to prevent SYN-flood attacks (TCP is vulnerable to these). SCTP also has no half-close state — the association is either open or closed.

**④ Example/code**
```c
// TCP: connection-oriented
int s = socket(AF_INET, SOCK_STREAM, 0);   // stream socket
connect(s, ...);   // three-way handshake happens here
send(s, data, len, 0);  // reliable byte stream

// UDP: connectionless
int s = socket(AF_INET, SOCK_DGRAM, 0);    // datagram socket
sendto(s, data, len, 0, &dest, sizeof(dest)); // no handshake needed

// SCTP: message-oriented
int s = socket(AF_INET, SOCK_SEQPACKET, 0); // seqpacket = ordered messages
```

**⑤ Common errors/limits**
- Confusing TCP's byte-stream model with UDP's datagram model: in TCP you can `send(4 bytes)` then `send(6 bytes)` and the receiver may get all 10 bytes in one `recv` — there is no message boundary. In UDP, each `sendto` = one `recvfrom` datagram.
- UDP has **no congestion control** — if you blast packets, you can cause congestion collapse.
- SCTP is not widely supported on Windows/macOS; Linux has it since kernel 2.6.

**⑥ Conclusion** — Choose TCP when reliability and ordering matter (web, email), UDP when low latency and tolerance for loss matter (streaming, DNS), and SCTP when you need reliability + message boundaries + multi-path fault tolerance (telephony, carrier networks).

---

### M2. TCP Three-Way Handshake + Why ISN should not start from 0  [🔴★ multiple papers]

**① Definition** — The **three-way handshake** is the procedure TCP uses to establish a connection before any application data flows. It synchronises both sides' **initial sequence numbers (ISNs)** and ensures both ends are ready to communicate.

**② Diagram**
```
 CLIENT (active open)                          SERVER (passive open = LISTEN)
      │                                              │
      │  ─── ① SYN (seq = x) ──────────────▶       │
      │       (client asks to connect)               │
      │                                              │
      │  ◀── ② SYN+ACK (seq = y, ack = x+1) ───   │
      │       (server agrees + invites)              │
      │                                              │
      │  ─── ③ ACK (ack = y+1) ──────────────▶     │
      │       (client confirms)                      │
      │                                              │
      │         ═══ CONNECTION ESTABLISHED ═══       │
      │         (both sides: ESTABLISHED state)      │
      │              ── data can now flow ──         │
```

**③ Full concept**
- **Segment ① (SYN):** the client sends a SYN segment with its **initial sequence number (ISN) = x**. This tells the server "I want to connect; my first byte will have sequence number x." The client enters **SYN_SENT** state.
- **Segment ② (SYN+ACK):** the server replies with its own SYN (ISN = y) and an ACK acknowledging the client's SYN (ack = x+1, meaning "I received byte x; send from x+1"). The server enters **SYN_RCVD** state.
- **Segment ③ (ACK):** the client sends ACK acknowledging the server's SYN (ack = y+1). Both sides now enter **ESTABLISHED** state and can exchange data.

Three segments are needed (not two) because TCP is **full-duplex** — each side's sequence number must be synchronised and acknowledged **independently**. Two segments would only acknowledge in one direction.

**Why the ISN should NOT start from 0:**
- If ISNs always started at 0, an **old, delayed segment** from a previous (now closed) connection could carry a sequence number that falls inside the new connection's receive window and be **mistakenly accepted as valid data** (the "wandering duplicate" problem).
- A predictable ISN also makes **session hijacking** trivial — an attacker can guess the ISN and inject fake packets.
- Starting from a **random/unpredictable ISN** (RFC 6528 recommends randomising) plus the **TIME_WAIT state** (2×MSL, which lets old duplicates expire) together prevent both problems.

**④ Example/code** — On the wire (Wireshark capture):
```
No.  Time     Source       Dest          Protocol  Info
1    0.000    192.168.1.1  93.184.216.34 TCP      54321→80 [SYN] Seq=0 Win=65535
2    0.012    93.184.216.34 192.168.1.1  TCP      80→54321 [SYN,ACK] Seq=1 Ack=1 Win=65535
3    0.014    192.168.1.1  93.184.216.34 TCP      54321→80 [ACK] Seq=1 Ack=2 Win=65535
```
(In this capture the ISNs are obfuscated by Wireshark for readability.) The `connect()` system call triggers the SYN; the server's `accept()` returns only after all three segments are done.

**⑤ Common errors/limits**
- A **SYN flood** (attacker sends many SYNs but never completes the handshake) can exhaust the server's **incomplete connection queue**. TCP uses **SYN cookies** (encode the ISN in a cryptographic token so no state is stored until the handshake completes) to defend against this. SCTP's four-way cookie handshake resists SYN floods by design.
- The three-way handshake adds **one round-trip time (1 RTT)** of latency before data can flow — this is why connection pooling and persistent HTTP connections matter.

**⑥ Conclusion** — The three-way handshake synchronises sequence numbers in both directions using three segments; keeping the ISN random and using TIME_WAIT prevents old duplicates and hijacking.

---

### M3. TCP State-Transition Diagram (all 11 states)  [🔴★ NCIT 2025, Gandaki]

**① Definition** — TCP is a **finite state machine**: at any moment a connection is in exactly one of **11 states** (as shown by `netstat` or `ss`). The **state-transition diagram** shows all possible states and how events (sending/receiving segments, application calls) cause transitions between them.

**② Diagram — the full 11-state transition diagram**
```
                        (passive open)
 ┌──────────────────────────────────────────────────────────┐
 │                                                          │
 CLOSED ──────────────────────────────────────────────────▶ LISTEN
   │                                                        │
   │ (active open:                                          │ (recv SYN
   │  connect/send SYN)                                     │  send SYN+ACK)
   ▼                                                        ▼
 SYN_SENT ◀── (simultaneous open) ──▶ SYN_RCVD ◀───────────┘
   │              send SYN+ACK,          │
   │              recv SYN+ACK           │ (recv ACK)
   │              send ACK               ▼
   └───────(recv SYN+ACK)─────────▶ ESTABLISHED
                                         │  ◀════ DATA FLOWS ════▶
                                         │
                    ╔═══════════════════════════════════════════╗
                    ║         TERMINATION (active close)        ║
                    ╚═══════════════════════════════════════════╝
                                         │
                  (app calls close/       │ (recv FIN
                   shutdown → send FIN)   │  send ACK)
                                         ▼
                                  FIN_WAIT_1 ─────────────────▶ CLOSE_WAIT
                                         │                       │
                           (recv ACK)    │               (app calls close
                                         ▼                send FIN)
                                  FIN_WAIT_2                 LAST_ACK
                                         │                       │
                            (recv FIN    │                       │ (recv ACK)
                             send ACK)   │                       │
                                         ▼                       ▼
                                    TIME_WAIT                 CLOSED
                                  (wait 2×MSL)
                                         │
                                         ▼
                                      CLOSED
```

**③ Full concept — each of the 11 states explained:**
1. **CLOSED** — the initial and final state; no connection exists. (This state is fictional — it represents the absence of a connection.)
2. **LISTEN** — the server has called `listen()` and is waiting for incoming connections. This is where servers spend most of their time.
3. **SYN_SENT** — the client has sent a SYN and is waiting for the server's SYN+ACK response. Entered after `connect()`.
4. **SYN_RCVD** — the server has received the client's SYN and sent a SYN+ACK back. Waiting for the final ACK. (Connections that stop here are called "half-open" and will time out.)
5. **ESTABLISHED** — the connection is open. Both sides can send and receive data. This is the normal operating state.
6. **FIN_WAIT_1** — the active closer has sent FIN. Waiting for ACK or FIN+ACK from the other side.
7. **FIN_WAIT_2** — received ACK of our FIN; still waiting for the other side's FIN. The initiator can no longer send data but can still receive.
8. **CLOSE_WAIT** — received FIN from the other side; sent ACK. The application has not yet called `close()`. This state can pile up if the server application is buggy (the "CLOSE_WAIT leak").
9. **CLOSING** — both sides sent FIN simultaneously (rare — a simultaneous close).
10. **LAST_ACK** — the passive closer has called `close()`, sent its own FIN, and is waiting for the final ACK.
11. **TIME_WAIT** — the active closer has sent the final ACK. The connection waits **2×MSL** (Maximum Segment Lifetime, typically 2 minutes) before becoming CLOSED. This lets old duplicate segments expire and allows the final ACK to be retransmitted if lost.

**Three key transitions you must draw:**
```
ESTABLISHED ──── (active close) ────▶ FIN_WAIT_1 ────▶ FIN_WAIT_2 ────▶ TIME_WAIT ────▶ CLOSED
ESTABLISHED ──── (passive close) ───▶ CLOSE_WAIT ────▶ LAST_ACK ────▶ CLOSED
ESTABLISHED ──── (both close) ──────▶ CLOSING ────▶ TIME_WAIT ────▶ CLOSED
```

**④ Example/code**
```bash
$ netstat -tan | head
tcp4  0  0  192.168.1.1.22  192.168.1.2.54321  ESTABLISHED
tcp4  0  0  127.0.0.1.8080  127.0.0.1.60123    TIME_WAIT
tcp4  0  0  *.*80            *.*                LISTEN
```
- A restarting server that reports "Address already in use" has a socket stuck in **TIME_WAIT**. The fix: `setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));` *before* `bind()`.

**⑤ Common errors/limits**
- Confusing `CLOSE_WAIT` with `FIN_WAIT_2`: `CLOSE_WAIT` means **we received FIN but haven't closed** (the application is at fault); `FIN_WAIT_2` means **we sent FIN and received ACK, waiting for their FIN** (the other side is slow).
- `TIME_WAIT` is **not a bug** — it is required for safe port reuse. It protects against old duplicate segments from the old connection contaminating a new one.
- On a high-traffic server, many TIME_WAIT sockets can consume ports; solutions: SO_REUSEADDR, a reverse proxy, or increasing the ephemeral port range.

**⑥ Conclusion** — The 11-state diagram is one of the most important things to memorise for this exam: it shows exactly how a TCP connection is established (through ESTABLISHED) and terminated (through TIME_WAIT or LAST_ACK) and explains what happens during the four-way close.

---

### M4. Value-Result Arguments  [🔴★ NCIT 2025 Q3a, Gandaki Q2b]

**① Definition** — A **value-result argument** is a function parameter that is a **value** (input) when the function is called and becomes a **result** (output) when the function returns. In socket programming this applies to the **address length** in functions where the **kernel writes** a socket address into the caller's buffer.

**② Diagram**
```
 TWO DIRECTIONS OF ADDRESS LENGTH:

 Process → Kernel (bind, connect, sendto):
 ┌──────────┐    addr pointer + integer size    ┌──────────┐
 │ process  │ ───────────────────────────────▶  │  kernel  │
 │ (caller) │   kernel reads the size only      │          │
 └──────────┘   (how big is my buffer?)          └──────────┘

 Kernel → Process (accept, recvfrom, getsockname, getpeername):
 ┌──────────┐   addr pointer + pointer-to-size  ┌──────────┐
 │ process  │ ◀──────────────────────────────── │  kernel  │
 │ (caller) │   size was a VALUE (buffer size)   │          │
 │          │   now it's a RESULT (bytes stored) │          │
 └──────────┘                                    └──────────┘
```

**③ Full concept**
- When a function like `bind()` or `connect()` sends a socket address **from the process to the kernel**, the kernel reads the address structure and its size. The size is simply passed **by value** as an integer — the kernel knows exactly how many bytes to copy. The length does not change.
- When a function like `accept()`, `recvfrom()`, `getsockname()` or `getpeername()` returns a socket address **from the kernel to the process**, the parameter is a **pointer to** the size (type `socklen_t *`). On **input** it holds the buffer size (telling the kernel "don't write more than this many bytes"). On **output** the kernel **updates it** to the actual number of bytes stored. This is the value-result pattern.
- This design is necessary because the kernel must not write past the end of the caller's buffer (buffer overflow protection), and the caller needs to know the real size of the address stored (especially for variable-length addresses like Unix domain socket paths).

**④ Example/code**
```c
struct sockaddr_in cli;
socklen_t len;
len = sizeof(cli);                               // ← VALUE: "I have 16 bytes of space"
getpeername(fd, (SA*)&cli, &len);               // ← on return, len = RESULT: "I wrote 16 bytes"
// You now know the client's IP and port are in cli.

// getsockname: learn the ephemeral port assigned to a socket
struct sockaddr_in local;
socklen_t locallen = sizeof(local);
getsockname(connfd, (SA*)&local, &locallen);
printf("Assigned port: %d\n", ntohs(local.sin_port));
```

**⑤ Common errors/limits**
- **Forgetting to initialise `len` before the call** — if `len` contains garbage, the kernel may truncate the address or copy too many bytes. Always set `len = sizeof(struct sockaddr_in)` (or `sizeof(sockaddr_in6)`).
- For fixed-size structures (IPv4 = 16 bytes, IPv6 = 28 bytes) the returned length is always the fixed size. For variable-size `sockaddr_un` (Unix domain, up to 104-byte pathname) the returned length can be smaller — this tells you the actual pathname length.
- The kernel **truncates** the address if your buffer is too small, rather than failing — so the returned length is the only reliable way to know if truncation happened.

**⑥ Conclusion** — The value-result length is how all kernel→process socket functions safely copy addresses into caller buffers while reporting the true size; you see it in every `accept`, `recvfrom`, `getsockname`, and `getpeername` call.

---

### M5. Socket Address Structures  [🔴★ NCIT Q2b, Gandaki Q2a/Q2b]

**① Definition** — A **socket address structure** describes **where a process can be reached** on the network: it contains an address family (IPv4/IPv6/Unix), a port number, and an IP address (or path for Unix). Every socket function takes a pointer to one of these structures.

**② Diagram — the four main structures**
```
 struct sockaddr            struct sockaddr_in       struct sockaddr_in6       struct sockaddr_un
 (generic cast target)      (IPv4, 16 bytes)        (IPv6, 28 bytes)         (Unix domain)
 ┌──────────────────┐      ┌──────────────────┐     ┌──────────────────┐      ┌──────────────────┐
 │ sa_family (2B)   │      │ sin_family  AF_INET│    │sin6_family AF_INET6│    │sun_family AF_UNIX│
 │ sa_data[14]      │      │ sin_port (2B)    │     │sin6_port (2B)    │     │sun_path[104]     │
 │  (protocol data) │      │ sin_addr (4B)    │     │sin6_flowinfo (4B)│     │ (null-terminated │
 └──────────────────┘      │ sin_zero[8]      │     │sin6_addr (16B)   │     │  pathname)       │
                           └──────────────────┘     │sin6_scope_id(4B) │     └──────────────────┘
                                                    └──────────────────┘

 struct sockaddr_storage (generic, ≥128 bytes, strict alignment — holds any address type)
```

**③ Full concept**
- **`struct sockaddr_in`** (IPv4, 16 bytes): contains `sin_family` = AF_INET, `sin_port` = 16-bit TCP/UDP port (network byte order), `sin_addr` = 32-bit IPv4 address (network byte order), `sin_zero[8]` = padding to align with `struct sockaddr`.
- **`struct sockaddr`** (generic, 16 bytes): the common "container" that all socket functions accept. You do **not fill this in directly** — you fill in `sockaddr_in` and then **cast** it: `bind(s, (struct sockaddr*)&servaddr, sizeof(servaddr))`.
- **`struct sockaddr_in6`** (IPv6, 28 bytes): has `sin6_addr` (128-bit IPv6 address), `sin6_port`, `sin6_flowinfo` (traffic class/flow label), `sin6_scope_id` (for link-local addresses).
- **`struct sockaddr_un`** (Unix domain): contains a `sun_path` (null-terminated pathname like `/tmp/mysock`) for same-host IPC.
- **`struct sockaddr_storage`**: a large (≥128-byte), properly aligned structure that can hold **any** address type. Use it when you don't know at compile time whether IPv4 or IPv6 addresses will arrive (e.g., a dual-stack server).

**Key rule:** `sin_port` and `sin_addr.s_addr` **must always be in network byte order** (big-endian). Use `htons()` for ports and `htonl()` for addresses:
```c
servaddr.sin_port = htons(8080);                        // host → network
servaddr.sin_addr.s_addr = htonl(INADDR_ANY);           // accept on any interface
```

**④ Example/code**
```c
struct sockaddr_in serv;
memset(&serv, 0, sizeof(serv));      // zero out the entire struct (including sin_zero)
serv.sin_family = AF_INET;
serv.sin_port = htons(80);           // HTTP port 80, in network byte order
serv.sin_addr.s_addr = htonl(INADDR_ANY);  // bind to all local interfaces

bind(listenfd, (struct sockaddr*)&serv, sizeof(serv));

// The cast from sockaddr_in to sockaddr is required because
// bind() takes a generic struct sockaddr pointer.
```

**⑤ Common errors/limits**
- **Forgetting `htons`/`htonl`**: port 80 stored as 0x0050 in host order on a little-endian machine becomes 0x5000 (port 20480) in network order — the server silently listens on the wrong port.
- **Not zeroing the struct first** (`memset(&serv, 0, sizeof(serv))`): `sin_zero[8]` may contain garbage, which can confuse some implementations.
- Using `sizeof(struct sockaddr)` instead of `sizeof(struct sockaddr_in)` in `bind()` — this works by coincidence (both are 16 bytes on many systems) but is technically wrong.

**⑥ Conclusion** — The socket address structures are the foundation every socket function relies on; the key exam points are the four variants, the casting from specific→generic, and the byte-order requirement for port and address.

---

### M6. The Five I/O Models  [🔴★ Gandaki Q4a]

**① Definition** — Every network input operation has **two phases**: (1) **waiting for data to become ready** (data arrives from the network into the kernel buffer) and (2) **copying data** from the kernel buffer into the application's buffer. The **five I/O models** are five different ways a process handles these two phases.

**② Diagram — the five models**
```
 PHASE 1 (wait)          PHASE 2 (copy)          MODEL
 ──────────────────────  ──────────────────────  ──────────────────────

 ┌ BLOCKING I/O ──────────────────────────────────────────────────────┐
 │ Process sleeps          Kernel copies          process blocked       │
 │ until data ready  ───▶ into app buffer  ───▶ until copy done       │
 │ (recvfrom blocks)      (recvfrom still         returns data         │
 │                         blocks in Phase 2)                         │
 └────────────────────────────────────────────────────────────────────┘

 ┌ NONBLOCKING I/O ───────────────────────────────────────────────────┐
 │ Kernel returns         Process polls           process wastes CPU   │
 │ EWOULDBLOCK if    ───▶ (calls recvfrom   ───▶ checking constantly │
 │ not ready, doesn't     again and again)      until data is ready  │
 │ block the process                                                 │
 └────────────────────────────────────────────────────────────────────┘

 ┌ I/O MULTIPLEXING (select/poll) ────────────────────────────────────┐
 │ Process blocks in      select returns,        still must call      │
 │ select() waiting  ───▶ telling which fd  ───▶ recvfrom to copy    │
 │ for any fd ready        is ready              (but it won't block) │
 └────────────────────────────────────────────────────────────────────┘

 ┌ SIGNAL-DRIVEN I/O (SIGIO) ────────────────────────────────────────┐
 │ Kernel sends SIGIO     Handler calls          both phases happen   │
 │ when fd is ready ───▶ recvfrom to   ───▶ in the handler            │
 │ (no blocking)           read data                                   │
 └────────────────────────────────────────────────────────────────────┘

 ┌ ASYNCHRONOUS I/O (POSIX aio_*) ───────────────────────────────────┐
 │ Kernel starts the      Kernel copies BOTH     notification when    │
 │ entire read       ───▶ phases (wait+copy) ───▶ WHOLE thing is done│
 │ (app returns           in background          (data already in     │
 │  immediately)                                   app buffer)        │
 └────────────────────────────────────────────────────────────────────┘
```

**③ Full concept**
- **Blocking I/O** (default): `recvfrom()` puts the process to sleep until data arrives AND is copied. Simple but ties up a thread.
- **Non-blocking I/O**: the kernel returns `EWOULDBLOCK` instead of sleeping. The process must **poll** (keep calling `recvfrom` in a loop) — this wastes CPU.
- **I/O Multiplexing (select/poll)**: the process blocks in `select()` watching **many** descriptors; when any becomes readable, `select` returns and the process calls `recvfrom`. Two system calls but the thread is shared among many connections.
- **Signal-driven I/O (SIGIO)**: the kernel notifies the process with a **signal** when the descriptor is ready. The process is free to do other work and only interrupted when I/O is possible.
- **Asynchronous I/O (POSIX `aio_*`)**: the kernel does the **entire operation** (wait + copy) in the background. The process is notified only when the operation is **complete** — data is already in the app buffer.

**The key distinction examiners test:** the first four models are all **synchronous** (the `recvfrom` call blocks). Only model 5 (asynchronous I/O) is **truly asynchronous** — the kernel does everything and the process is never blocked.

| Model | Blocks during recvfrom? | Truly async? | Typical use |
|---|---|---|---|
| Blocking | **Yes** (sleeps) | No | simple client |
| Non-blocking | No (polls) | No | dedicated single-task systems |
| Multiplexing | **Yes** (but on select, not recv) | No | servers with many connections |
| Signal-driven | **Yes** (in handler) | No | event-driven systems |
| Async | **No** (kernel copies in background) | **Yes** | high-throughput systems |

**④ Example/code**
```c
// Blocking I/O
n = recvfrom(sockfd, buf, MAXLINE, 0, NULL, NULL);  // sleeps until data arrives

// Non-blocking I/O
int flags = fcntl(sockfd, F_GETFL, 0);
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);         // make non-blocking
n = recvfrom(sockfd, buf, MAXLINE, 0, NULL, NULL);  // returns immediately
if (n == -1 && errno == EWOULDBLOCK) { /* no data yet */ }

// I/O multiplexing
fd_set rset;
FD_ZERO(&rset); FD_SET(sockfd, &rset);
select(sockfd+1, &rset, NULL, NULL, NULL);           // blocks until sockfd ready
n = recvfrom(sockfd, buf, MAXLINE, 0, NULL, NULL);   // guaranteed to return quickly
```

**⑤ Common errors/limits**
- Thinking non-blocking I/O "fixes" blocking: it doesn't — you must either poll (wasteful) or combine with `select`/events.
- Confusing signal-driven ("you can start I/O now") with async ("I/O is done"). Signal-driven says the fd is *ready*; async says the operation is *complete*.
- POSIX AIO is not widely implemented on all UNIX systems (Linux has `io_uring` as a more modern alternative).

**⑥ Conclusion** — All five models serve the same purpose (getting data from the kernel to the app) but differ in how they handle the two phases; the trade-off is between simplicity, CPU usage, thread usage and programming complexity.

---

### M7. Mechanisms to Handle Multiple Clients (fork / select / threads)  [🔴★ Gandaki Q3b, NCIT Q5a]

**① Definition** — A **concurrent server** handles **many clients simultaneously**, as opposed to an iterative server which handles one client at a time. There are three main mechanisms in UNIX: **forking a child per client**, **using select/poll to multiplex**, and **using threads**.

**② Diagram — the three approaches**
```
Approach 1: FORK PER CLIENT

  MAIN SERVER PROCESS
  ┌──────────────────────────┐
  │ listen();                │
  │ loop {                   │
  │   connfd = accept();     │
  │   fork(); ──────────────────────────▶ CHILD PROCESS
  │   │                      │            close(listenfd);
  │   │                      │            serve_client(connfd);
  │   close(connfd);         │            close(connfd); exit(0);
  │ }                        │
  └──────────────────────────┘

Approach 2: SELECT MULTIPLEXING

  SINGLE PROCESS
  ┌──────────────────────────────────────────────────┐
  │ select(listenfd, client1, client2, ..., timeout)  │
  │   ├─ listenfd ready?  → accept new client         │
  │   ├─ client1 ready?   → handle client1            │
  │   ├─ client2 ready?   → handle client2            │
  │   └─ ...                                          │
  └──────────────────────────────────────────────────┘

Approach 3: THREADS (pthread_create per client)

  MAIN THREAD                    WORKER THREADS
  ┌────────────────────┐        ┌──────────────────────┐
  │ loop {             │        │ serve_client(connfd)  │
  │   connfd=accept(); │──clone─┤ (shares memory with   │
  │   pthread_create() │        │  main — use mutex)    │
  │ }                  │        └──────────────────────┘
  └────────────────────┘
```

**③ Full concept**
- **Fork per client**: after `accept()`, the server calls `fork()`. The child inherits the connected socket and handles that client, then exits. The parent closes the connected socket and loops back to accept the next client. After `fork()`, parent and child **share the connected socket descriptor**, so the parent must close its copy and the child must close the listening socket copy. The parent must handle **SIGCHLD** to reap zombie children (`waitpid(-1, NULL, WNOHANG)` in the handler).
- **Select/poll multiplexing**: a single process watches **all** descriptors (listening + connected) using `select()`. When any becomes readable, the process handles only that one. This is single-threaded and avoids the overhead of forking.
- **Threads (pthread)**: instead of forking (which copies the entire process), create a **lightweight thread** (`pthread_create`) that shares the same address space. Threads are faster to create than processes but need **mutexes** to protect shared data. A **thread pool** (pre-creating N worker threads) avoids the per-request creation cost.

**④ Example/code — fork-per-client server:**
```c
void sig_chld(int signo) { pid_t pid; while ((pid = waitpid(-1, NULL, WNOHANG)) > 0); }

int main() {
    signal(SIGCHLD, sig_chld);  // reap zombie children
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    bind(listenfd, ...); listen(listenfd, LISTENQ);
    for (;;) {
        connfd = accept(listenfd, (SA*)&cliaddr, &clilen);
        if ((pid = fork()) == 0) {       // child process
            close(listenfd);             // child doesn't need listener
            doit(connfd);                // handle this one client
            close(connfd);
            exit(0);
        }
        close(connfd);                   // parent closes connected socket
    }
}
```

**⑤ Common errors/limits**
- **Zombie children**: if the parent never calls `waitpid`, dead children remain in the process table. Always handle `SIGCHLD`.
- **Fork overhead**: forking copies the entire process (expensive at high scale); select or a thread pool is better for thousands of clients.
- **Thread safety**: threads share memory — you must protect shared data (connection lists, counters) with mutexes. Fork gives isolation by default.
- **select limit**: `FD_SETSIZE` (typically 1024) limits how many descriptors `select` can monitor; `poll` or `epoll` avoids this.

**⑥ Conclusion** — Fork gives isolation and is simple, select is efficient for a single process, and threads give lightweight concurrency — choose based on your scale and isolation needs, and always reap children / synchronise shared data.

---

### M8. Socket Options (SO_REUSEADDR, SO_BROADCAST, SO_KEEPALIVE, SO_LINGER)  [🔴★ NCIT Q5b, Gandaki Q4b]

**① Definition** — **Socket options** let you customise a socket's behaviour without changing the protocol. They are set and queried using **`setsockopt()`/`getsockopt()`** at levels like `SOL_SOCKET` (generic) or `IPPROTO_TCP` (TCP-specific).

**② Diagram**
```
  ┌─────────────┐       setsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &on, sizeof(on))
  │ application │ ─────────────────────────────────────────────────────────────────▶ kernel socket
  │             │ ◀───────────────────────────────────────────────────────────────── kernel socket
  └─────────────┘       getsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &val, &len)

  setsockopt prototype:
  int setsockopt(int sockfd, int level, int optname,
                 const void *optval, socklen_t optlen);

  getsockopt prototype:
  int getsockopt(int sockfd, int level, int optname,
                 void *optval, socklen_t *optlen);
```

**③ Full concept — each option in detail:**
- **SO_REUSEADDR**: allows a socket to **bind to an address/port that is already in use**, specifically when the old socket is stuck in **TIME_WAIT**. Without this, a server that crashes and restarts gets "Address already in use" and cannot rebind. Set it **before** `bind()`. (On Linux, SO_REUSEPORT allows multiple sockets to bind the same port for load balancing.)

- **SO_BROADCAST**: enables a UDP socket to send to **broadcast addresses** (e.g., 192.168.1.255 or 255.255.255.255). By default, broadcast is disabled — if you try, you get `EACCES`. Routers do not forward broadcast packets, so it is limited to the local subnet.

- **SO_KEEPALIVE**: enables TCP **keep-alive probes** to detect dead peers. If the connection is idle for **2 hours**, TCP sends a probe. If the peer responds with ACK, the connection is kept alive. If the peer sends RST, the connection is dead (`ECONNRESET`). If no response: TCP sends 8 probes, 75 seconds apart (~11 minutes total), then returns `ETIMEDOUT`. Useful for long-lived connections (SSH, database connections).

- **SO_LINGER**: controls what happens when you call `close()` on a socket with unsent data:
  ```
  struct linger { int l_onoff; int l_linger; };

  l_onoff=0      → close() returns immediately (default; data sent in background)
  l_onoff≠0, l_linger=0   → TCP ABORTS: discards send buffer, sends RST (immediate close, no FIN)
  l_onoff≠0, l_linger≠0   → close() BLOCKS (lingers) until data is sent+ACKed or timeout expires
  ```

**④ Example/code**
```c
// Enable SO_REUSEADDR before bind (so server can restart immediately)
int on = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
bind(listenfd, (SA*)&servaddr, sizeof(servaddr));

// Enable SO_BROADCAST for UDP broadcast
setsockopt(udpfd, SOL_SOCKET, SO_BROADCAST, &on, sizeof(on));
sendto(udpfd, msg, strlen(msg), 0, &bcast_addr, sizeof(bcast_addr));

// Enable SO_KEEPALIVE to detect dead peers
setsockopt(connfd, SOL_SOCKET, SO_KEEPALIVE, &on, sizeof(on));

// SO_LINGER: linger up to 5 seconds
struct linger li = {1, 5};  // l_onoff=1, l_linger=5
setsockopt(connfd, SOL_SOCKET, SO_LINGER, &li, sizeof(li));
```

**⑤ Common errors/limits**
- **SO_REUSEADDR on Linux** does not let two different processes bind the same port for a new listener; that requires **SO_REUSEPORT** (which also requires both sockets to have SO_REUSEPORT set). On BSD, SO_REUSEADDR allows port rebinding after TIME_WAIT, which is what servers need.
- **SO_KEEPALIVE's default 2-hour idle time** is too long for many applications; you must adjust it at the TCP level (`TCP_KEEPIDLE`) if you want shorter detection.
- **SO_LINGER with a timeout blocks `close()`** — if the peer is slow or unreachable, `close()` can hang for up to `l_linger` seconds, which may stall your server.
- Set buffer options (`SO_RCVBUF`, `SO_SNDBUF`) **before** `connect()` or `listen()`, not after.

**⑥ Conclusion** — These four options solve the most common practical server problems: restart without port conflict (SO_REUSEADDR), send to all hosts (SO_BROADCAST), detect dead peers (SO_KEEPALIVE), and control graceful vs abrupt close (SO_LINGER). They are a staple exam topic.

---

### M9. How is Winsock different from UNIX sockets? + static vs dynamic linking  [🔴★ NCIT Q6a]

**① Definition** — **Winsock (Windows Sockets)** is the Windows implementation of the BSD/Berkeley socket API, providing the same socket programming model but adapted to the Windows environment with its own setup, types, error handling and I/O models.

**② Diagram**
```
UNIX model:                         Windows (Winsock) model:
┌────────────────┐                  ┌────────────────┐
│  Application   │                  │  Application   │
├────────────────┤                  ├────────────────┤
│ socket(), bind │  system calls    │ socket(), bind │  → ws2_32.dll
│ listen(), etc. │  (kernel)        │ listen(), etc. │  (user-mode library)
├────────────────┤                  ├────────────────┤
│     kernel     │                  │  Windows kernel │
└────────────────┘                  └────────────────┘
                                    (loaded dynamically via DLL at run time)
```

| Feature | Unix / Berkeley | Winsock |
|---|---|---|
| Socket type | `int` (file descriptor, e.g. 3, 4, 5) | `SOCKET` (handle, opaque type) |
| Close a socket | `close(fd)` | `closesocket(s)` |
| Error reporting | global variable `errno` | `WSAGetLastError()` function |
| Read/write | `read()`/`write()` (also `send`/`recv`) | `send()`/`recv()` only (no `read`/`write`) |
| Setup before use? | None needed | **Must call `WSAStartup()` first, `WSACleanup()` after** |
| Address structure | `struct sockaddr_in` (same semantics) | `SOCKADDR_IN` (same semantics) |
| Include header | `<sys/socket.h>`, `<netinet/in.h>` | `<winsock2.h>` + link `ws2_32.lib` |
| Extra I/O models | select, poll, epoll, kqueue | WSAAsyncSelect, WSAEventSelect, overlapped, IOCP |

**③ Full concept** — **Why `WSAStartup()`?** Windows loads network protocol modules as **DLLs** (Dynamic Link Libraries) — the program doesn't call the kernel directly for socket functions; it loads a DLL that provides the Winsock API. `WSAStartup()` loads `ws2_32.dll` and negotiates the version. Unix uses kernel system calls, so no DLL loading is needed.

**Static vs dynamic linking:**
- **Dynamic linking (DLL)**: the library code lives in a separate `.dll` file loaded at run time. The `.exe` does not contain the library code — it calls into the DLL. **Advantages:** smaller executable, easy to update (replace the DLL, all programs pick up the fix), code is shared across programs. **Disadvantages:** if the DLL is missing, wrong version, or corrupted, the program may fail to start ("dependency problem" / "DLL hell").
- **Static linking**: the library code is copied into the `.exe` at compile time. **Advantages:** no external dependency, always runs. **Disadvantages:** larger executable, must recompile to update the library code.

**④ Example/code**
```c
// Winsock — typical setup and teardown
WSADATA wsaData;
int result = WSAStartup(MAKEWORD(2,2), &wsaData);  // load ws2_32.dll, request v2.2
if (result != 0) { /* handle error */ }

SOCKET s = socket(AF_INET, SOCK_STREAM, 0);  // create TCP socket

// ... use socket (bind, listen, accept, send, recv) ...

closesocket(s);     // close the socket (not close())
WSACleanup();       // unload the library (must match every WSAStartup)
```

**⑤ Common errors/limits**
- Forgetting `WSAStartup()` before socket calls: every function returns `WSANOTINITIALISED`.
- Calling `close()` instead of `closesocket()` on Windows — the function does not exist in Winsock.
- Not matching every `WSAStartup()` with a `WSACleanup()` (reference counting — leaks resources).

**⑥ Conclusion** — Winsock reuses the Berkeley socket model but adapts it for Windows (SOCKET type, closesocket, WSAGetLastError, WSAStartup/Cleanup, extra async I/O models); Unix code ports to Windows with small `#ifdef` wrappers and DLL linking.

---

### M10. Winsock TCP and UDP Client-Server Sequences with Code  [🔴★ Gandaki Q5b]

**① Definition** — Winsock client and server applications follow the same call sequences as Unix sockets, wrapped with `WSAStartup()` at the beginning and `WSACleanup()` at the end. TCP uses connection-oriented calls (listen/accept/connect); UDP uses connectionless calls (sendto/recvfrom).

**② Diagram — the four call sequences**
```
TCP SERVER:                          TCP CLIENT:
WSAStartup()                         WSAStartup()
  socket()                             socket()
  bind()                               (bind optional — kernel assigns port)
  listen()                             connect()     ← three-way handshake happens here
  accept()  ←── waiting for client     send() / recv()
  send() / recv()                      closesocket()
  closesocket()                       WSACleanup()
  WSACleanup()

UDP RECEIVER:                        UDP SENDER:
WSAStartup()                         WSAStartup()
  socket()                             socket()
  bind()                               sendto()  ← no connect needed
  recvfrom()                           closesocket()
  closesocket()                       WSACleanup()
  WSACleanup()
```

**③ Full concept**
- **TCP Server**: after `WSAStartup`, create a socket, `bind` to a local port, `listen` to queue connections, then `accept` in a loop — each call to `accept` returns a **new** socket for a specific client. Data flows with `send`/`recv`. Errors come from `WSAGetLastError()`.
- **TCP Client**: `WSAStartup`, create a socket, `connect` to the server's IP:port (this triggers the three-way handshake — no `bind` or `listen` needed). Data flows with `send`/`recv`.
- **UDP Receiver**: `WSAStartup`, create a datagram socket, `bind` to a port, then `recvfrom` in a loop — each `recvfrom` also learns the sender's address.
- **UDP Sender**: `WSAStartup`, create a datagram socket, `sendto` each message with the destination address — no `connect`, `listen`, or `accept` needed.
- **Graceful close**: `shutdown(s, SD_SEND)` sends a TCP FIN (no more data), then `closesocket(s)` fully releases the socket.

**④ Example/code — Winsock TCP server (complete, minimal):**
```c
#pragma comment(lib, "ws2_32.lib")  // link ws2_32.dll

int main() {
    WSADATA w; WSAStartup(MAKEWORD(2,2), &w);

    SOCKET s = socket(AF_INET, SOCK_STREAM, 0);
    SOCKADDR_IN sa; sa.sin_family=AF_INET;
    sa.sin_port=htons(5150); sa.sin_addr.s_addr=htonl(INADDR_ANY);
    bind(s, (SOCKADDR*)&sa, sizeof(sa));
    listen(s, 5);

    SOCKADDR_IN cli; int clen=sizeof(cli);
    SOCKET cs = accept(s, (SOCKADDR*)&cli, &clen);  // blocks until client connects

    char buf[1024]; int n = recv(cs, buf, sizeof(buf), 0);
    send(cs, buf, n, 0);  // echo back

    closesocket(cs); closesocket(s);
    WSACleanup();
    return 0;
}
```

**⑤ Common errors/limits**
- Using `close()` instead of `closesocket()` — undefined on Windows.
- Not calling `WSAStartup()` — every socket function fails with `WSANOTINITIALISED`.
- Blocking `recv()` on the server stalls all other clients — use non-blocking I/O or select/events for concurrency.

**⑥ Conclusion** — Winsock sequences mirror Berkeley exactly (except WSAStartup/WSACleanup and closesocket); memorise both TCP and UDP orders for full marks.

---

### M11. Overlapped I/O in Winsock  [🔴★ NCIT Q7a]

**① Definition** — **Overlapped I/O** is Winsock's mechanism for issuing **multiple I/O operations simultaneously** without blocking the calling thread. Each operation completes in the background and the application is notified via an **event object** or a **completion routine** (callback function). It is the highest-performance single-socket I/O model in Winsock.

**② Diagram**
```
  APP: post WSARecv(s1, buf1, &ovl1)  ──▶  kernel runs it in background
      post WSARecv(s2, buf2, &ovl2)  ──▶  kernel runs it in background
      post WSASend(s1, data, &ovl3)  ──▶  kernel runs it in background
          (thread is free to do other work)
      ┌──────────────────────────────────────────────┐
      │  option A: event objects                       │
      │  ovl1.hEvent signals → WSAWaitForMultipleEvents │
      │                                                 │
      │  option B: completion routine (callback)        │
      │  when ovl1 finishes → Winsock calls your fn     │
      └──────────────────────────────────────────────┘
```

**③ Full concept**
- The socket must be created with the **`WSA_FLAG_OVERLAPPED`** flag: `WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED)`.
- Use `WSASend`, `WSARecv`, `WSARecvFrom`, `WSASendTo`, `WSAIoctl`, `AcceptEx`, `TransmitFile` — all accept a `WSAOVERLAPPED` structure.
- If the call returns `SOCKET_ERROR` and `WSAGetLastError() == WSA_IO_PENDING`, the operation has been **queued** — it is not an error; it will complete later.
- **Completion method 1 (event object)**: put an event in `ovl.hEvent`; after posting operations, call `WSAWaitForMultipleEvents(n, events, ...)` to wait; then call `WSAGetOverlappedResult()` to check which operation completed.
- **Completion method 2 (completion routine)**: pass a callback function; when the operation finishes, Winsock calls your function automatically. The thread must be in **alertable wait** (`SleepEx`, `WaitForSingleObjectEx`, etc.) for the callback to fire.
- **Advantage**: one thread manages many outstanding I/O operations, giving much higher throughput than sequential blocking calls.

**④ Example/code**
```c
WSAOVERLAPPED ovl = {0};
WSABUF buf = {len, data};
DWORD flags = 0, nrecv = 0;
ovl.hEvent = WSACreateEvent();

WSARecv(s, &buf, 1, &nrecv, &flags, &ovl, NULL);
if (WSAGetLastError() == WSA_IO_PENDING) {
    // operation queued — wait for completion
    WSAWaitForMultipleEvents(1, &ovl.hEvent, FALSE, WSA_INFINITE, FALSE);
    DWORD bytes;
    WSAGetOverlappedResult(s, &ovl, &bytes, FALSE, &flags);
    // bytes now contains how many were received
}
```

**⑤ Common errors/limits**
- Treating `WSA_IO_PENDING` as a failure — it is normal and expected.
- Reusing the `WSAOVERLAPPED` structure or buffer before the previous operation completes — data race / corruption.
- For the completion-routine method, the thread must be in **alertable wait**; otherwise the callback never fires.
- Overlapped I/O is per-socket — you need to track which socket/overlapped structure belongs to which completion.

**⑥ Conclusion** — Overlapped I/O lets a single thread manage many concurrent I/O operations through background processing and completion notification; it is the foundation for the even more powerful IOCP (I/O Completion Ports) model used in high-scale servers.

---

### M12. HTTP vs WebSocket + Simple Server  [🔴★ NCIT Q7b]

**① Definition** — HTTP is a **request-response** protocol: the client asks, the server answers. WebSocket is a **full-duplex, persistent** protocol that upgrades an HTTP connection so **both sides can push messages at any time** with very low overhead.

**② Diagram**
```
 HTTP (request-response):
  CLIENT ──GET /page──▶ SERVER
  CLIENT ◀──page────── SERVER
  CLIENT ──GET /data──▶ SERVER    (new connection or keep-alive)
  CLIENT ◀──data────── SERVER
  (client must always ask; server cannot push)

 WebSocket (full-duplex):
  CLIENT ──GET /chat (Upgrade: websocket)──▶ SERVER
  CLIENT ◀──HTTP/1.1 101 Switching Protocols── SERVER
         ═══════════════════════════════════════
         CLIENT ◀──── text frame ────── SERVER    (server pushes!)
         CLIENT ── text frame ────▶ SERVER        (client sends)
         CLIENT ◀──── text frame ────── SERVER    (server pushes!)
         (both sides can send at any time, no headers)
         ═══════════════════════════════════════

 Comparison Table:
 ┌─────────────┬─────────────────────────┬───────────────────────────────┐
 │             │ HTTP                    │ WebSocket                     │
 ├─────────────┼─────────────────────────┼───────────────────────────────┤
 │ Direction   │ request-response        │ full-duplex (both push)       │
 │ Connection  │ closed after response   │ persistent (stays open)       │
 │ Overhead    │ headers repeated each   │ tiny frames (~2-10 bytes      │
 │             │ request (~800 bytes)    │ header)                       │
 │ Server push │ not possible            │ yes (main advantage)          │
 │ Use cases   │ web pages, APIs, forms  │ chat, gaming, live dashboards │
 │ URI scheme  │ http:// / https://      │ ws:// / wss:// (encrypted)    │
 └─────────────┴─────────────────────────┴───────────────────────────────┘
```

**③ Full concept**
- HTTP is great for fetching web pages, but every message repeats the full HTTP headers (cookies, user-agent, content-type — hundreds of bytes each). For real-time apps (chat, multiplayer games, live stock tickers) this overhead is unacceptable and the server cannot push updates on its own.
- WebSocket fixes this: after a one-time **HTTP Upgrade handshake**, the connection stays open and both sides exchange **small frames** (~2-10 byte headers) with no repeated headers. Either side can push at any time.
- The **handshake**: client sends `GET /path HTTP/1.1` with headers `Upgrade: websocket`, `Connection: Upgrade`, `Sec-WebSocket-Key: <base64 random>`. Server replies `HTTP/1.1 101 Switching Protocols` with `Sec-WebSocket-Accept: <hash of key>`. After that, the connection is WebSocket.
- After the handshake, data travels in **frames**: each frame has a **FIN** bit (is this the last frame?), an **opcode** (0x1=text, 0x2=binary, 0x8=close, 0x9=ping, 0xA=pong), a **MASK** bit (must be 1 for client→server to prevent cache-poisoning), and a **payload length** (7-bit; if 126 → next 2 bytes; if 127 → next 8 bytes).
- **Ping/Pong**: either side sends a ping frame; the other must reply with pong (same payload). This keeps idle connections alive and detects dead peers.
- **Close**: a close frame carries a **close code** (1000=normal, 1001=going away, 1008=policy violation, 1011=server error).

**④ Example — simple WebSocket server (pseudo-code):**
```c
// 1. Normal TCP server setup
listenfd = socket(AF_INET, SOCK_STREAM, 0);
bind(listenfd, ...); listen(listenfd, 5);

// 2. Accept a client
connfd = accept(listenfd, ...);

// 3. Read the HTTP Upgrade request
read(connfd, buf, sizeof(buf));  // contains "GET /chat HTTP/1.1\r\nUpgrade: websocket\r\n..."

// 4. Verify and reply with 101
snprintf(response, sizeof(response),
    "HTTP/1.1 101 Switching Protocols\r\n"
    "Upgrade: websocket\r\n"
    "Connection: Upgrade\r\n"
    "Sec-WebSocket-Accept: %s\r\n\r\n", computed_accept_key);
write(connfd, response, strlen(response));

// 5. Now read/write WebSocket frames
while (1) {
    n = read_frame(connfd, &opcode, payload);  // reads one WS frame
    if (opcode == 0x8) break;                   // close frame
    if (opcode == 0x9) send_pong(connfd, payload);  // ping → pong
    if (opcode == 0x1) broadcast_to_all(connfd, payload);  // text → echo to all
}
```

**⑤ Common errors/limits**
- Forgetting the `Sec-WebSocket-Key`/`Accept` exchange — the handshake fails silently.
- Not masking client→server frames (mandatory per RFC 6455).
- Not handling ping/pong — idle connections may be killed by proxies/load balancers after ~60 seconds.
- Concurrently serving many WebSocket connections requires `select`/`poll`/async I/O on the server side, just like any other socket.

**⑥ Conclusion** — WebSocket replaces HTTP's one-way request-response with a persistent, low-overhead, full-duplex channel, making it the standard for real-time push applications (chat, gaming, live dashboards); the 101-upgrade handshake is the key transition point.

---

### M13. SDN: Concept, Architecture and Advantages  [🔴★ NCIT Q6b]

**① Definition** — **SDN (Software-Defined Networking)** is a network architecture that **separates the control plane** (the brain — deciding where packets go) **from the data/forwarding plane** (the muscle — actually moving packets), centralising the brain in a software **SDN controller** while leaving switches as simple, programmable devices.

**② Diagram — before SDN vs after SDN**
```
 BEFORE SDN (traditional):
 ┌──────────┐  ┌──────────┐  ┌──────────┐
 │ switch A │  │ switch B │  │ switch C │
 │ brain +  │  │ brain +  │  │ brain +  │    ← each device has its own brain
 │ muscle   │  │ muscle   │  │ muscle   │       (closed firmware, vendor-specific)
 └──────────┘  └──────────┘  └──────────┘

 AFTER SDN:
               ┌────────────────────────┐
               │    SDN CONTROLLER      │    ← one centralised brain
               │  (software program)    │       (open, programmable, full network view)
               │  computes routes,      │
               │  installs flow rules   │
               └────────────┬───────────┘
                            │ OpenFlow protocol
               ┌────────────┼───────────┐
               ▼            ▼           ▼
          ┌─────────┐  ┌─────────┐  ┌─────────┐
          │switch 1 │  │switch 2 │  │switch 3 │  ← simple forwarding devices
          │(muscle) │  │(muscle) │  │(muscle) │    (just follow flow rules)
          └─────────┘  └─────────┘  └─────────┘
```

**③ Full concept**
- In traditional networks, each router/switch has both the **control plane** (routing algorithms, policy decisions) and the **data plane** (actually forwarding packets). Each device is configured individually, with vendor-specific closed firmware. Changing the network means manually reconfiguring each device.
- SDN **centralises the control plane** in a single software **SDN controller** (e.g., OpenDaylight, ONOS, Floodlight). The controller has a **global view** of the entire network and can make optimal routing decisions. Switches become **simple, dumb forwarders** that just follow instructions (flow rules) pushed by the controller.
- The standard protocol between the controller and switches is **OpenFlow**: the controller writes rules into each switch's **flow table** — "if a packet matches pattern X (source IP, port, protocol), then action Y (forward to port 3, drop, modify header)."
- **Benefits**: (1) **centralised control** — one place to see and manage the whole network; (2) **programmability** — network behaviour is changed in software (Python/Java) without touching hardware; (3) **agility/automation** — new policies deployed in seconds, not hours; (4) **better utilisation** — controller balances load network-wide; (5) **vendor independence** — switches are generic, not locked to one vendor's firmware.

**④ Example**
- A datacenter controller detects congestion on one path and instantly reroutes all traffic through an alternative path — all by writing new flow rules via OpenFlow, without reconfiguring any switch manually.
- Campus networks use SDN to implement access control policies (e.g., "block student VLANs from the admin server") centrally rather than per-switch.

**⑤ Common errors/limits**
- **Single point of failure**: if the controller goes down, switches still forward based on existing flow rules but cannot learn new paths. Solutions: controller clustering / redundancy.
- **Controller-switch latency**: for very fast flow setup (microbursts), the time to query the controller can be too slow — switches use **proactive** flow rules (installed in advance) for common paths.
- **Flow table size**: switches have limited TCAM (flow table memory); very granular rules can exhaust it.
- **OpenFlow** is the dominant but not the only SDN protocol; P4 and others exist.

**⑥ Conclusion** — SDN decouples network intelligence from hardware, making networks programmable, agile and centrally manageable; OpenFlow is the protocol that enables this separation, and P4 extends it to the data plane.

---

### M14. TLS/SSL  [🔴 8-mark]

**① Definition** — **TLS (Transport Layer Security)**, the successor of SSL, is a security protocol layered between the application and TCP that provides **confidentiality** (encryption — no eavesdropping), **authenticity** (server/client identity — no impersonation), and **integrity** (tamper detection — no modification in transit). HTTPS is HTTP over TLS; WSS is WebSocket over TLS.

**② Diagram — where TLS sits + the handshake**
```
 Where TLS sits:
 ┌──────────────────────────────────────┐
 │  Application (HTTP, WebSocket, etc.) │
 ├──────────────────────────────────────┤
 │  TLS layer (encrypt + authenticate)  │  ← this layer
 ├──────────────────────────────────────┤
 │  TCP                                 │
 ├──────────────────────────────────────┤
 │  IP                                  │
 └──────────────────────────────────────┘

 TLS Handshake (simplified):
 CLIENT                                          SERVER
   │                                                │
   │──① ClientHello (version, ciphers, random)──▶ │
   │                                                │
   │◀──② ServerHello (chosen cipher, random) ─────│
   │◀──③ Certificate (server's public key) ──────│
   │◀──④ ServerKeyExchange (DH params, if used) ──│
   │                                                │
   │ ⑤ [client verifies cert via CA]                │
   │ ⑥ [both derive same session key]               │
   │                                                │
   │──⑦ Finished (verify handshake) ──────────────▶│
   │◀──⑧ Finished ─────────────────────────────────│
   │                                                │
   │ ════════ all data now encrypted ══════════════│
```

**③ Full concept**
- Three cryptographic building blocks:
  1. **Encryption** — **symmetric** encryption (e.g., AES-256) is fast and used for bulk data (both sides use the same key). **Asymmetric** encryption (e.g., RSA, Diffie-Hellman) is slow but used to safely establish the symmetric **session key**. Diffie-Hellman gives **forward secrecy** — even if the server's private key is later leaked, past traffic remains secret.
  2. **Hashing** — **SHA-256** (one-way digest) combined with HMAC detects **tampering**: any modification to the ciphertext changes the hash, which the receiver rejects.
  3. **Certificates** — bind a public key to an identity (e.g., `example.com`). A certificate is **digitally signed by a Certificate Authority (CA)** (e.g., Let's Encrypt, DigiCert). The client checks: (a) the CA signature is valid, (b) the hostname matches, (c) the certificate hasn't expired.

- **The handshake in detail** (7 steps):
  1. Client **ClientHello**: sends supported TLS versions, list of cipher suites (e.g., TLS_AES_256_GCM_SHA384), and a random number.
  2. Server **ServerHello**: picks a cipher suite, sends its random number.
  3. Server **Certificate**: sends its X.509 certificate chain (containing its public key).
  4. Server **KeyExchange**: sends Diffie-Hellman parameters (for forward secrecy).
  5. Client **verifies** the certificate against its trusted-CA store; if valid, authenticates the server.
  6. Both sides independently **derive the same session key** from the DH parameters and both randoms.
  7. Both sides send **Finished** messages (encrypted with the session key) to confirm the handshake succeeded.

- **TLS 1.3** (current standard): only 1 round-trip (0-RTT for resumption), only forward-secret key exchange, removed legacy ciphers.

**④ Example — OpenSSL on a socket:**
```c
// Server side
SSL_CTX *ctx = SSL_CTX_new(TLS_server_method());
SSL_CTX_use_certificate_file(ctx, "server.pem", SSL_FILETYPE_PEM);
SSL_CTX_use_PrivateKey_file(ctx, "server.key", SSL_FILETYPE_PEM);

SSL *ssl = SSL_new(ctx);
SSL_set_fd(ssl, sockfd);         // attach to existing connected socket
SSL_accept(ssl);                  // perform TLS handshake

SSL_write(ssl, "Hello", 5);      // encrypted write (instead of write())
SSL_read(ssl, buf, sizeof(buf));  // encrypted read

SSL_shutdown(ssl); SSL_free(ssl); SSL_CTX_free(ctx);

// Client side (similar)
SSL_CTX *ctx = SSL_CTX_new(TLS_client_method());
SSL *ssl = SSL_new(ctx);
SSL_set_fd(ssl, sockfd);
SSL_connect(ssl);                 // perform TLS handshake as client
SSL_read(ssl, buf, sizeof(buf));  // receive server's certificate + data
```

**⑤ Common errors/limits**
- **Certificate expired / hostname mismatch / untrusted CA** → handshake fails or browser warns; for self-signed certs you must add them to the trust store manually.
- On **non-blocking** sockets, `SSL_read`/`SSL_write` return `SSL_ERROR_WANT_READ` or `SSL_ERROR_WANT_WRITE` — these are not errors, you must retry later. Many beginners treat them as hard failures.
- **SSL/TLS 1.0 and 1.1 are deprecated** and disabled by modern browsers and libraries; only use TLS 1.2+.
- No certificate means **no authentication** — an attacker can MITM (man-in-the-middle) the connection. Always use certificates in production.
- RSA key-transport (old cipher suites) gives **no forward secrecy**; prefer ECDHE/DHE cipher suites.

**⑥ Conclusion** — TLS is the universal answer to "how do I make my sockets secure"; as a programmer you attach a library like OpenSSL to your socket and swap `read`/`write` for `SSL_read`/`SSL_write` — the library handles encryption, certificate verification, and key exchange for you.

---

### M15. gRPC  [🔴 8-mark]

**① Definition** — **gRPC (Google Remote Procedure Call)** is a high-performance, open-source **RPC framework** that lets a client program call a **method on a remote server as if it were a local function call**. It is built on **HTTP/2** and uses **Protocol Buffers (protobuf)** for fast binary serialisation.

**② Diagram**
```
 ┌──────────┐  HTTP/2 + protobuf  ┌──────────┐
 │ Client   │◀──────────────────▶ │ Server   │
 │ stub     │  (binary, fast,     │ stub     │
 │          │   multiplexed)      │          │
 └──────────┘                     └──────────┘
      │                                │
      ▼                                ▼
 .proto file                    generated code
 (interface contract)           (stubs in any language)

 The 4 call models:
 ┌──────────────────┬───────────────────────────────────────────────┐
 │ Unary            │  request ──▶ server ──▶ response              │
 │                  │  (1:1 — like a normal function call)          │
 ├──────────────────┼───────────────────────────────────────────────┤
 │ Server streaming │  request ──▶ server ──▶ response, response,  │
 │                  │                            response…          │
 │                  │  (1:N — server sends a stream)                │
 ├──────────────────┼───────────────────────────────────────────────┤
 │ Client streaming │  request, request… ──▶ server ──▶ response   │
 │                  │  (N:1 — client sends a stream)                │
 ├──────────────────┼───────────────────────────────────────────────┤
 │ Bidi streaming   │  request, request… ◀──▶ response, response… │
 │                  │  (N:N — both sides stream simultaneously)     │
 └──────────────────┴───────────────────────────────────────────────┘
```

**③ Full concept**
- You define the **service interface** in a `.proto` file (using Protocol Buffers language):
  ```proto
  syntax = "proto3";
  service Greeter {
    rpc SayHello (HelloRequest) returns (HelloReply);
    rpc StreamGreetings (HelloRequest) returns (stream HelloReply);
  }
  message HelloRequest { string name = 1; }
  message HelloReply   { string message = 1; }
  ```
- The **`protoc` compiler** with the gRPC plugin generates **client and server stub code** in many languages (C++, Java, Go, Python, C#, Node.js, Rust, etc.). The client calls a generated method; it serialises the request to protobuf, sends it over HTTP/2, and the server's generated stub deserialises it, calls your real method, and sends the protobuf response back.
- **HTTP/2** gives: **multiplexing** (many calls share one TCP connection without head-of-line blocking), **header compression** (HPACK — tiny headers), and **bidirectional streaming**.
- **Protobuf** gives: compact binary messages (much smaller and faster to parse than JSON/XML), strong typing (schema enforced at compile time), and backward/forward compatibility (optional fields, versioning by field numbers).
- gRPC also supports **deadlines/timeouts** (the client can specify "give up after 5 seconds"), **cancellation**, **interceptors** (middleware for logging/auth), and **load balancing**.

**④ Example — complete gRPC flow:**
```bash
# 1. Define the service
cat > greeter.proto <<EOF
syntax = "proto3";
package greeter;
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}
message HelloRequest { string name = 1; }
message HelloReply   { string message = 1; }
EOF

# 2. Generate code (C++ example)
protoc --cpp_out=. --grpc_out=. --plugin=protoc-gen-grpc=$(which grpc_cpp_plugin) greeter.proto
# Produces: greeter.pb.h, greeter.pb.cc (protobuf), greeter.grpc.pb.h, greeter.grpc.pb.cc (gRPC stubs)

# 3. Server implementation
class GreeterImpl final : public Greeter::Service {
  Status SayHello(ServerContext* context, const HelloRequest* req, HelloReply* reply) override {
    reply->set_message("Hello " + req->name());
    return Status::OK;
  }
};

# 4. Client call
Greeter::Stub stub(channel);  // channel = gRPC connection to "localhost:50051"
HelloRequest req; req.set_name("World");
HelloReply reply;
ClientContext ctx;
Status status = stub.SayHello(&ctx, req, &reply);
// reply.message() now contains "Hello World"
```

**⑤ Common errors/limits**
- **Binary payloads** — not human-readable like JSON; you need `grpcurl` (CLI), Wireshark with protobuf dissector, or gRPC reflection to debug.
- **Protobuf toolchain dependency** — `protoc` and language-specific plugins must be installed; version mismatch between client and server proto definitions can cause silent failures.
- **HTTP/2 required** — proxies and load balancers that don't support HTTP/2 will break gRPC. Need an L7 load balancer or a service mesh (Istio/Linkerd).
- Heavier than raw TCP for trivial use cases (handshake, HTTP/2 framing, protobuf schema overhead).

**⑥ Conclusion** — gRPC is the modern default for microservices and streaming workloads that need performance, strong typing and multi-language support — it replaces hand-written JSON/REST with a fast, contract-driven, streaming RPC system over HTTP/2 + protobuf.

---

### M16. WebSockets (short note, also see M12)

**① Definition** — WebSocket is a **full-duplex, persistent** messaging protocol over a single TCP connection, enabling low-latency, low-overhead two-way communication between client and server.

**② Handshake** — starts as an HTTP request: client sends `GET /chat HTTP/1.1` with `Upgrade: websocket` and `Sec-WebSocket-Key: <base64>`; server replies `101 Switching Protocols` with `Sec-WebSocket-Accept: <hash>`; after this the connection is upgraded to WebSocket and both sides exchange frames.

**③ Frame format** — each frame starts with: FIN (1 bit, is this the final fragment?) + opcode (4 bits: 0x1 text, 0x2 binary, 0x8 close, 0x9 ping, 0xA pong, 0x0 continuation) + MASK (1 bit, must be 1 for client→server) + payload length (7 bits; 126→2 extra bytes; 127→8 extra bytes) + masking key (4 bytes, if masked) + payload.

**④ Use cases** — chat applications, multiplayer games, live dashboards, stock tickers, IoT real-time push, collaborative editing.

**⑤ Limits** — idle connections may be killed by proxies after ~60 seconds (need ping/pong keepalive); masking adds overhead; server must handle many concurrent connections (select/poll/async).

**⑥ Conclusion** — WebSocket replaces HTTP's request/response with a persistent, low-overhead, full-duplex channel, making it the standard for real-time web applications.

---

### M17. Blocking vs Non-blocking I/O  [🔴★ Gandaki Q4a]

**① Definition** — Blocking and non-blocking are the two modes a socket can operate in, determining whether a system call **sleeps** (blocks the thread) or **returns immediately** when no data is available.

**② Diagram**
```
 BLOCKING MODE (default):
 ┌────────────┐      ┌─────────────────────────┐      ┌────────────┐
 │   app calls│      │ kernel: data not ready   │      │ data arrives│
 │   recvfrom │ ───▶ │ process sleeps (blocked) │ ───▶ │ data copied │
 │            │      │ ... wait ...             │      │ return data │
 └────────────┘      └─────────────────────────┘      └────────────┘
 (thread tied up the entire time)

 NON-BLOCKING MODE:
 ┌────────────┐      ┌──────────────┐      ┌────────────┐      ┌────────────┐
 │   app calls│      │ kernel: data │      │ app calls  │      │ data now   │
 │   recvfrom │ ───▶ │ not ready →  │ ───▶ │ recvfrom   │ ───▶ │ ready →    │
 │            │      │ EWOULDBLOCK  │      │ again...   │      │ data copied│
 └────────────┘      └──────────────┘      └────────────┘      └────────────┘
 (thread is free between calls, but wastes CPU polling)
```

**③ Full concept**
- **Blocking mode** (default): `recvfrom()` does not return until data arrives AND is copied into the app buffer. The process/thread sleeps (no CPU used, but tied up — cannot do anything else on this thread).
- **Non-blocking mode**: you set the socket to non-blocking (with `fcntl(F_SETFL, O_NONBLOCK)` on Unix, or `ioctlsocket(FIONBIO)` on Windows). Now `recvfrom()` returns **immediately** — if no data is ready, it returns -1 with `errno = EWOULDBLOCK` (or `WSAEWOULDBLOCK` on Windows). The application must **poll** (call again later).
- Non-blocking alone wastes CPU (busy-waiting). The practical solution: set non-blocking AND use `select()`/`poll()`/`epoll` to only call `recvfrom` when `select` tells you data is ready. This gives you non-blocking without busy-waiting.
- Blocking is simpler to code but ties up a thread per connection. Non-blocking + select is more complex but lets one thread handle many connections.

**④ Example/code**
```c
// Unix: make socket non-blocking
int flags = fcntl(sockfd, F_GETFL, 0);
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);

// Now recvfrom returns immediately:
ssize_t n = recvfrom(sockfd, buf, MAXLINE, 0, NULL, NULL);
if (n == -1) {
    if (errno == EWOULDBLOCK) { /* no data yet — try later */ }
    else { perror("recvfrom error"); }
} else { /* got n bytes of data */ }

// Windows: make socket non-blocking
unsigned long mode = 1;
ioctlsocket(s, FIONBIO, &mode);
// recv(s, buf, len, 0) now returns SOCKET_ERROR + WSAEWOULDBLOCK if no data
```

**⑤ Common errors/limits**
- Non-blocking without select/poll = busy-waiting (100% CPU, wasteful).
- `recv` returning 0 means **orderly close** (peer called `close`), not "no data" — this is different from -1/EWOULDBLOCK.
- Some functions behave differently in non-blocking mode (e.g., `connect()` returns immediately with `EINPROGRESS`; use `select` to know when the connection is complete).

**⑥ Conclusion** — Blocking is simple but ties up a thread; non-blocking lets one thread serve many sockets at the cost of polling — the practical pattern is non-blocking + select/poll/events for efficiency.

---

### M18. Signal-driven I/O vs I/O Multiplexing  [🔴★ Gandaki Q4a]

**① Definition** — Two I/O models that both solve the problem of "how do I know when data is ready on a socket?": **I/O multiplexing** uses `select()`/`poll()` to watch many descriptors; **signal-driven I/O** uses the **SIGIO** signal to get notified when a descriptor is ready.

**② Diagram**
```
 I/O MULTIPLEXING (select/poll):
 ┌───────────────────────────────────────────────────────┐
 │  app: select(sockfd+1, &readfds, NULL, NULL, NULL)    │
 │       │                                               │
 │       ▼  (process blocks in select)                   │
 │  kernel: wait for any fd in the set to become ready    │
 │       │                                               │
 │       ▼  (select returns when sockfd is readable)      │
 │  app: recvfrom(sockfd, ...)   ← blocks briefly here   │
 └───────────────────────────────────────────────────────┘

 SIGNAL-DRIVEN I/O:
 ┌───────────────────────────────────────────────────────┐
 │  app setup: sigaction(SIGIO, handler);                │
 │             fcntl(sockfd, F_SETOWN, getpid());        │
 │             fcntl(sockfd, F_SETFL, O_ASYNC);          │
 │       │                                               │
 │       ▼  (main loop runs free — doing other work)     │
 │  kernel: sockfd becomes readable                       │
 │       │                                               │
 │       ▼  (kernel sends SIGIO to the process)           │
 │  signal handler: recvfrom(sockfd, ...)   ← reads data │
 └───────────────────────────────────────────────────────┘
```

**③ Full concept**
- **I/O multiplexing**: the process calls `select()` (or `poll()`) and blocks, telling the kernel "watch these descriptors and wake me when any is ready." When `select` returns, the process knows which descriptors are readable and calls `recvfrom` on them (which now returns immediately). Requires **two system calls** per read (select + recvfrom). Supports watching **many** descriptors at once.
- **Signal-driven I/O**: the process enables the socket for SIGIO by calling `fcntl(sockfd, F_SETOWN, getpid())` (tell the kernel which process to signal) and `fcntl(sockfd, F_SETFL, O_ASYNC)` (enable asynchronous notification). Install a signal handler with `sigaction()`. When the socket becomes readable, the kernel sends **SIGIO**; the handler runs and calls `recvfrom`. The main loop is never blocked — it is free to do other work and only interrupted when I/O is possible.
- **Key difference**: multiplexing blocks the thread in `select()`; signal-driven never blocks the main thread — it runs free and is interrupted by the kernel. However, signal handling has its own complexities (signal delivery can be lost, handlers run asynchronously, reentrancy issues).
- **Both are synchronous**: the actual `recvfrom` still blocks briefly in both models. Only POSIX `aio_*` (model 5) is truly asynchronous.

**④ Example/code**
```c
// Signal-driven I/O setup:
void sigio_handler(int signo) {
    ssize_t n = recvfrom(sockfd, buf, MAXLINE, 0, NULL, NULL);
    printf("Received %zd bytes\n", n);
}

int main() {
    struct sigaction sa;
    sa.sa_handler = sigio_handler;
    sigemptyset(&sa.sa_mask);
    sigaction(SIGIO, &sa, NULL);

    fcntl(sockfd, F_SETOWN, getpid());         // which process gets SIGIO
    int flags = fcntl(sockfd, F_GETFL);
    fcntl(sockfd, F_SETFL, flags | O_ASYNC);   // enable async notification

    while (1) { /* main loop: do other work; SIGIO handler will interrupt */ }
}
```

**⑤ Common errors/limits**
- SIGIO can be **lost** if multiple signals arrive while the handler is running (they queue but the default action is to merge).
- Signal handlers must be **reentrant** (no mutex, no `printf` in some implementations, no malloc).
- Multiplexing has **FD_SETSIZE** limits; signal-driven doesn't, but requires careful signal management.
- On most systems, only one signal is delivered per fd readiness event — you may need non-blocking mode inside the handler to drain all data.

**⑥ Conclusion** — Both models tell you "when to do I/O"; multiplexing blocks on `select()` watching many fds, signal-driven never blocks the main loop but requires careful signal handling — choose based on your architecture's needs.

---

### M19. Daemonizing a Process (with code)  [🔴★ NCIT Q4a]

**① Definition** — A **daemon** is a long-running background process with **no controlling terminal**, started at boot, running until shutdown. Examples: `sshd`, `httpd`, `crond`, `syslogd`. **Daemonizing** means detaching a program from the terminal, working directory, umask, and standard file descriptors so it survives user logout.

**② Diagram — the full daemonization sequence**
```
 ┌──────────────┐     fork()      ┌──────────────┐
 │  parent      │ ──────────────▶ │  child       │
 │  (shell)     │  parent exits   │  (orphaned,   │
 │  exit(0)     │                 │  adopted by   │
 └──────────────┘                 │  init/PID 1)  │
                                  └──────┬───────┘
                                         │
                                    setsid()     ← new session, detached from controlling TTY
                                         │
                                  (optional 2nd fork) ← prevents re-acquiring a controlling TTY
                                         │
                              ┌──────────┴──────────┐
                              │  chdir("/")          │  ← don't hold a mount point busy
                              │  umask(0)            │  ← full control over file creation
                              │  close/reopen fd 0,1,2 → /dev/null  ← no stray output
                              │  (optional: write PID to /var/run/mydaemon.pid)
                              └─────────────────────┘
                                         │
                                         ▼
                              DAEMON RUNNING IN BACKGROUND
```

**③ Full concept — why each step is needed:**
1. **`fork()` + parent `exit()`**: the child becomes an **orphan**, adopted by init (PID 1). This ensures the daemon is not a session leader and the shell gets its prompt back.
2. **`setsid()`**: creates a **new session** and a new process group, detaching the daemon from any controlling terminal. Without this, logging out sends SIGHUP and kills the daemon.
3. **(Optional second `fork()`)**: a session leader (the first child after `setsid`) can still **re-acquire a controlling terminal** if it opens a terminal device without `O_NOCTTY`. The second fork makes the daemon a non-session-leader, preventing this. This is what `daemon(3)` in glibc does.
4. **`chdir("/")`**: the daemon inherits the shell's working directory; if that directory is on a removable filesystem, it stays mounted. Changing to `/` avoids this.
5. **`umask(0)`**: clears the file-mode creation mask so the daemon has full control over file permissions.
6. **Close/reopen fd 0,1,2 to `/dev/null`**: the daemon inherits stdin/stdout/stderr from the shell (which point to the terminal). Closing them and reopening to `/dev/null` means any `printf`/`fprintf(stderr,...)` goes nowhere instead of crashing or writing to the wrong place.

**④ Example/code — complete daemonization function:**
```c
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

void daemonize(void) {
    pid_t pid;

    // 1. First fork — parent exits
    if ((pid = fork()) < 0) err_sys("fork error");
    if (pid != 0) exit(0);          // parent (the shell) exits

    // 2. New session — detach from terminal
    setsid();

    // 3. Second fork — prevent re-acquiring a controlling TTY
    if ((pid = fork()) < 0) err_sys("fork error");
    if (pid != 0) exit(0);          // first child exits; grandchild is the daemon

    // 4. Set file permissions
    umask(0);

    // 5. Change working directory
    if (chdir("/") < 0) err_sys("chdir error");

    // 6. Close and redirect standard file descriptors
    int fd = open("/dev/null", O_RDWR);    // fd = 0
    if (fd >= 0) {
        dup2(fd, STDIN_FILENO);             // fd 0 → /dev/null
        dup2(fd, STDOUT_FILENO);            // fd 1 → /dev/null
        dup2(fd, STDERR_FILENO);            // fd 2 → /dev/null
        if (fd > STDERR_FILENO) close(fd);
    }
    // Now running as a proper daemon — no terminal, no CWD issue, no stray output
}
```

**⑤ Common errors/limits**
- **Forgetting `setsid()`**: the daemon still has a controlling terminal → killed on logout.
- **Not redirecting fd 0/1/2**: `printf` in the daemon writes to a dead terminal (or worse, to the wrong user's terminal).
- **systemd approach**: on modern Linux, daemons are often run as **foreground children of systemd**, which handles terminal/umask/cwd/signals for you. In that case, do NOT daemonize — just run in the foreground and let systemd manage it.
- Closing inherited file descriptors beyond 0/1/2 is also good practice (e.g., database connections, log files from a restart) — use `closefrom(3)` or iterate through `/proc/self/fd`.

**⑥ Conclusion** — The fork→setsid→fork→chdir→umask→redirect sequence produces a proper daemon; on modern Linux, systemd handles most of this for you — the key exam points are *why* each step is needed, especially setsid (terminal detachment) and the file descriptor redirect.

---

### M20. Socket Options (SO_LINGER, SO_KEEPALIVE, SO_REUSEADDR, SO_BROADCAST)  [🔴★ NCIT Q5b, Gandaki Q4b]

**① Definition** — Socket options let you customise how a socket behaves: port reuse, keepalive probes, close behaviour, and broadcast capability — controlled via `setsockopt()`/`getsockopt()`.

**② Diagram**
```
 ┌─────────────┐  setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on))  ┌────────┐
 │ application │ ───────────────────────────────────────────────────────────▶ │ kernel │
 │             │ ◀────────────────────────────────────────────────────────── │ socket │
 └─────────────┘  getsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &val, &len)       └────────┘
```

**③ Full concept — each option in detail:**
- **SO_REUSEADDR**: allows binding to a port in **TIME_WAIT** state. Without this, a crashed server that restarts gets "Address already in use." Set **before** `bind()`. Essential for all TCP servers.
- **SO_BROADCAST**: enables UDP sockets to send to broadcast addresses (e.g., 192.168.1.255). By default disabled; without it you get `EACCES` when sending to a broadcast address. Routers do not forward broadcast — limited to local subnet.
- **SO_KEEPALIVE**: TCP sends **keep-alive probes** after 2 hours idle. ACK → alive. RST → peer crashed (`ECONNRESET`). No response → 8 probes 75 s apart, then `ETIMEDOUT`. Detects dead peers on long-lived connections.
- **SO_LINGER**: controls `close()` behaviour:
  ```
  struct linger { int l_onoff; int l_linger; };

  l_onoff=0                  → close() returns immediately (default)
  l_onoff≠0, l_linger=0     → TCP ABORTS: discard buffer, send RST (no FIN)
  l_onoff≠0, l_linger≠0     → close() BLOCKS until data ACKed or timeout
  ```
- **SO_RCVBUF / SO_SNDBUF**: set kernel receive/send buffer sizes (must set **before** connect/listen).

**④ Example/code**
```c
int on = 1;
// SO_REUSEADDR — before bind
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));

// SO_BROADCAST — before sendto
setsockopt(udpfd, SOL_SOCKET, SO_BROADCAST, &on, sizeof(on));

// SO_KEEPALIVE — on connected socket
setsockopt(connfd, SOL_SOCKET, SO_KEEPALIVE, &on, sizeof(on));

// SO_LINGER — linger up to 5 seconds
struct linger li = {1, 5};
setsockopt(connfd, SOL_SOCKET, SO_LINGER, &li, sizeof(li));

// SO_LINGER — abort immediately (discard data, send RST)
struct linger li2 = {1, 0};
setsockopt(connfd, SOL_SOCKET, SO_LINGER, &li2, sizeof(li2));

// getsockopt to check current value
int val; socklen_t len = sizeof(val);
getsockopt(connfd, SOL_SOCKET, SO_KEEPALIVE, &val, &len);
printf("SO_KEEPALIVE = %d\n", val);
```

**⑤ Common errors/limits**
- `SO_REUSEADDR` semantics differ between Linux, BSD, and Windows; on Linux you may also need `SO_REUSEPORT` for multiple listeners on the same port.
- `SO_KEEPALIVE`'s 2-hour default is too long for most apps; adjust `TCP_KEEPIDLE` for shorter detection.
- `SO_LINGER` with a timeout **blocks** `close()` — dangerous if the peer is slow; can stall the server.
- Buffer options (`SO_RCVBUF`, `SO_SNDBUF`) must be set **before** connect/listen; setting after has no effect on existing connections.

**⑥ Conclusion** — These four options solve the most common server problems (restart without port conflict, detect dead peers, graceful vs abrupt close, broadcast); the `struct linger` table is a frequent exam question — memorise it.

---

### M21. WSAAsyncSelect vs WSAEventSelect  [🟡★ NCIT alt]

**① Definition** — Two Winsock async I/O models for non-blocking sockets: **WSAAsyncSelect** delivers socket-event notifications as **Windows messages** to a window; **WSAEventSelect** signals an **event object** instead.

**② Diagram**
```
 WSAAsyncSelect (message-based):
 ┌──────────┐  FD_READ event  ┌─────────────┐  WM_SOCKET message  ┌──────────┐
 │ socket   │ ──────────────▶ │ Winsock DLL  │ ──────────────────▶ │ WndProc  │
 │          │                 │ posts msg to │  (wParam=socket,   │ handles  │
 │          │                 │ window hWnd   │   lParam=event)    │ event    │
 └──────────┘                 └─────────────┘                     └──────────┘

 WSAEventSelect (event-based):
 ┌──────────┐  FD_READ event  ┌─────────────┐  signals hEvent     ┌──────────┐
 │ socket   │ ──────────────▶ │ Winsock DLL  │ ──────────────────▶ │ WSAWait  │
 │          │                 │ sets event   │                     │ ForMulti │
 │          │                 │ object       │                     │ pleEvents│
 └──────────┘                 └─────────────┘                     └──────────┘
```

**③ Full concept**
- **WSAAsyncSelect**: associates a socket with a **window handle (HWND)** and a **Windows message**. When any of the specified events occur (FD_READ, FD_WRITE, FD_OOB, FD_ACCEPT, FD_CONNECT, FD_CLOSE), Winsock **posts a Windows message** to the window. The window procedure (WndProc) decodes `wParam` (the socket) and `lParam` (the event) and handles it. Calling `WSAAsyncSelect` automatically switches the socket to **non-blocking** mode.
- **WSAEventSelect**: uses **event objects** instead of messages. Call `WSACreateEvent()` to create an event, `WSAEventSelect(s, hEvent, FD_READ|FD_CLOSE)` to associate the socket with the event, then `WSAWaitForMultipleEvents(nEvents, events, ...)` to wait. When signaled, call `WSAEnumNetworkEvents()` to find which socket/event fired. No window needed → good for console/background apps. Maximum **64 events per thread**.
- Both are **non-blocking** but still **synchronous** — the actual `recv` still happens in your code, not automatically by the kernel. They tell you *when to read*, not *that reading is done*.

**④ Example/code**
```c
// WSAEventSelect example:
WSAEVENT events[2];
SOCKET socks[2];
events[0] = WSACreateEvent();
WSAEventSelect(listenSock, events[0], FD_ACCEPT);
events[1] = WSACreateEvent();
WSAEventSelect(connSock, events[1], FD_READ | FD_CLOSE);

while (1) {
    DWORD idx = WSAWaitForMultipleEvents(2, events, FALSE, WSA_INFINITE, FALSE);
    idx -= WSA_WAIT_EVENT_0;
    WSANETWORKEVENTS netEvents;
    WSAResetEvent(events[idx]);
    WSAEnumNetworkEvents(socks[idx], events[idx], &netEvents);

    if (netEvents.lNetworkEvents & FD_ACCEPT) {
        SOCKET newConn = accept(listenSock, ...);
        // add newConn to the events array
    }
    if (netEvents.lNetworkEvents & FD_READ) {
        recv(socks[idx], buf, sizeof(buf), 0);
        // handle data
    }
}
```

**⑤ Common errors/limits**
- WSAAsyncSelect requires a **window and message pump** — not usable in console apps or services.
- WSAEventSelect limited to **64 events per thread** — must build your own mapping from event index to socket.
- Neither model does the I/O for you — you still must call `recv`/`send` yourself (only overlapped I/O/IOCP run I/O in the background).
- Both set the socket to non-blocking automatically — if you want blocking later, call `ioctlsocket(FIONBIO, 0)`.

**⑥ Conclusion** — WSAAsyncSelect is natural for GUI apps (message-driven), WSAEventSelect for console/background apps (no window needed); both are non-blocking notification models that tell you *when* to do I/O, not *that it's done*.

---

### M22. Securing a Network Application (hostname / IP / wrapper)  [🟡★ short note]

**① Definition** — Securing a network application means restricting who can connect and protecting the data, using **hostname-based access control**, **IP-based access control**, and **wrapper programs**, plus **TLS** for encryption.

**② Diagram**
```
 CLIENT ──▶ [WRAPPER PROGRAM / in.tcpd]
              │
              │ checks: is this hostname trusted? (DNS lookup)
              │ checks: is this IP in the allow list? (/etc/hosts.allow)
              │
              ├─ ALLOWED ──▶ start real service, relay data
              │
              └─ DENIED ──▶ log the attempt, drop connection, close
```

**③ Full concept**
- **By hostname/domain**: resolve the client's IP to a hostname (reverse DNS) and allow only trusted names. ⚠️ **DNS can be spoofed** — an attacker can forge DNS replies, so this alone is weak.
- **By IP number**: restrict by source IP address using `/etc/hosts.allow` + `/etc/hosts.deny` (TCP wrappers style), or firewall rules (`iptables`, `nftables`). Simple but IPs can be spoofed (though harder than DNS spoofing).
- **Wrapper program** (e.g., **TCP wrappers / `in.tcpd`**): a small front-end that intercepts the incoming connection, checks the client against a policy (hosts.allow/deny or custom rules), and only if allowed **launches the real service** and relays data. This implements access control **without modifying the server application** itself.
- **TLS/SSL** for data protection: encrypts all data in transit, authenticates the server via certificates, and detects tampering — essential in addition to access control.

**④ Example** — `/etc/hosts.allow`:
```
sshd: 192.168.1.0/24          # allow SSH from local subnet
in.telnetd: .trusted.com      # allow telnet from trusted.com domain
```
`/etc/hosts.deny`:
```
ALL: ALL                       # deny everything not explicitly allowed
```

**⑤ Common errors/limits**
- Trusting hostname alone (DNS spoofing); relying on source IP alone (IP spoofing).
- Forgetting that access control is **not encryption** — data is still in the clear. Pair with TLS.
- TCP wrappers are deprecated in modern Linux (systemd doesn't use them); use firewall rules instead, but the concept is the same.

**⑥ Conclusion** — A layered defence (hostname allowlist + IP firewall + wrapper program + TLS) protects services from unwanted connections and in-transit tampering; the wrapper concept is key to understanding how to gate access without modifying the server.

---

## Quick topic → past-paper match (priority recap)
- **TCP state-transition diagram** — NCIT & Gandaki ★ HIGH
- **TCP/UDP/SCTP compare** — NCIT ★ HIGH
- **Value-result arguments / sockaddr structures / byte ordering** — NCIT & Gandaki ★ HIGH
- **bind() in TCP client** — NCIT ★ HIGH
- **I/O models / blocking vs non-blocking / signal-driven vs multiplexing** — both ★ HIGH
- **Multiple-client handling (fork / select / threads)** — both ★ HIGH
- **Socket options + setsockopt/getsockopt** — both ★ HIGH
- **Winsock vs UNIX, WSAStartup/WSACleanup, DLLs, UDP/TCP client-server** — both ★ HIGH
- **Overlapped I/O, WSAAsyncSelect/WSAEventSelect, WSAPoll** — both ★ HIGH
- **HTTP vs WebSocket + server, SDN, gRPC, TLS/SSL, iperf/netstat** — both ★ HIGH (mostly short notes)

## Exam tips
- Always **draw the diagram** (TCP state transition, three-way handshake, I/O models, syslog block) — examiners award marks for diagrams.
- Write the **call sequences** for TCP/UDP in *both* Unix and Winsock.
- For 8-mark questions: **Definition → diagram → explanation → example/limits → conclusion** to maximise full marks.
