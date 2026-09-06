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

**Plain meaning:** think of the transport layer as a "delivery company" that moves your application's data from one computer to another. TCP, UDP and SCTP are three different delivery services with different guarantees.

- **TCP = registered, safe courier.** Every packet must arrive, in the right order. If one is lost, it is re-sent. Slow but bulletproof.
- **UDP = a letter thrown in a mailbox.** Fastest possible, but no guarantee it arrives, and no order. Great for things that must not wait.
- **SCTP = safe courier with two postmen and multiple letter-boxes.** Like TCP's reliability, but for phone-network signalling — built so a call never drops even if one network link dies.

**Where they all live:** all three sit **between your application and the network (IP)**. They take your data, split it into pieces, add a **port number** (so it reaches the right app on the right machine), and hand it to IP for delivery. They are all "Layer 4" (transport layer according to the TCP/IP model).

**Point-by-point comparison (memorise this table — it is most of the marks):**

| Feature | TCP | UDP | SCTP |
|---|---|---|---|
| Connection | Connection-**oriented** (shake hands first) | **Connectionless** (send whenever) | Connection-oriented (4-way handshake) |
| Reliability | Reliable via ACK + retransmission | Unreliable (best effort) | Reliable (ACK + retransmit) |
| Ordering | Byte stream, **sequenced** | No ordering | **Message-ordered** |
| Message boundaries | **No** (pure byte stream) | **Yes** (datagrams) | **Yes** (messages) |
| Flow/congestion control | Yes | No | Yes |
| Multi-homing | No (one IP at a time) | No | **Yes** (multiple IPs per endpoint) |
| Data transfer | Stream | Datagram | **Message + stream** |
| Handshake | 3-way (SYN/SYN+ACK/ACK) | none | **4-way** (INIT/INIT-ACK/COOKIE) |
| Protection against SYN flood | Partial | n/a | **4-way handshake w/ cookie** |
| Usage | HTTP, FTP, SMTP, SSH | DNS, NFS, SNMP, VoIP, streaming | Telephony/Signaling (SIGTRAN), IP telephony |

**Why these differences matter (the "which one when" logic — write one line each):**
- **Reliability costs extra work** (headers, ACKs, retransmissions). Use TCP when the data *must* arrive intact — files, web pages, email.
- **No reliability gives low latency and tiny headers.** Use UDP when the data must arrive *fast* and a small loss is fine — DNS (one quick query), live video/audio, fast-action games (a late packet is worse than a missing one).
- **SCTP is the best of both worlds:** reliable AND keeps message boundaries AND survives a network-interface failure (multi-homing). That is why telephone signalling uses it — a call must not drop because one link dies.

**How to remember SCTP's multi-homing:** it is designed for **telephone** networks, and a phone system cannot say "sorry, the line is busy" — so SCTP lets one call use **two IP addresses at once**; if one path breaks, the other carries the call seamlessly.

**What an examiner wants:** the table (bulk of the marks), one line on *why each exists*, and one real example per protocol.

---
**Marking scheme (8 marks):** table = **4**, context sentence = **1**, why-differences-matter = **2**, examples = **1**.

---

### Q2 🔴★ Explain the TCP three-way handshake. (asked in multiple papers)

**Plain meaning:** before two computers can exchange data, they must prove "I'm here and ready". TCP does that with exactly **three packets** — a kind of introduction ceremony between the client and the server, like two walkie-talkie users confirming each other before a conversation.

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

**Step by step (the ceremony):**
1. **Client → SYN (seq = x):** the client raises a "SYN" flag and says "hi, my numbering starts at x". This is the *active open* — the client is the one reaching out.
2. **Server → SYN+ACK (seq = y, ack = x+1):** the server answers both at once: "I got your x" (ack = x+1) "and my numbering starts at y". This is the *passive open* — the server accepts.
3. **Client → ACK (ack = y+1):** the client says "I got your y". Now **both sides know the other is alive and their numbers agree** — data can flow.

**Why "three-way" and not "two-way":** step 2 cleverly packs the server's "yes" (SYN) and its confirmation of the client (ACK) into **one packet**. That is why it is exactly three packets total. If only two packets were used, the server could never be sure the client actually received its reply — the classic "two-army problem". The third ACK removes the doubt.

**Why sequence numbers are sent at all (deep understanding):** each side must know the other's *numbering start point* so that when data floods in later, it can (a) put bytes in the correct order and (b) spot and reject **duplicates**. That number is the ISN (initial sequence number) — normally random and unpredictable (see Q3).

**What the handshake achieves (three exam marks):**
- Proves both sides are **reachable and ready**.
- Exchanges **initial sequence numbers (ISN)**, so later data can be ordered and duplicates detected.
- Negotiates options (MSS — biggest chunk of data allowed, window scaling, timestamps) during these first packets.

**Try it yourself:** run `tcpdump -nn` or `tshark` on any machine, then open a website. You will literally see these three packets — the SYN, the SYN+ACK, and the ACK — before any HTTP data.

---
**Marking scheme (7–8 marks):** correct diagram with all seq/ack labels = **3**, each step explained = **3**, "why three-way not two-way" + purpose = **2**.

---

### Q3 🔴★ Why should the initial sequence number NOT start from 0? Explain TCP state transition diagram. (asked: NCIT 2025 Q1a — 7 marks)

**Part A — Why ISN (initial sequence number) should not start from 0:**

**Plain meaning:** TCP labels every byte with a **sequence number** (a 32-bit counter, so it is huge). If every connection started at the same number (0), two different connections could accidentally use the same labels — and an old, lost packet from an earlier connection could be mistaken for *fresh data* in the new one, silently corrupting it. It is like re-using the same page numbers in a new notebook: an old torn-out page found on the floor could be filed in the wrong notebook.

The three reasons (write all three):
1. **Stale-packet collision:** an **old, delayed segment** still floating in the network could carry a sequence number that **matches** the new connection's numbers. The receiver would accept this ancient garbage as valid new data — corruption with no error message.
2. **Security:** if ISN always = 0, an attacker can **guess/predict** the numbers and inject fake data into the connection (sequence-number prediction attack).
3. **Solution in practice:** each side picks a **random, unpredictable ISN** AND waits in **TIME_WAIT** (about 2× the Maximum Segment Lifetime) so old duplicates die off before the numbers can be reused. Classic rule: the ISN advances with a timer (~every 4 µs), making it unpredictable.

Takeaway line (memorise): *"A non-zero, unpredictable ISN + TIME_WAIT prevents a stale segment from being mistaken for new data, and defeats sequence-number guessing."*

**Part B — TCP state-transition diagram (draw this — the single most-asked diagram):**

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

**Three easy "journeys" to memorise:**
- **Server (setup):** `CLOSED → LISTEN → SYN_RCVD → ESTABLISHED`
- **Client (setup):** `CLOSED → SYN_SENT → ESTABLISHED`
- **Active closer (who sends FIN first):** `ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED`
- **Passive closer (who receives FIN first):** `ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED`

**How to read the diagram for marks:** every arrow is labelled with either a **cause** (an event: `send FIN`, `recv SYN+ACK`) or an **effect** (a state change). Say for any state: *which side* is in it ('active' = the one that dialed, 'passive' = the one that listens) and *what event moves it out*.

**Where the marks come from:** drawing the state diagram is worth the most (proof of understanding). Then add one sentence per key state — *which side* is in it and *what event* moves it out.

---
**Marking scheme (7 marks):** ISN part = **3** (problem, TIME_WAIT, security), state diagram = **3**, one sentence per key state = **1**.

---

### Q4 🟡★ What is a socket? What are the types of sockets? (asked: NCIT 2025 Q2 — differentiate TCP vs UDP socket)

**Plain meaning:** a socket is the **doorway your program uses to talk over the network** — a combination of an **IP address + port number** that uniquely identifies "this program on this machine". When an application wants to send or receive data, it opens a socket and uses it like a file handle.

**Types of sockets (three main ones + one extra):**
1. **Stream socket (`SOCK_STREAM`)** → uses **TCP** — enters a connection first, reliable, ordered **byte stream** (a continuous flow, no "message" boundaries).
2. **Datagram socket (`SOCK_DGRAM`)** → uses **UDP** — no connection, unreliable, but keeps each **message (datagram)** separate and whole.
3. **Raw socket (`SOCK_RAW`)** → bypasses the protocols and gives **direct access to raw IP packets** (you build your own headers) — for low-level tools like `ping`, packet sniffers, routing protocols.
4. (Unix also has **sequenced-packet** sockets `SOCK_SEQPACKET` — reliable, ordered *messages*; used with SCTP/Unix-domain sockets.)

**How to choose:** need guaranteed delivery → `SOCK_STREAM`; need speed and can survive loss → `SOCK_DGRAM`; need to craft your own network packets → `SOCK_RAW`.

**TCP socket vs UDP socket (asked directly — write this mini-table):**

| | TCP socket | UDP socket |
|---|---|---|
| Type constant | `SOCK_STREAM` | `SOCK_DGRAM` |
| Connection | must `connect` first | connectionless |
| Reliable | yes (ack/retransmit) | no |
| Ordering | byte stream (no boundaries) | keeps datagram boundaries |
| Server calls | `bind → listen → accept` | `bind → recvfrom` only (no listen/accept) |
| Client send | `send`/`write` | `sendto` |
| Data recv | `recv`/`read` (may come in partial pieces) | `recvfrom` (gets one whole datagram) |

**Why the server call sequences differ (understand, not memorise):** TCP must answer a phone that keeps ringing — hence `listen` (start watching the line) and `accept` (pick up each caller). UDP doesn't ring a phone at all — datagrams just arrive, so the server only needs `bind` (claim the address) and then constant `recvfrom`.

**How a socket is created (same for both):** `socket(family, type, protocol)` e.g. `socket(AF_INET, SOCK_STREAM, 0)` — family says *which network* (IPv4), type says *which service* (stream vs datagram), protocol 0 means "pick the default for that type".

---
**Marking scheme (6–8 marks):** definition of socket = **2**, three socket types = **3**, TCP-vs-UDP table = **3**.

---

### Q5 🟢 What is IPC? List the evolution of UNIX IPC mechanisms.

**Plain meaning:** **IPC (InterProcess Communication)** = the ways two (or more) **processes exchange data and coordinate** with each other. Each process has its own private memory, so they need a *common channel* — a file, a kernel object, or shared memory — to talk.

**Evolution (timeline):**
```
Pipes ─▶ Named pipes (FIFOs) ─▶ System V message queues ─▶ POSIX msg queues ─▶ RPC
                                                                         ─▶ Sockets/Networks
```

**Each in one line (how they improved over time):**
1. **Pipes** — a one-way byte stream, **parent↔child only**, no name.
2. **FIFOs (named pipes)** — same idea but with a **pathname**, so *unrelated* processes can open them.
3. **System V message queues** — kernel-managed, **message-addressed** (send whole messages by key).
4. **POSIX message queues** — the modern replacement, namespaced (`/mq`).
5. **RPC / sockets** — let processes on *different machines* communicate (the basis of this whole course).
6. **Shared memory** — the fastest (no kernel copy) but must be synchronised with semaphores.

**What problem each step solved (the "why" line):**
- Pipes couldn't let *unrelated* processes talk → FIFOs added a name.
- Flows of bytes were awkward for structured data → message queues added message boundaries.
- Everything so far was **local** → RPC and sockets extended the same idea to the *network*.
- Kernel copying was slow → shared memory removed the kernel from the hot path.

**Why this matters for a network course:** most IPC is *local* (same machine); **sockets carry IPC to remote machines** — which is exactly what network programming is about.

---
**Marking scheme (4–5 marks):** definition = **1**, correct chronological list = **2**, one line per mechanism = **2**.

---

### Q6 🟢 What are the three ways two UNIX processes can share info?

**Plain meaning:** two running programs can't "see" each other's memory, so UNIX gives exactly **three** channels they can share information through — like three different ways two people in separate rooms can pass notes: through a shared notebook, through a reception desk, or by pointing both at the same whiteboard.

1. **Through a shared file** in the filesystem — both processes read/write the same file (the kernel handles it). Simple, but needs **locking** to avoid conflicts and is **slow** (disk I/O).
2. **Through the kernel** — pipes, FIFOs, message queues, semaphores. Every operation is a **system call**; the kernel stores and hands over the data. Needs synchronisation (especially semaphores).
3. **Through shared memory** — both processes map the **same physical memory** into their own address space. **Fastest** (no kernel copying every time) but **most error-prone** — you must synchronise manually (semaphores/mutexes) and there's no kernel safety net.

**Why three and not one:** it's a **speed-vs-simplicity trade-off**. Files = easiest but slowest; shared memory = fastest but you must be careful; kernel objects = the balanced middle ground.

**Quick memory hook:** *"File = the slow, safe option; kernel = the medium, structured option; shared memory = the fast, dangerous option."*

---
**Marking scheme (4–5 marks):** each way = **1**, plus **1–2** for the trade-off/why-all-three explanation.

---

### Q7 🟡★ Define the port number ranges & socket pair.

**Plain meaning:** a **port number** is a 16-bit number (0–65535) that tags "which application on this machine". Many apps can share one IP because each has a different port — like one apartment building (IP) with many flats (ports).

**The three ranges (memorise):**

| Range | Name | Notes / examples |
|---|---|---|
| **0–1023** | **Well-known** | Standard services; e.g. 80 HTTP, 443 HTTPS, 21 FTP, 25 SMTP, 53 DNS |
| **1024–49151** | **Registered** | Applications register these with IANA; e.g. 1433 SQL Server, 3306 MySQL |
| **49152–65535** | **Dynamic / private (ephemeral)** | Auto-assigned by the OS to clients for the **local** end of a connection |

**Why the ranges exist (understand):** the well-known range is reserved so you can always find a standard service on its famous port; the registered range is for apps that paid to have a stable, non-conflicting port; the dynamic range is the OS's free pool to hand out automatically to outgoing client connections.

**Socket pair (TCP):** one TCP connection is uniquely identified by **four values** — the 4-tuple:
```
(local IP, local port, foreign IP, foreign port)
```
- **local** = your side, **foreign** = the peer's side.
- This lets two hosts run *many* simultaneous connections — each differs in at least one of the four values.
- Compare: one *socket* = (local IP, local port); the *socket pair* = the full 4-tuple.

**Concrete example:** your browser and your mail program both connect to `google.com` from `192.168.1.5`. The two connections are different because the OS gave each a different ephemeral port (e.g. 50000 for the browser, 50001 for mail) — `(192.168.1.5, 50000, 172.217.10.14, 443)` vs `(192.168.1.5, 50001, 172.217.10.14, 443)`.

---
**Marking scheme (5 marks):** three ranges = **3**, well-known examples = **1**, socket-pair 4-tuple + why unique = **1–2**.

---

### Q8 🟡★ How is a TCP connection terminated? Why is TIME_WAIT needed?

**Plain meaning:** closing a TCP connection is a **polite two-way farewell** — each side says "I'm done sending" separately, and both must acknowledge. This normally takes **four packets** (a FIN/ACK exchange for each direction):

```
A ── FIN ──▶ B      (A says: I have no more data to send)
A ◀── ACK ── B      (B acknowledges A's FIN)
A ◀── FIN ── B      (B also finishes its side)
A ── ACK ──▶ B      (A acknowledges B's FIN)  → connection fully closed
```

**Who starts it:** the side that calls `close()` first sends the FIN. Because the two directions close independently, a connection can be **half-closed** — one side done sending while still receiving (this is exactly what `shutdown(SD_SEND)` gives you).

**Why TIME_WAIT is needed (it lasts 2×MSL — twice the Maximum Segment Lifetime, roughly 2 minutes):**
1. **Reliable close:** if the **final ACK is lost**, the peer will re-send its FIN; TIME_WAIT lets the closing side re-send the ACK. Without it, the peer would be stuck in LAST_ACK forever.
2. **Let old duplicates die:** a delayed packet from *this* connection could still be floating in the network. TIME_WAIT holds the 4-tuple long enough (2×MSL) that any duplicate either arrives (and is ignored) or expires — so it **cannot contaminate a new connection** that reuses the same local IP/port.

**Trade-off (common exam point):** TIME_WAIT is why a server restarting sometimes gets **"address already in use"** — fixed with `SO_REUSEADDR` (see Q26/Q27).

**Memory hook:** TIME_WAIT's two jobs are *"answer the last ACK if it's re-asked"* and *"hold the address till old ghosts are gone"*.

---
**Marking scheme (5–6 marks):** 4-segment diagram = **2**, half-close idea = **1**, the two TIME_WAIT reasons = **2–3**.

---

## Unit 2 — Unix Basics

### Q9 🔴★ Explain value-result arguments in socket programming. Why are they needed? (asked: NCIT 2025 Q3a, Gandaki 2025 Q2b)

**Plain meaning:** a **value-result argument** is one variable that does *two* jobs: your program puts a **value in** (to tell the kernel something), the kernel does its work, then **writes a new value back** into the same variable. Like filling a form with the number of seats you have, handing it to the booking office, and getting it back stamped with the *actual* number of seats they allocated.

**Two directions of length passing in socket calls (the key idea):**
- **Process → kernel** (`bind`, `connect`, `sendto`): your program *supplies* the address, so it just passes the size **by value** — the kernel only *reads* it to know how many bytes to copy.
- **Kernel → process** (`accept`, `recvfrom`, `getsockname`, `getpeername`): the *kernel produces* the address, so you pass a **pointer to the size** (`socklen_t *`). On input the *value* tells the kernel how big your buffer is (so it never overflows); on output the kernel *updates* it to the actual size it stored. **This two-way use is the value-result pattern.**

```c
struct sockaddr_in cli;
socklen_t len = sizeof(cli);          /* value: "my buffer is this big" */
int cfd = accept(listenfd, (SA*)&cli, &len);  /* kernel returns real size in len → result */
printf("peer stored %d bytes in cli\n", len); /* now len = the actual length used */
```

**Trace the code step by step (exam-grade understanding):**
1. `len = sizeof(cli)` — this is the **value** part: you advertise "my buffer can hold one IPv4 address".
2. `accept(...)` — the kernel fills `cli` with the client's real address and sets `len` to how many bytes it actually used.
3. After the call, `len` is the **result** — the real size. You can use it to tell which family connected (16 bytes = IPv4, 28 bytes = IPv6).

**Why value-result is needed:**
1. **Safety (no overflow):** the kernel must know your buffer's size, or it could write past the end of your memory.
2. **Variable sizes:** address families have different lengths (IPv4 = 16 B, IPv6 = 28 B). You give the input size, the kernel reports the real output size — so after `accept` you can tell *which family* actually connected.

**Common mistake (worth a mark):** passing `&len` **uninitialised** — the kernel reads garbage → unpredictable behaviour (overflow or wrong data). Always set `len = sizeof(buffer)` first.

**Memory hook:** *"When you give the address → pass size by VALUE. When the kernel gives you an address → pass a pointer to size."*

---
**Marking scheme (8 marks):** definition = **2**, two directions table = **2**, code with value+result explained = **2**, why-needed = **2**.

---

### Q10 🔴★ Ways to pass the length of a socket structure — with function prototypes. (asked: NCIT 2025 Q2a)

**Plain meaning:** there are exactly **two** styles for passing the length of a socket address, and they match the two groups from Q9: **by value** (when you give the kernel an address) and **by reference / value-result** (when the kernel gives you an address back).

**Group 1 — length passed BY VALUE (you supply the address, one-way):**
```c
int bind(int socket, const struct sockaddr *address, socklen_t address_len);
int connect(int socket, const struct sockaddr *address, socklen_t address_len);
ssize_t sendto(int socket, const void *buffer, size_t length, int flags,
               const struct sockaddr *dest_addr, socklen_t dest_len);
```
Here `address_len` is a plain value the kernel reads (it tells the kernel how many bytes of `address` to copy). No write-back.

**Group 2 — length passed BY REFERENCE / value-result (kernel produces the address, two-way):**
```c
int accept(int socket, struct sockaddr *address, socklen_t *address_len);
ssize_t recvfrom(int socket, void *buffer, size_t length, int flags,
                 struct sockaddr *address, socklen_t *address_len);
int getsockname(int socket, struct sockaddr *address, socklen_t *address_len);
int getpeername(int socket, struct sockaddr *address, socklen_t *address_len);
```
In group 2, `address_len` is a **pointer (`&len`)**. You set `*address_len` to the buffer size *before* the call; the kernel writes back the **actual** size after. That is value-result.

**How to remember which group (the one-line rule):**
- If the **kernel produces** the address (accept/recvfrom/getsockname/getpeername) → length is **value-result (pointer)**.
- If **you supply** the address (bind/connect/sendto) → length is **by value**.

**Quick check on the pointer functions:** `recvfrom` and `accept` write *into* your `sockaddr` — so they MUST know how big your buffer is, hence a pointer arg. `bind` and `connect` read *from* your struct, so a plain number is enough.

**Common exam trap:** forgetting to initialise `*address_len` before `accept`/`recvfrom` → garbage length → overflow or wrong data. Always do `socklen_t len = sizeof(addr);`.

---
**Marking scheme (8 marks):** stating the two ways = **2**, correct prototypes for each group = **4** (2 each), how-to-remember rule = **1**, common trap = **1**.

---

### Q11 🔴★ Explain the socket address structures (sockaddr, sockaddr_in, sockaddr_in6, sockaddr_storage). (asked: NCIT 2025 Q2b, Gandaki 2025 Q2a/Q2b)

**Plain meaning:** a socket address structure is a small C structure that holds "who to talk to" — the **family** (IPv4 / IPv6 / Unix), the **port**, and the **address**. You fill it, then hand it to every socket call. Think of it as the *address card* you give the post office.

| Structure | Family | Size | Key fields | Purpose |
|---|---|---|---|---|
| `struct sockaddr` | generic | 16 B | `sa_family`, `sa_data[14]` | generic/old casting form used by all socket functions |
| `struct sockaddr_in` | AF_INET | 16 B | `sin_family`, `sin_port`, `sin_addr.s_addr`, `sin_zero[8]` | IPv4 addresses |
| `struct sockaddr_in6` | AF_INET6 | 28 B | `sin6_family`, `sin6_port`, `sin6_flowinfo`, `sin6_addr`, `sin6_scope_id` | IPv6 addresses |
| `struct sockaddr_storage` | generic | ≥128 B | opaque; aligned for any family | can hold **any** family safely |

**Field by field (`sockaddr_in`, the IPv4 one):**
- `sin_family` — always `AF_INET` (2 bytes).
- `sin_port` — 16-bit port in **network byte order** → set with `htons()`.
- `sin_addr.s_addr` — 32-bit IPv4 address in **network byte order** → set with `htonl()` or `inet_pton()`.
- `sin_zero[8]` — padding to make the size match the generic `sockaddr`; you must **zero** it (usually `bzero()` the whole struct). Ignore it — it's just filler so the sizes line up.

**Why every call takes `struct sockaddr *` (the trick that confuses beginners):**
All socket functions accept a **generic pointer** `(const struct sockaddr *)`. You *declare* a concrete struct (`sockaddr_in`) but *cast* it to `(struct sockaddr *)` when calling `bind`/`connect`/`accept`. This is C's trick for "polymorphism": one generic pointer that actually points to a family-specific struct.
```c
struct sockaddr_in serv;
bind(sockfd, (struct sockaddr *)&serv, sizeof(serv));
```
The function doesn't need to know which family struct it is — because `sa_family` *inside* the struct tells it (AF_INET, AF_INET6, AF_UNIX, ...).

**Why `sockaddr_storage` matters (asked):**
- It is **big enough (≥128 B)** to hold the **largest** socket-address type the system supports (IPv4 *or* IPv6).
- It has the **strictest alignment**, so you can safely declare it, pass it to `accept`/`recvfrom`, and afterwards inspect `ss_family` to learn which family actually arrived. This is how you write **family-neutral** servers (they work with IPv4 and IPv6 without knowing in advance).

**Naming note (common mix-up):** generic = `sa_*`, IPv4 = `sin_*`, IPv6 = `sin6_*`, storage = `ss_family`. Don't confuse them. The prefix tells you which struct you're touching.

---
**Marking scheme (8 marks):** table of 4 structs = **4**, one-field explanation (sockaddr_in) = **1**, generic-pointer casting = **1**, sockaddr_storage significance = **2**.

---

### Q12 🔴★ Byte ordering & manipulation functions. (asked: NCIT 2025 Q2a)

**Plain meaning:** computers can store numbers "right-way-round" or "back-to-front". If two machines disagree on which, a port number sent by one will be read wrongly by the other. TCP/IP solved this by declaring **one worldwide standard** — *network byte order* — and giving you little conversion helpers so you never have to think about it.

**The problem (two memory layouts):**
- **Big-endian:** most-significant byte first (e.g. `0x1234` → `12 34`) — like writing a number the way English reads digits.
- **Little-endian** (Intel): least-significant byte first (`0x1234` → `34 12`) — like writing it backwards.
- TCP/IP protocols mandate **network byte order = big-endian**. Without converting, a little-endian machine and a big-endian machine would read the same port/address differently.

**Why the conversion is needed even on an Intel PC:** your Intel machine stores 0x1234 as `34 12` in memory. When you send it on the wire you must put `12 34` on the wire (network order). `htons(0x1234)` does exactly that swap for you — so your code is correct on *every* machine, Intel or not. On a big-endian machine the same function does nothing. **Always use them — never assume.**

**The solution — four conversion functions (memorise):**
```c
uint16_t htons(uint16_t hostshort);  /* Host → Network, Short (16-bit, ports)  */
uint32_t htonl(uint32_t hostlong);   /* Host → Network, Long  (32-bit, addresses) */
uint16_t ntohs(uint16_t netshort);   /* Network → Host, Short */
uint32_t ntohl(uint32_t netlong);    /* Network → Host, Long  */
```
- `htons`/`htonl` **before sending**: build `sin_port` with `htons(...)`, `sin_addr.s_addr` with `htonl(...)` or `inet_pton`.
- `ntohs`/`ntohl` **after receiving**: read `sin_port`, `sin_addr` values back into host order for printing/logging.

**Memory hook for which is which:** the letter order spells a route: `h`→`to`→`n` (host becoming network = *before* sending) and `n`→`to`→`h` (network becoming host = *after* receiving). "S" for **s**hort = ports (2 bytes, 16-bit). "L" for **l**ong = IP addresses (4 bytes, 32-bit).

**Manipulation functions** — for raw binary data (not C strings):
- BSD (old): `bzero(ptr, n)`, `bcopy(src,dst,n)`, `bcmp(...)`.
- ANSI/modern (preferred): `memset(ptr, 0, n)`, `memcpy(dst,src,n)`, `memcmp(a,b,n)`.
- Use them to **zero/initialise** address structures (e.g. `bzero(&serv, sizeof(serv))` or `memset(&serv, 0, sizeof(serv))`) before filling fields — this clears the `sin_zero` padding and prevents uninitialised bytes from leaking into your data.

---
**Marking scheme (8 marks):** endianness concept = **2**, four functions = **2**, when to apply each = **2**, manipulation functions = **2**.

---

### Q13 🟡★ inet_aton / inet_addr / inet_ntoa / inet_pton / inet_ntop.

**Plain meaning:** your program deals with IP addresses in two forms: the **human form** everyone types (`"192.168.1.1"`) and the **machine form** stored in the struct (4 bytes of binary). These five functions convert between the two. Think of them as translators between "people speak" and "computer speak".

- **`inet_aton("1.2.3.4", &addr)`** — human → machine (IPv4). Returns non-zero on success, 0 on failure. **Preferred** for IPv4 (gives a proper error signal). `addr` is a `struct in_addr`.
- **`inet_addr("1.2.3.4")`** — the same idea but returns the address **by value** (`in_addr_t`); returns `INADDR_NONE` on error. **Problem:** the valid address `255.255.255.255` also equals `INADDR_NONE`, so you cannot tell "error" from "a real address" — avoid it.
- **`inet_ntoa(addr)`** — machine → human (returns a pointer to a **static** buffer). **Not thread-safe** (the next call overwrites the buffer). Deprecated by POSIX.
- **`inet_pton(family, src, dst)`** — **human → machine for BOTH IPv4 AND IPv6** (`AF_INET` or `AF_INET6`). Returns 1 on success, 0 if the address is invalid, -1 if the family is wrong.
- **`inet_ntop(family, src, dst, size)`** — **machine → human for BOTH IPv4 AND IPv6**, into a buffer you provide. Thread-safe and preferred.

**Why these functions exist (the "why" for the exam):** a human address like `"192.168.1.1"` is *four numbers and three dots* — but the kernel stores it as *four raw bytes*. You cannot just `strcpy` a string into `sin_addr`; you must convert. These helpers do the parse and the number conversion for you.

**Recommendation (modern):** use `inet_pton`/`inet_ntop` — they handle both families, are thread-safe, and give clean error codes. `inet_aton` is fine for IPv4-only code; avoid `inet_addr` and prefer not to use `inet_ntoa`.

---
**Marking scheme (6 marks):** purpose (two-way conversion) = **2**, each function's job = **2**, modern recommendation/limits = **2**.

---

### Q14 🔴★ Write the TCP server & client system-call sequence. (asked multiple times)

**Plain meaning:** this is the exact *order of function calls* a TCP program makes. There is one set for the **server** (which waits for calls) and one for the **client** (which dials out). Memorise both sequences, then each call's one-line job.

**Server (passive open):**
```
socket() → bind() → listen() → accept() → read()/write() → close()
```
**Client (active open):**
```
socket() → connect() → read()/write() → close()     (bind optional)
```

**What each call does (one line each):**
- `socket(family,type,proto)` — create the endpoint (the doorway).
- `bind(sock, &addr, len)` — **server**: attach a specific IP/port to the socket (a client *may* skip this).
- `listen(sock, backlog)` — **server only**: mark the socket as ready to accept; incoming connections wait in a queue of size `backlog`.
- `accept(sock, &cli, &len)` — **server only**: block until a client connects, then return a **new** connected socket just for that client.
- `connect(sock, &addr, len)` — **client only**: dial the server and start a connection.
- `read()`/`write()` or `send()`/`recv()` — exchange data.
- `close(sock)` — release the socket; also `shutdown()` for a graceful half-close.

**Why listen + accept are separate (understand, don't memorise):** `listen` tells the kernel "start accepting connections into the backlog queue" — from this moment a client's connection is actually completed even if you haven't called `accept` yet. `accept` then simply *picks up* the next completed connection from that queue. That split is why the server handles one client while others wait politely in the queue.

```
SERVER                              CLIENT
socket()  ───────── course of time ─► socket()
bind()  ────────── (client may also bind a specific source port) ─►
listen()
accept() ◀── 3-way handshake ─────── connect()
         ◀── data: read()/write()──────▶
close()                                 close()
```

**Gotchas (worth marks):** the server blocks in `accept` until a client arrives; `accept` returns a *new* fd (the original listener is kept for more connections); a real server loops `accept → fork/thread` to serve many clients at once.

---
**Marking scheme (8 marks):** correct server sequence = **2**, client sequence = **2**, diagram with both sides = **2**, one-line job of each call = **2**.

---

### Q15 🔴 What happens if you call bind() in a TCP client? (asked: NCIT 2025 Q3b)

**Plain meaning:** a TCP client normally **does NOT call `bind()`**. When the client calls `connect()`, the kernel quietly picks a free **ephemeral port** for you and uses your machine's address as the source — a built-in, automatic "bind" (called an **implicit bind**).

**If you DO call `bind()` in a client:**
- You force a **specific local IP/port** as the source instead of the automatic ephemeral one.
- If that port is **already in use**, `bind()` fails with **`EADDRINUSE`**.
- Binding to a specific source IP forces traffic to leave via a **particular interface/network card**.
- It is only needed in special cases — e.g. an **FTP active-mode** client that must tell the server: "connect back to me on port 20000" (the client binds to 20000 first).

**Why the client doesn't usually bind (deep understanding):** the server needs a *known* port so clients can find it. The client's own port is never advertised to anyone who cares — so letting the OS pick a random ephemeral port is free and safe. Binding manually just risks conflicts.

**Why it's a common exam trick question:** many students think "every socket needs a bind". The correct answer is: *bind is for the server's well-known address; a client's source address is auto-chosen by the kernel.*

**Also asked (send/recv in UDP, sendto/recvfrom in TCP):**
- **TCP** normally uses `write()`/`read()` or `send()`/`recv()` — the connection already identifies both ends, so you don't repeat the address on every call.
- **UDP** uses `sendto()`/`recvfrom()` because each datagram may go to / come from a **different peer** — you must name (or learn) the peer address on *every* datagram.
- Technically you *could* use `sendto`/`recvfrom` on a connected TCP socket (the address would be ignored), but it is unnecessary and confusing.

---
**Marking scheme (6 marks):** why client skips bind (ephemeral) = **2**, consequences of explicit bind + EADDRINUSE = **2**, TCP-vs-UDP send functions = **2**.

---

### Q16 🟡★ What is a daemon? How do you daemonize a process in UNIX? (asked: NCIT 2025 Q4a)

**Plain meaning:** a **daemon** is a background program that **runs forever without anyone logged in** — it has no terminal, no keyboard, and usually high privileges. Examples: `sshd`, `httpd`, `syslogd`, `crond`. (Think "server process that lives in the background".)

A daemon typically:
- has **no controlling terminal** (cannot read from the keyboard),
- usually **runs with special privileges** (may need to bind to well-known ports),
- **starts at boot/login** and runs until shutdown,
- writes output to a **log file or syslog** (not the screen).

**To daemonize (standard steps + code):**
```c
fork();            /* 1. parent exits, child becomes the daemon  */
setsid();          /* 2. new session, detach from controlling terminal */
chdir("/");        /* 3. safe working directory (don't lock a mounted fs) */
umask(0);          /* 4. clear file-mode mask so files aren't too restrictive */
/* 5. redirect stdin/stdout/stderr to /dev/null */
```

**Why each step (understand, don't just memorise):**
1. **`fork()`** — spawns a child, then the parent exits immediately so the shell prompt returns. (Also makes the child non-session-leader, required for `setsid` to work.)
2. **`setsid()`** — creates a **new session**; the process now has **no controlling terminal** (no keyboard, no screen).
3. **`chdir("/")`** — moves to the root directory so the daemon doesn't keep a busy/locked filesystem as its current directory (otherwise that disk can never be unmounted).
4. **`umask(0)`** — clears the file-permission mask so files the daemon creates can have the full intended permissions.
5. **Redirection** — `open("/dev/null"); dup2(0,1); dup2(0,2);` sends stdin/stdout/stderr to the "null device" (a black hole), so no terminal interaction is possible and no stray output appears.

**Optional but common:** a **second `fork()`** (double-fork) so the daemon is *not* a session leader and can never re-acquire a controlling terminal.

```c
if (fork() > 0) exit(0);   /* 1 */
setsid();                  /* 2 */
chdir("/");                /* 3 */
umask(0);                  /* 4 */
open("/dev/null"); dup2(0,1); dup2(0,2);  /* 5 */
```

**What the result looks like:** an orphan child adopted by `init`/`systemd` (PID 1), in its own session with no terminal — it lives until killed, writing only to log files.

---
**Marking scheme (7 marks):** daemon definition = **2**, code/5 steps = **3** (1 each), why-each-step = **2**.

---

### Q17 🟡★ signal() and sigaction(), and signal handling in UNIX. (asked: Gandaki 2025 Q3a)

**Plain meaning:** **signals are the kernel's way of tapping your process on the shoulder** to say "this happened!" (SIGINT = Ctrl-C, SIGIO = socket ready, SIGCHLD = a child died, SIGTERM = please quit). You can **ignore** it, take the **default** action, or **catch** it by providing your own handler function `void handler(int)`.

**`signal(signum, handler)`** — the old, simple version, but **not portable/robust**:
- Behaviour differs across UNIX variants.
- It may **reset the handler to default** after catching once (BSD fixed this, others not).
- You cannot **block other signals** while your handler runs.

**`sigaction(signum, &act, &old)`** — the **modern, powerful, recommended** form:
```c
struct sigaction act;
act.sa_handler = my_handler;      /* or sa_sigaction for extra args */
sigemptyset(&act.sa_mask);        /* which signals to block during handler */
act.sa_flags = SA_RESTART;        /* auto-restart interrupted syscalls */
sigaction(SIGIO, &act, NULL);
```
Why it is better than `signal`:
- **`sa_flags`** — e.g. `SA_RESTART` auto-restarts a system call interrupted by the signal (avoids annoying `EINTR` errors).
- **`sa_mask`** — blocks other signals **while this handler runs** (prevents re-entrant races).
- **Portable** behaviour across all UNIX systems.

**Why do signals matter in network programming (three network uses):**
- **`SIGIO`** — signals "socket is ready" → the foundation of **signal-driven I/O**.
- **`SIGCHLD`** — sent to the parent when a child exits → used to **reap zombies** in a fork-based server (`waitpid` in the handler).
- **`EINTR`** — an interrupted `accept`/`read` returns -1 with `errno == EINTR`; you either loop-and-retry or use `SA_RESTART`.

---
**Marking scheme (7 marks):** what signals are = **1**, signal() form = **1**, sigaction() form = **2**, why sigaction better = **2**, network-programming uses = **1**.

---

### Q18 🟡★ fork() and exec() — creating processes. (asked: NCIT 2025 Q4a)

**Plain meaning:** `fork` **copies your running program** into a second process; `exec` **replaces** a process with a totally different program. Together they let a server run a new copy of itself for each client, or launch other programs (like a shell).

**`fork()`** — creates a **child process** that is an **exact copy** of the parent:
- The child gets its **own PID**; both run the *same* code from the point of `fork` onwards.
- Return values: **0** in the child, the **child's PID** in the parent, **-1** on failure.
- Used right after `accept()` so **each client gets its own server process**. (The parent goes back to `accept`; the child talks to that one client.)

```c
pid_t pid = fork();
if (pid == 0) {           /* child */
    handle_client(connfd);
    exit(0);
} else if (pid > 0) {     /* parent */
    close(connfd);        /* parent doesn't need the per-client socket */
}
```

**Why fork returns 0 in the child (common confusion):** the child needs to know it is the child, and the parent needs to know its child's PID (to wait on it later). 0 in the child and the real PID in the parent is simply the convention that makes both possible with one call.

**`exec()`** — **replaces the current process image** with a brand-new program and runs it from its entry point:
- On success it **never returns**; the PID stays the same, but the code/data/stack are completely new.
- Variants: `execl`, `execlp`, `execle`, `execv`, `execvp`, `execve` — they differ in how arguments and environment are passed.

**How they combine (classic pattern):**
```c
fork();   /* (a) create a child that is a copy of the parent */
exec();   /* (b) in the child, replace it with a new program (e.g. /bin/ls) */
```
- **fork + exec together = run a *different* program.** `fork` gives you the *process*, `exec` gives you the *program*. A shell does exactly this: fork a child, then exec `ls` in it.
- In a network server: `fork()` after `accept` services many clients in parallel; a client may then `exec()` a shell to let the user type commands (the `telnet`/`ssh` model).

**Zombies:** if a child exits and the parent never `wait()`s/`waitpid()`s on it, the child stays as a **zombie** (dead but still in the process table). A server must handle `SIGCHLD` and call `waitpid` to **reap** children, or it leaks resources.

---
**Marking scheme (6–7 marks):** fork() = **2**, exec() = **2**, fork+exec pattern and use in servers = **2**, zombies = **1**.

---

### Q19 🟢★ UNIX domain sockets & socketpair. (asked: short note)

**Plain meaning:** UNIX domain sockets are **sockets for *local* communication only** — two processes on the *same* machine talk using the socket API, but with **no IP and no network** involved. Instead of an IP address, the "address" is a **pathname** in the filesystem, e.g. `"/tmp/foo.sock"`.

**Features:**
- **Faster** than TCP/IP on the same host (no protocol headers, no routing, kernel-internal copy).
- Supports **stream (`SOCK_STREAM`)**, **datagram (`SOCK_DGRAM`)**, and **sequenced-packet** forms (the last keeps message boundaries and is reliable).
- Can **pass file descriptors** between processes via `sendmsg()`/`recvmsg()` (with a `SCM_RIGHTS` control message) — the receiver gets a *new* fd referring to the same open file. Used by X Window, systemd, and daemons.
- Used widely: `X11`, `PostgreSQL` (local connections), `systemd`, Docker socket.

**Why sockets (and not just pipes) for local IPC:** pipes are one-way and parent↔child only. Unix sockets give you a *two-way* channel between *any* two processes, with the same API as network sockets — so code written for TCP can be dropped straight onto a Unix socket.

**`socketpair()`** — makes **two connected sockets at once**, already linked to each other:
```c
int fds[2];
socketpair(AF_UNIX, SOCK_STREAM, 0, fds);
/* fds[0] and fds[1] are connected; write on fds[0], read on fds[1] */
```
- Gives a **bidirectional** pipe (two-way, unlike a normal one-way pipe) — handy for **parent↔child** IPC without needing a filesystem path.
- Faster/cleaner than a named socket for two related processes.

**vs TCP loopback:** for same-host IPC, prefer a Unix socket over TCP (`AF_INET` 127.0.0.1) — it avoids port conflicts, routing, and firewall overhead.

---
**Marking scheme (5–6 marks):** what it is (pathname, same-host) = **2**, stream/datagram forms = **1**, fd passing = **1**, socketpair = **1–2**.

---

### Q19b 🟡 Hostname & service name resolution (gethostbyname / getservbyname / getaddrinfo).

**Plain meaning:** machines talk in **IP addresses and port numbers**, but humans use **names** ("google.com", "http"). **Name resolution** is the translation in between — handing a name to the system and getting back the numbers the network understands. Think of it as the phone book of the Internet.

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
- One call gets name + service and returns a **linked list of `struct addrinfo`**, each already filled with a usable `sockaddr` for **IPv4 or IPv6** (pass `AF_UNSPEC` to let the system pick).
- **Thread-safe**, family-neutral, and the canonical new API.

```c
struct addrinfo hints, *res;
memset(&hints, 0, sizeof hints);
hints.ai_family = AF_UNSPEC;            /* IPv4 or IPv6 */
hints.ai_socktype = SOCK_STREAM;        /* TCP */
getaddrinfo("www.example.com", "http", &hints, &res);
/* walk res; each node has res->ai_addr, res->ai_addrlen */
freeaddrinfo(res);
```

**Why getaddrinfo is the modern choice (the "why"):** it combines `gethostbyname` + `getservbyname` into one call, handles both address families, and returns *ready-to-use* `sockaddr` structures — so you can go straight to `socket()`/`connect()` without manual conversion, and it's safe in multithreaded programs.

**Which to use:** for new code always `getaddrinfo` (IPv6-ready, thread-safe). Know `gethostbyname` because it still appears in old textbooks/exams, but state its limits when asked.

---
**Marking scheme (7 marks):** what resolution is = **1**, gethostbyname + hostent = **2**, getservbyname = **1**, getaddrinfo = **2**, limits/recommendation = **1**.

---

## Unit 3 — Advanced Unix

### Q20 🔴★ The five I/O models; which are synchronous? (asked: Gandaki 2025 Q4a)

**Plain meaning:** when your program `recv`s data, two things must happen: the kernel waits for a packet to arrive, then copies it into your buffer. The five I/O models differ only in **what your program is doing while it waits** — sleeping, asking again and again, or doing something else entirely.

**The two phases of every network read (this is the mental core of the whole question):**
1. **Wait for data to be ready** — the kernel waits for a packet to arrive (from the network card).
2. **Copy data from kernel to process** — once ready, the kernel copies data into the user's buffer (never instant, even with zero-copy tricks).

**Every model = a different answer to "what does the process do in phase 1?"**

| # | Model | Phase 1 (wait) | Phase 2 (copy) | Blocking? |
|---|---|---|---|---|
| 1 | **Blocking I/O** | Process **sleeps** (blocks in `recvfrom`) | Process **sleeps** while kernel copies | Yes — blocks on recvfrom |
| 2 | **Non-blocking I/O** | Process **polls** (`EWOULDBLOCK` until ready) | Kernel copies once ready | Yes — poll+recvfrom both block/sleep |
| 3 | **I/O multiplexing** (`select`/`poll`) | Process **blocks in select()** until one fd is ready | Process does `recvfrom` (kernel copies) | Yes — blocks in select, then blocks in recvfrom |
| 4 | **Signal-driven I/O** (`SIGIO`) | Kernel **sends SIGIO** when fd ready (process runs handler) | Process does `recvfrom` (kernel copies) | Yes — recvfrom blocks |
| 5 | **Asynchronous I/O** (`aio_read`, POSIX AIO) | Kernel waits for data | Kernel **copies + notifies process** (via signal/callback) | **No** — fully async |

**Which are synchronous? (the tricky part)**
A "synchronous" operation is one that **blocks the process until the whole operation completes** — i.e. the `recvfrom()` call does not return until the data is actually in your buffer. Models 1–4 are all **synchronous** — in each of them the actual `recvfrom()` call **blocks** until the data is copied. Only model **5 (asynchronous I/O)** is truly asynchronous — the process is *never* blocked waiting for data, because the kernel does everything (wait + copy) and notifies you later.

**Most common confusion (clarify for a mark):** "non-blocking" does NOT mean "asynchronous". A non-blocking socket still makes the process wait during the kernel copy — that's synchronous. Non-blocking only means the *phase-1 wait* doesn't tie you up. Asynchronous = the *entire* operation (wait + copy) happens without the process waiting at all.

**Which to use for many clients:** models 3 or 4 are the standard server models. Model 5 (POSIX AIO) exists on paper, but the real-world server winner is **model 3 (select/poll/epoll)** — portable, mature, and lets one process watch thousands of sockets.

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

**Plain meaning:** when no data has arrived yet, what does your `recv` do? **Blocking** I/O says *"I'll just sleep until data comes"*. **Non-blocking** I/O says *"Check now — if nothing's there, return and let me do something else"*.

**Blocking I/O (`SOCK_STREAM` default):**
- The `recvfrom()` call **sleeps** until data arrives AND is copied into your buffer. The process does nothing until then.
- **Advantage:** simplest code, no polling logic.
- **Disadvantage:** **ties up the thread entirely** — one thread handles *one* client at a time. A blocking server needs a thread per client (or `fork`) — scales poorly.
- Example: `read(fd, buf, n)` on a blocking socket returns `n` bytes read, or 0 if the peer closed, or -1 on error. It never returns "no data yet" — that just cannot happen.

**Non-blocking I/O (`O_NONBLOCK` or `FIONBIO`):**
- Set the socket non-blocking: `fcntl(fd, F_SETFL, O_NONBLOCK)`.
- Now `recvfrom()` **returns immediately** — if data isn't ready, it returns **-1** with `errno = EWOULDBLOCK` (or `EAGAIN`). If data *is* ready, it copies and returns the count.
- **Advantage:** the process can work on other things between attempts.
- **Disadvantage:** you must **poll repeatedly** (busy-wait) until data arrives → **wastes CPU**. Polling thousands of sockets in a tight loop burns 100% CPU doing nothing useful.

**The real reason non-blocking exists — combine it with `select()`:**
Non-blocking sockets are *not* meant for manual polling. Their true purpose is to be used **with `select`/`poll`**: `select` blocks efficiently until *one or more* sockets are ready, then you do a non-blocking read on the ready ones. Best of both worlds:
- `select` tells you which sockets are ready (efficient, no busy-wait).
- Non-blocking read on each ready socket returns immediately with data (or an unexpected `EWOULDBLOCK` if readiness was wrong — a "spurious readiness").

**Summary table:**
| | Blocking | Non-blocking |
|---|---|---|
| `recv` when no data | Blocks (sleeps) | Returns `EWOULDBLOCK` |
| CPU usage | Low (sleep) | High (polling) if busy-waiting |
| Thread per client? | Required | Not required |
| Use alone? | Simple, one client | Poor (wastes CPU) |
| Use with select? | Not needed | **Best combination** |

**Memory hook:** blocking = "sleep 'til it's there", non-blocking = "peek now, do something else, peek again". Alone, non-blocking wastes CPU; with `select`, it's the standard server pattern.

---
**Marking scheme (6 marks):** blocking explanation = **2**, non-blocking explanation = **2**, why-combine-with-select = **2**.

---

### Q22 🔴★ Explain signal-driven I/O, compare with I/O multiplexing. (asked: Gandaki 2025 Q4a, NCIT 2025 alternative)

**Plain meaning:** both models solve the same problem — "how does my program know when a socket has data?" **Signal-driven** = the kernel *interrupts you* with a signal ("data's here!"). **Multiplexing** = you sit and *wait* at a guard post (`select`) until the guard tells you which sockets are ready.

**Signal-driven I/O:**
- Your process enables the socket for `SIGIO`, installs a signal handler with `sigaction`, and goes about its business.
- When a **datagram arrives** (data ready in the kernel), the kernel sends **SIGIO** to your process. The **signal handler** then runs `recvfrom()` to read the data.
- This is an **interrupt model** — the kernel *tells you* when data is ready, instead of you asking.
- Real Linux implementation: uses `F_SETOWN` + `F_SETFL` with `O_ASYNC`:
  ```c
  int flags = 1;
  ioctl(fd, FIOASYNC, &flags);        /* enable async */
  fcntl(fd, F_SETOWN, getpid());      /* deliver SIGIO to this process */
  signal(SIGIO, my_handler);
  ```

**I/O Multiplexing (select/poll):**
- Your process calls `select()` or `poll()` with a set of file descriptors. It **blocks** inside `select` until one or more fds are ready.
- Then you do `recvfrom` on each ready fd (this `recv` returns immediately since data is there).
- No signals involved — it is purely a **blocking waiting loop**:
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
- However, signal-driven I/O has **lower latency** when data arrives while the process is sleeping — you don't have to be blocked in `select` to learn about it.

**Both are synchronous:** in both cases, the actual `recvfrom()` call (in the handler or after select) **blocks** while data is copied. Neither is *asynchronous* (model 5).

---
**Marking scheme (8 marks):** signal-driven explanation = **2**, select explanation = **2**, comparison table = **2**, which preferred/why = **1**, both-synchronous note = **1**.

---

### Q23 🔴★ I/O multiplexing & select(). (asked: NCIT 2025 Q4b)

**Plain meaning:** `select` lets **one process watch many sockets at once** and sleep until *any* of them is ready — instead of asking each socket "are you ready?" over and over. Like a hotel receptionist who takes a nap and wakes up only when a guest rings any bell.

**`select()` prototype:**
```c
int select(int maxfdp1, fd_set *readset, fd_set *writeset,
           fd_set *exceptset, const struct timeval *timeout);
// returns: number of ready fds, 0 on timeout, -1 on error
```

**Parameters explained (one line each):**
- `maxfdp1` — one more than the **highest-numbered** fd you're watching (e.g. fds 0, 3, 5 → pass 6). This is a *performance hint*: the kernel only checks fds 0..maxfdp1-1.
- `readset` — the fds you're watching for **read readiness** (data arrived, peer closed, or a listening socket has a pending connection).
- `writeset` — the fds you're watching for **write readiness** (enough buffer space to send without blocking).
- `exceptset` — the fds you're watching for **exceptional conditions** (e.g. out-of-band TCP data).
- `timeout` — how long to wait: `NULL` = wait forever; `5 sec` = wait 5 seconds; zero = return immediately (poll).

**Macros (all operate on `fd_set`):**
```c
FD_ZERO(&set);          /* clear the set */
FD_SET(fd, &set);       /* add fd to the set */
FD_CLR(fd, &set);       /* remove fd from the set */
FD_ISSET(fd, &set);     /* is fd in the set? (after select returns) */
```

**Typical pattern (follow the four macro steps every loop):**
```c
fd_set readset;
int maxfd = listenfd;
for (;;) {
    FD_ZERO(&readset);          /* 1. empty the set */
    FD_SET(listenfd, &readset); /* 2. add fds we care about */
    FD_SET(stdin_fd, &readset);
    select(maxfd + 1, &readset, NULL, NULL, NULL);  /* 3. sleep 'til any ready */

    if (FD_ISSET(listenfd, &readset)) {   /* 4. check which ones are ready */
        /* new connection arrived */
    }
    if (FD_ISSET(stdin_fd, &readset)) {
        /* user typed something */
    }
}
```

**Use cases:**
- A **single-threaded client** watching both **stdin** and its **socket** at once (e.g. a telnet client).
- A **single-threaded server** handling many sockets in one thread (a simple chat server).
- Combined with non-blocking sockets: do non-blocking reads on ready fds so a single slow read never blocks the whole `select`.

**Limits of select (and why `poll`/`epoll` exist):**
- `fd_set` has a fixed maximum size (typically `FD_SETSIZE = 1024`).
- It must be **re-initialized** before every `select` call (select overwrites it).
- `maxfdp1` can be no larger than `FD_SETSIZE`.
- For very large numbers of sockets, `poll` or Linux `epoll` scale better.

---
**Marking scheme (8 marks):** what multiplexing is = **1**, select prototype + params = **2**, macros = **1**, code pattern = **2**, use cases + limits = **2**.

---

### Q24 🔴★ Mechanisms to handle multiple clients in UNIX — with code. (asked: Gandaki 2025 Q3b, NCIT 2025 Q5a)

**Plain meaning:** one server must serve *many clients at once*. There are three standard strategies: **give each client its own copy of the process** (fork), **check all sockets in one loop** (select), or **give each client its own thread**.

**Approach 1 — `fork()` per client (process-per-connection):**
- After `accept()`, the server `fork()`s a child process for each client. The child handles that client; the parent goes back to `accept()` for the next one.
- **Advantage:** simple, each client is isolated (one crash doesn't kill the server).
- **Disadvantage:** one process per client uses lots of memory — doesn't scale to thousands of clients.
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
- One process watches *all* sockets with `select()` or `poll()`. When data arrives on any socket, read it. No fork, no threads.
- **Advantage:** low memory (one process), good for thousands of idle connections.
- **Disadvantage:** more complex code; one slow handler blocks everything.
- **Add non-blocking sockets** so a read on a ready socket never blocks unexpectedly.

**Approach 3 — threads (`pthread_create`) per client:**
- Like fork, but lighter-weight: threads **share the same address space** (no copying the whole process, no zombie issue).
- **Advantage:** lighter than processes, good concurrency.
- **Disadvantage:** shared memory → **race conditions**; need mutexes/locks. One thread crash can kill all threads.

**Comparison:**
| Mechanism | Pros | Cons | Best for |
|---|---|---|---|
| fork() | Isolation, simple | Slow, zombie issues, 1-conn/memory | Small servers |
| select() | Scalable, low memory | Complex, single-point-of-failure | Thousands of clients (chat, proxy) |
| pthreads | Lightweight, shared memory | Race conditions, need locks | Moderate concurrency, shared data |

**How to choose (exam-ready):** few clients and want simplicity → fork; many idle clients with low memory budget → select; moderate load where threads can share state → pthreads.

**Code example (fork):** shown above.

---
**Marking scheme (8 marks):** 3 approaches explained = **3**, code example = **2**, comparison table = **2**, zombies note = **1**.

---

### Q25 🟡★ Broadcast vs multicast. (asked: NCIT 2025 Q4a)

**Plain meaning:** both send **one packet to many receivers** (one-to-many). The difference is *how many* receivers: **broadcast shouts to everyone on your street**; **multicast only to people who joined your "club"**.

**Broadcast (UDP only):**
- Sends a datagram to **every host on the local subnet** (e.g. `255.255.255.255` = limited broadcast).
- Must enable it: `setsockopt(s, SOL_SOCKET, SO_BROADCAST, &on, sizeof(on))` — the socket has to explicitly allow broadcast.
- **Routers do NOT forward** broadcast packets → it is **local-subnet only**.
- **Wastes resources:** every host must process the packet even if it's not interested (CPU overhead, interrupt storms).
- Use case: **network-wide discovery/announce** (finding a printer on the LAN, DHCP discovery).

**Why the OS forbids broadcast by default:** sending a broadcast by mistake would spam every host on the subnet, so the kernel makes you explicitly opt in. That's why you must set `SO_BROADCAST` first.

**Multicast:**
- Sends to a **group of hosts** that have "subscribed" (joined) to a multicast group address.
- **Group address:** Class D IP range `224.0.0.0` to `239.255.255.255`. A multicast address is like a "virtual club" — you join it, you receive its packets.
- Hosts join a group: `setsockopt(s, IPPROTO_IP, IP_ADD_MEMBERSHIP, &mreq, ...)`. They leave with `IP_DROP_MEMBERSHIP`.
- `IP_MULTICAST_TTL` controls how far packets travel (how many routers forward them).
- **NIC hardware filtering:** network cards accept only packets for their group, so non-members never see the packet — **far more efficient** than broadcast.
- Use case: **video conferencing**, streaming, live stock tickers, multiplayer games — anywhere only *interested* hosts should receive the data.

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

**Plain meaning:** a socket is a kernel object with many **configuration switches**. `setsockopt`/`getsockopt` are how your program **flips those switches** (set) or **reads them** (get).

```c
int setsockopt(int s, int level, int optname, const void *optval, socklen_t optlen);
int getsockopt(int s, int level, int optname, void *optval, socklen_t *optlen);
```
- `level` — protocol layer: `SOL_SOCKET` (general), `IPPROTO_TCP`, `IPPROTO_IP`, `IPPROTO_IPV6`.
- `optname` — the specific option to get/set.
- `optval` — a pointer to the value (usually an `int`).

**The four key options you must know:**

**1. `SO_REUSEADDR` — reuse a local address/port:**
- **The problem:** after a server closes, its address sits in **TIME_WAIT** (2×MSL, ~2 minutes). Restart right away and `bind()` fails with `EADDRINUSE` — the port looks "still in use" because of old orphaned packets.
- **The fix:** `SO_REUSEADDR` lets you bind even while the address is in TIME_WAIT. **Essential for servers that restart often** (HTTP servers, database servers).
- Example: `setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));`

**2. `SO_BROADCAST` — allow sending broadcast messages:**
- **The problem:** by default, a socket **rejects** broadcast sends (`EACCES`).
- **The fix:** setting `SO_BROADCAST` allows `sendto()` to a broadcast address.
- UDP only; routers do not forward broadcast (local-subnet only). Required *before* sending; otherwise you get `EACCES`.

**3. `SO_KEEPALIVE` — TCP keepalive probes:**
- After **2 hours of inactivity**, the OS sends a keepalive **probe** to check the peer is still alive (see Q28 for full detail).
- Peer ACKs → connection alive, reset the timer, wait another 2 hours.
- Peer crashed → RST received → `ECONNRESET`.
- No response → up to **8 probes**, **75 seconds apart** (~10 minutes) → `ETIMEDOUT` (or `EHOSTUNREACH` if ICMP unreachable).
- Useful for long-lived connections (SSH, database, VPN) to detect dead peers and free resources.

**4. `SO_LINGER` — control close() behaviour (full detail in Q27):**
- Controls what `close()` does with unsent data.
- Three modes: default (return immediately), abort (RST, discard data), linger (wait up to `l_linger` seconds for data to be acked).

---
**Marking scheme (8 marks):** setsockopt/getsockopt = **2**, SO_REUSEADDR = **2**, SO_BROADCAST = **1**, SO_KEEPALIVE = **2**, SO_LINGER = **1**.

---

### Q27 🟡 SO_LINGER in detail.

**Plain meaning:** what should happen to **data still waiting to be sent** when you `close()` a TCP socket? `SO_LINGER` gives you three choices: let the OS handle it quietly, wait for it to be delivered, or cancel everything with a hard reset. It's deciding whether closing a connection is like *finishing your sentence politely* or *slamming the phone down*.

Configured with:
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
2. **Linger with timeout (on=1, linger=5):** you **must** ensure data is delivered before the process exits — a database flush, a financial transaction. You're willing to wait 5 seconds for the peer to ACK.
3. **Abort (on=1, linger=0):** you want to **terminate immediately** and discard unsent data — cancelling an operation, deliberately sending an error via RST.

**Common error:** setting linger without understanding it — calling `close()` on a socket with unsent data and a long linger time can **block the process for several seconds**, which is catastrophic in a high-performance server (it stalls one thread and eats up one connection slot).

**Note on TCP RST:** when the linger timeout expires (or linger=0), the OS sends a TCP **RST** (reset) segment — the peer must abort immediately and sees `ECONNRESET` on its next read/write.

---
**Marking scheme (5–6 marks):** struct definition = **1**, three modes = **3**, when to use each = **1–2**.

---

### Q28 🟢 SO_KEEPALIVE in detail.

**Plain meaning:** `SO_KEEPALIVE` makes the OS send a tiny **"are you still there?"** probe on idle connections, so dead peers get detected instead of the connection hanging around forever. Like a waiter checking a table that hasn't ordered in two hours.

**Default timers (Linux):**
1. After **2 hours** of inactivity → send first keepalive **probe**.
2. Peer ACK → connection alive, **reset 2-hour timer**.
3. No ACK → wait **75 seconds**, send another probe. Repeat up to **8 times**.
4. After 8 failed probes (~10 minutes total) → report **`ETIMEDOUT`**, or **`EHOSTUNREACH`** if an ICMP unreachable was received.

**What the peer sees:**
- The peer sees a **regular TCP segment** with `ACK` set and the **same sequence number** as the last data (a zero-length segment). If alive, it ACKs immediately.
- If the peer **crashed and rebooted**: it replies with `RST` → `ECONNRESET`.
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

**Warning:** `SO_KEEPALIVE` adds a tiny overhead (one extra byte per probe), and NAT devices/firewalls may time out idle connections before the 2-hour TCP timer fires — use shorter app-level heartbeats (ping/pong) in those cases.

---
**Marking scheme (4–5 marks):** what it is = **1**, timers = **1**, what peer sees = **1**, why-useful = **1**, tuning (bonus) = **1**.

---

### Q29 🟡★ Syslog & logging from network applications. (asked: NCIT 2025 Q4b — with block diagram)

**Plain meaning:** a daemon has no terminal to print to, so how does it report errors? **Syslog** is UNIX's central "message board" — any program can post messages to it, and the `syslogd` daemon writes them to the right log files (and optionally forwards them to other machines).

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

**Why a daemon *needs* syslog (the reason):** after daemonization (Q16), fds 0,1,2 point to `/dev/null`. The daemon literally cannot print anywhere. Syslog gives it a permanent, structured output channel that the OS manages — plus timestamps and categories for free.

**The three functions:**
- `openlog(ident, options, facility)` — open the connection to syslog; `ident` is your program's name, `facility` is the category (LOG_AUTH, LOG_DAEMON, LOG_LOCAL0-7, etc.).
- `syslog(priority, format, ...)` — post a message; `priority` combines `facility | level` (LOG_ERR, LOG_WARNING, LOG_INFO, etc.).
- `closelog()` — close the connection.

**Priorities (most → least severe):**
`LOG_EMERG` > `LOG_ALERT` > `LOG_CRIT` > `LOG_ERR` > `LOG_WARNING` > `LOG_NOTICE` > `LOG_INFO` > `LOG_DEBUG`

**How it helps network applications:**
- A daemon cannot write to `stdout`/`stderr` (no terminal) — syslog is its **output channel**.
- `/etc/syslog.conf` controls where messages go (which file, which host) — no code change needed to redirect logs.
- Centralised → one daemon collects logs from all processes; logs are **timestamped**, **facility-tagged**, and can be **rotated**.

**Network option:** a client can send syslog messages to a **remote server** (UDP port 514) — so multiple machines share one log server, invaluable for distributed systems (if one machine dies, its logs are still safe on the central server).

---
**Marking scheme (7 marks):** block diagram = **3**, openlog/syslog/closelog = **2**, priorities/facilities = **1**, why-network-daemons-need-syslog = **1**.

---

### Q30 🟡★ How to secure a network application. (asked: short note "Wrapper function…")

**Plain meaning:** securing a network app means keeping **unwanted clients out** (access control) and keeping **data safe while travelling** (encryption). The exam specifically wants the three access-control methods — **hostname, IP number, and wrapper program** — plus TLS.

**1. By hostname/domain:**
- Allow connections only from trusted **hostnames** — resolve the client's IP to a hostname with `gethostbyaddr()`, then check it against an allow-list.
- **Limitation:** DNS can be **spoofed** (an attacker forges a DNS reply to match a trusted hostname) — this alone is weak. Use it as part of a layered defence, not alone.

**2. By IP number:**
- Restrict connections by source IP using **`/etc/hosts.allow` + `/etc/hosts.deny`** (TCP wrappers) or **firewall rules** (`iptables`, `nftables`, `pf`).
- Simpler and stronger than hostname-only — IP spoofing is harder than DNS spoofing.
- But: IPs can still be spoofed, and VPN/NAT make IP-based control less reliable.

**3. Wrapper program (TCP wrappers concept — the exact short note):**
- A **wrapper** is a small front-end that intercepts the incoming connection *before* the real service starts.
- It checks the client against a policy (hosts.allow/deny, custom rules) and **only if allowed** launches the real service and relays the connection to it.
- This adds access control **without modifying the server application** — the server binary stays untouched; the wrapper does the gating.
- Classic example: `inetd` + TCP wrappers (`in.tcpd`) — `inetd` accepts the connection, runs `tcpd`, `tcpd` checks allow/deny, then runs the real `telnetd`, `ftpd`, etc.
- **Modern equivalent:** systemd socket activation + firewall rules.

**Why the wrapper is clever (understanding):** the *same* server program can be protected without changing a line of its code — you put an "armed guard" in front of it. That's the whole point of wrappers.

**4. Encryption — TLS/SSL:**
- **Encrypts** all data in transit (AES, ChaCha20 — symmetric ciphers).
- **Authenticates** the server via **certificates** signed by a trusted CA.
- **Detects tampering** via message authentication codes (HMAC).
- Without TLS, access control only restricts *who* can connect — data is still in the clear for eavesdropping and modification.

**Best practice:** combine all layers — firewall/TCP wrappers for access control **+** TLS for confidentiality/integrity. Access control gates the door; TLS protects the data beyond the door.

---
**Marking scheme (6–8 marks):** hostname = **2**, IP = **1**, wrapper program (concept + example) = **2**, TLS/SSL = **2**, best practice layering = **1**.

---

## Unit 4 — Winsock Basics

### Q31 🔴★ How is Winsock different from UNIX sockets? + static vs dynamic linking. (asked: NCIT 2025 Q6a — 7 marks)

**The short answer:** Berkeley sockets were invented on UNIX. **Winsock is simply the same socket idea rewritten for Windows.** Because Windows is *not* UNIX, the same concepts have different names, a different socket type, different error handling, and one extra "setup step" you must do first.

**Simple mental model:** the socket API is like a language. Both UNIX and Windows speak it, but Windows spells some words differently and insists you "introduce yourself" (init) before talking.

| Feature | UNIX | Windows (Winsock) |
|---|---|---|
| What a socket is | an `int` (0,1,2,...) | a `SOCKET` (special number) |
| Close it with | `close(fd)` | `closesocket(s)` |
| Send/receive with | `read()`/`write()` or `send()`/`recv()` | only `send()`/`recv()` |
| Errors reported via | `errno` (a variable) | `WSAGetLastError()` (a function) |
| Before the first socket call | nothing needed — works instantly | **must call `WSAStartup()` first** |
| After the last socket call | nothing needed | **must call `WSACleanup()`** |
| Header file | `<sys/socket.h>` | `<winsock2.h>` |
| Extra I/O models | select, poll, epoll | WSAAsyncSelect, WSAEventSelect, overlapped, IOCP |

**Why the setup step exists — DLLs:** on UNIX the network code always lives *inside the kernel*, ready to use. On Windows the network code lives in a **DLL file** (`ws2_32.dll`) that must be **loaded into your program first**. That "load + agree on a version" is exactly what `WSAStartup()` does. So Winsock = socket programming that first loads a library.

**One thing to memorise:** UNIX gives you an *int* and does nothing to prepare. Windows gives you a `SOCKET` handle, demands `WSAStartup` first, wants `closesocket` at the end, and reports failures via a function, not a variable.

**Static vs dynamic linking (how you attach that library):**

| | Dynamic (DLL) | Static |
|---|---|---|
| Where the library code lives | in a separate `.dll` file | copied inside your `.exe` |
| Executable size | small | big |
| Updating it | just replace the DLL | must recompile everyone |
| Runs everywhere? | fails if DLL missing/wrong version ("DLL hell") | always runs |
| Shared by programs | yes, many apps share one DLL | no, each program has its own copy |

**Think of it as:** dynamic = borrow a book from the library (small bookbag, but the library must have the book); static = photocopy the whole book (big bag, but you always have it).

**Mini example showing the differences:**
```c
#include <winsock2.h>                    // Windows header
#pragma comment(lib, "ws2_32.lib")       // tells the compiler to use the DLL

WSADATA wd;
WSAStartup(MAKEWORD(2,2), &wd);          // STEP 0: load the DLL (UNIX has no step 0)
SOCKET s = socket(AF_INET, SOCK_STREAM, 0);
// ... normal socket code ...
closesocket(s);                          // not close()
WSACleanup();                            // end: unload the DLL
```

---
**Marking scheme (7 marks):** comparison table = **3**, DLL + WSAStartup idea = **2**, static-vs-dynamic = **2**.

---

### Q32 🔴★ WSAStartup / WSACleanup; role of setup(), cleanup(). (asked: NCIT Q6b, Gandaki Q5a)

**Plain meaning:** on Windows the network code is in a DLL. Before you use *any* socket function you must (1) load that DLL and (2) agree on which Winsock version you want. That is exactly what `WSAStartup()` does. When your program is done, `WSACleanup()` unloads it. They are a **pair — one cleanup per startup** (like open/close a file).

**`WSAStartup(MAKEWORD(2,2), &wsadata)` step by step:**
1. `MAKEWORD(2,2)` = "I want version 2.2" (major 2, minor 2).
2. Windows loads `ws2_32.dll` (the network library) into your program.
3. It fills `wsadata` (a `WSADATA` struct) with: the version actually loaded, a description string, and limits.
4. It returns `0` if all went well — **you must check this**. If it fails, no network code will work, so exit.

**Why check the return value (understand):** `WSAStartup` can load an *older* Winsock than you asked for (if the system only has 1.1). The `WSADATA.wVersion` field tells you what you *actually* got. If the request version doesn't match what's loaded, some functions may not exist — so a careful program verifies both.

**`WSACleanup()`:**
- Unloads the DLL / frees network resources.
- Counting rule: Windows keeps a **reference count** — each `WSAStartup` adds 1, each `WSACleanup` subtracts 1. The DLL is truly unloaded only when the count reaches 0.
- Forgetting it wastes a little memory (Windows cleans up at program exit anyway) — but always pair them for correctness.

**"setup() / cleanup()" in the exam = `WSAStartup()` / `WSACleanup()`.** They exist only on Windows; on UNIX the kernel always has the socket code ready, so there is nothing to set up.

**Classic beginner mistake:** call any socket function before `WSAStartup`, and every call returns error **`WSANOTINITIALISED`** ("you forgot to init").

---
**Marking scheme (6–7 marks):** WSAStartup explained = **3**, WSACleanup explained = **2**, reference count + error = **1–2**.

---

### Q33 🔴★ Major DLLs needed for a Winsock app. (asked: NCIT Q6b — 8 marks)

**Plain meaning:** Windows splits its network stack into several DLL files. Your program usually deals with only one — `ws2_32.dll` — the rest are loaded automatically when needed.

| DLL | Job | Loaded when |
|---|---|---|
| `ws2_32.dll` | **the main one** — the whole Winsock 2 API | by `WSAStartup` |
| `wsock32.dll` | old Winsock 1.1 (32-bit) | an old program asks for 1.1 |
| `winsock.dll` | ancient Winsock 1.1 (16-bit, Windows 3.1) | ancient programs only |
| `mswsock.dll` | Microsoft extras: `AcceptEx`, `TransmitFile`, ... | you call one of those extras |
| `wshtcpip.dll` | TCP/IP helper functions | helper functions are used |
| `msafd.dll` | links Winsock to the kernel (the "engine") | internally by the stack |
| `wship6.dll` | IPv6 helpers (`WSAAddressToString`, ...) | you do IPv6 operations |

**Which one do you actually care about?** `ws2_32.dll`. The rest are behind the scenes. If an exam asks "which DLL do you link against", the answer is `ws2_32.lib` (which pulls in `ws2_32.dll` at run time).

**DLL concept (know this):** a DLL is a library that lives in its own file and is attached to your program at *run time* — your `.exe` does not contain that code, it *calls into* the DLL.
- **Pros:** smaller programs, easy updates (replace one DLL fixes all apps), code shared by many apps.
- **Cons:** if the DLL is missing or the wrong version, your program won't start — the famous "**DLL hell**".

**Static vs dynamic (asked together):** static = library code copied *inside* your `.exe` (bigger, always runs); dynamic = lives in the `.dll` (smaller, needs the DLL present).

---
**Marking scheme (8 marks):** DLL table = **4**, ws2_32 = **1**, mswsock extras = **1**, DLL pros/cons = **1**, static vs dynamic = **1**.

---

### Q34 🔴★ Winsock TCP & UDP client-server sequences with code. (asked: Gandaki Q5b — 8 marks)

**TCP server — the whole recipe (memorise this order):**
```
WSAStartup → socket → bind → listen → accept → recv/send → closesocket → WSACleanup
  init      make      claim    wait in   pick up   talk     hang up     unload
  (load)    a raw     an       line      the       (data)
  the DLL   socket    address  standing  waiting
```

```c
// Winsock TCP server
#include <winsock2.h>
#pragma comment(lib, "ws2_32.lib")

int main() {
    WSADATA w; WSAStartup(MAKEWORD(2,2), &w);   // 1. init: load the network DLL

    SOCKET s = socket(AF_INET, SOCK_STREAM, 0); // 2. create a raw socket

    SOCKADDR_IN sa;                             // 3. prepare the address:
    sa.sin_family = AF_INET;                    //    - IPv4
    sa.sin_port = htons(5150);                  //    - port 5150
    sa.sin_addr.s_addr = htonl(INADDR_ANY);     //    - accept on any IP
    bind(s, (SOCKADDR*)&sa, sizeof(sa));        // 4. claim this address

    listen(s, 5);                               // 5. "wait in line" (5 queued)

    SOCKADDR_IN cli; int clen = sizeof(cli);
    SOCKET cs = accept(s, (SOCKADDR*)&cli, &clen); // 6. pick up the first caller (blocks)

    char buf[1024]; int n = recv(cs, buf, sizeof(buf), 0); // 7. read client's message
    send(cs, buf, n, 0);                        // 8. echo it back

    closesocket(cs); closesocket(s);            // 9. hang up
    WSACleanup();                               // 10. unload the DLL
}
```

**TCP client:**
```
WSAStartup → socket → connect → send/recv → closesocket → WSACleanup
```
(no `bind` — the OS freely picks a port for you; no `listen`/`accept` — the client is not a server)

```c
SOCKET s = socket(AF_INET, SOCK_STREAM, 0);
SOCKADDR_IN sa; sa.sin_family = AF_INET; sa.sin_port = htons(5150);
inet_pton(AF_INET, "127.0.0.1", &sa.sin_addr);
connect(s, (SOCKADDR*)&sa, sizeof(sa));         // dial the server
send(s, "hello", 5, 0);
recv(s, buf, sizeof(buf), 0);
closesocket(s);
```

**UDP — no queue, no connection (connectionless):**
- Server: `WSAStartup → socket → bind → recvfrom → closesocket → WSACleanup` (no `listen`/`accept`).
- Client: `WSAStartup → socket → sendto → closesocket → WSACleanup` (no `bind`, no `connect` — each `sendto` names the destination).

**The one big TCP-vs-UDP difference:** TCP needs `listen()` + `accept()` to build a connection, then you talk with `recv`/`send` on it. UDP skips the queue entirely — the server just binds and waits for datagrams with `recvfrom`, and each client `sendto` already names where it is going.

---
**Marking scheme (8 marks):** TCP server code = **3**, TCP client = **2**, UDP sequence = **2**, TCP-vs-UDP difference = **1**.

---

### Q39 🟡 Graceful close in Winsock.

**Plain meaning:** "graceful close" = tell the other side *"I'm done sending"* **politely** (data delivered first) instead of just hanging up violently.

**The two-step close:**
```c
shutdown(s, SD_SEND);   // step 1: "no more data from me" → sends a polite FIN
// (optionally keep receiving what the peer still sends)
closesocket(s);         // step 2: actually release the socket handle
```

**Why two steps (understand):**
- `shutdown(SD_SEND)` sends the TCP **FIN** ("I'm finished *sending*"), but the socket still exists — you can keep **receiving** (this is the half-close).
- `closesocket()` frees the handle for good. Any data already queued to send is still delivered; but if you just want to stop immediately you'd use `SO_LINGER` + RST instead.
- If you call only `closesocket`, the OS *may* throw away unsent buffered data or send a harsh **RST** (abort) instead of the polite FIN — the peer then sees an error.

**The three `shutdown` options:**
| Option | Meaning |
|---|---|
| `SD_SEND` | stop sending (polite FIN) — most common |
| `SD_RECEIVE` | stop receiving (RST if the peer still sends) |
| `SD_BOTH` | stop both directions |

**Full graceful sequence:** server finishes → `shutdown(SD_SEND)` (FIN) → server `closesocket` → client's next `recv` returns 0 (EOF = "server done") → client closes its side too.

---
**Marking scheme (5 marks):** shutdown explained = **2**, closesocket vs shutdown = **1**, half-close idea = **1**, graceful vs abrupt = **1**.

---

### Q40 🟢 WSAEnumProtocols / WSAAccept / WSAConnect (Winsock extensions).

**Plain meaning:** these three are Windows-only extras (not in the old Berkeley API) that add a power feature each.

- **`WSAEnumProtocols` — "list what's installed."** Returns an array of `WSAPROTOCOL_INFO` structs, one per protocol (TCP, UDP, ...). Each describes family, socket type, and capabilities. Use it to discover which protocols exist and pick the right one.
```c
WSAEnumProtocols(lpiProtocols, lpProtocolBuffer, lpdwBufferLength);
```

- **`WSAAccept` — "accept, but check the visitor first."** Like `accept`, but you pass a **condition function** that is called *before* the connection is accepted:
  - `CF_ACCEPT` — let the client in,
  - `CF_REJECT` — refuse (sends RST to the client),
  - `CF_DEFER` — postpone the decision.
  Useful for filtering/rejecting clients before accepting them.
```c
WSAAccept(s, addr, addrlen, conditionFunction, callbackData);
```

- **`WSAConnect` — "connect, with extra requirements."** Like `connect`, but lets you attach **caller data** (`lpCallerData`) for the peer, and **QoS** parameters (`lpSQOS`/`lpGQOS`, e.g. bandwidth/latency) — useful for real-time/multimedia apps.
```c
WSAConnect(s, name, namelen, lpCallerData, lpCalleeData, lpSQOS, lpGQOS);
```

**When to use:** WSAEnumProtocols for protocol discovery, WSAAccept for connection filtering, WSAConnect for QoS. Most normal apps just use plain `accept`/`connect` — these are advanced extras.

---
**Marking scheme (4–5 marks):** WSAEnumProtocols = **1**, WSAAccept = **2**, WSAConnect = **1**, when to use = **0–1**.

---

## Unit 5 — Advanced Winsock

### Q35 🔴★ What is overlapped I/O in Winsock? How does it support async? (asked: NCIT 2025 Q7a — 7 marks)

**Plain meaning:** in normal (blocking) I/O you *wait* for each `send`/`recv` to finish — do one, wait, do the next. **Overlapped I/O** lets you *fire off many socket operations at once* and get a "done" signal later, so your thread is never stuck waiting. It is the trick behind high-speed servers.

**Analogy:** blocking I/O = ordering one meal and standing at the counter until it's ready. Overlapped I/O = ordering 10 meals, then doing everything else, and being *called* (or getting a bell ring) as each meal becomes ready.

**How it works (the steps):**
1. Create the socket as overlapped: `WSASocket(..., WSA_FLAG_OVERLAPPED)` — forget this flag and overlapped calls fail.
2. Launch work with the overlapped calls: `WSASend`, `WSARecv`, `WSARecvFrom`, `WSAIoctl`, `AcceptEx`. Every call passes a **`WSAOVERLAPPED`** struct (a small "box" holding a Win32 event + status info).
3. One of two outcomes:
   - The operation finishes **instantly** → function returns TRUE, done.
   - It needs time → function returns `SOCKET_ERROR` with error **`WSA_IO_PENDING`**. **This is NOT a failure** — it means "your job is queued, I'll tell you when it's done."
4. When each queued job finishes, you are told by one of two mechanisms:
   - an **Event object** gets signaled (check with `WaitForSingleObject`/`WaitForMultipleObjects`), or
   - a **completion routine** (a callback function you passed) gets called automatically.
5. After a signal, `WSAGetOverlappedResult()` tells you how many bytes actually moved.

**Why this means "asynchronous":**
- Your thread is **never blocked** — it issues 10 operations, then goes and does useful work.
- **One thread can manage hundreds of outstanding operations** — no need for a thread *per connection* (expensive).
- Best single-thread throughput of all the Winsock I/O models.

**The key mental shift:** a `WSA_IO_PENDING` return is *good news*, not an error. Beginners panic and treat it as a failure, then the whole program breaks. The pattern is: queue the work → wait for completion events → collect results.

**Where IOCP fits:** attach overlapped sockets to an **I/O Completion Port**, and the OS runs a **thread pool** for you — completed operations are handed to idle threads automatically. That is how servers handle thousands of connections.

---
**Marking scheme (7 marks):** what overlapped I/O is = **2**, WSA_FLAG_OVERLAPPED + WSAOVERLAPPED = **2**, event/callback completion = **2**, thread-never-blocks advantage = **1**.

---

### Q36 🟡★ Event-driven programming & WSAEventSelect. (asked: NCIT Q7a alt, Gandaki Q6b)

**Event-driven programming (plain):** instead of "do step 1, then 2, then 3...", the program *waits for things to happen* (events) and reacts. Network servers are event-driven because you never know which socket will need attention next.

**Think of it as a receptionist:** instead of pestering each guest "are you ready?", the receptionist sits and waits — guests ring a bell when they need something. Each socket that has data "rings" your event.

**WSAEventSelect = "tell me, via an event, when my socket needs me":**
```c
WSAEVENT h = WSACreateEvent();                  // 1. create an "event" object (a bell)
WSAEventSelect(s, h, FD_READ | FD_WRITE | FD_CLOSE); // 2. ring the bell when readable/writable/closing
```

**The event loop (how you wait):**
1. Create one event per socket.
2. `WSAWaitForMultipleEvents(n, events, ...)` — **block until any bell rings**.
3. `WSAEnumNetworkEvents(s, h, &ne)` — ask "which socket, and what happened?" → it fills FD_READ / FD_WRITE / FD_CLOSE flags.
4. Handle it: FD_READ → `recv`/`WSARecv`; FD_WRITE → `send`; FD_CLOSE → clean up.

**Why the third step is needed (common confusion):** `WSAWaitForMultipleEvents` only tells you *some* event fired — not WHICH socket or WHICH type of readiness. `WSAEnumNetworkEvents` is what answers both questions. Always call it after a wait.

**Key facts:**
- Unlike WSAAsyncSelect, **no window is needed** — works in console apps and services.
- One thread can wait on up to **64 events**.
- After `WSAEventSelect`, the socket automatically becomes **non-blocking**.

---
**Marking scheme (6 marks):** event-driven concept = **2**, WSAEventSelect mechanism = **2**, code pattern = **1**, no-window advantage = **1**.

---

### Q37 🟡★ WSAAsyncSelect vs WSAEventSelect. (asked: NCIT alt)

**One idea, two delivery channels:** both tell you "your socket is ready" — the only difference is *how* the message reaches you.

| | WSAAsyncSelect (older, Winsock 1.1) | WSAEventSelect (newer, Winsock 2.0) |
|---|---|---|
| How you're told | by a Windows **message** sent to a **window** | by a Win32 **event** (a flag/button) being set |
| Needs a window? | **Yes** — needs a message loop (HWND) | **No** — works in console/service apps |
| Best for | GUI programs that already have a window | console servers, background services |
| How you react | handle the `WM_SOCKET` message in the window procedure | `WSAWaitForMultipleEvents` + `WSAEnumNetworkEvents` |
| Socket mode | becomes non-blocking | becomes non-blocking |
| Limitation | message queue can overflow with many sockets | max 64 events per thread |

**Simple way to remember:** WSA**Async**Select uses Windows **message**s (think "GUI"), WSA**Event**Select uses Win32 **event objects** (think "no GUI needed"). Both are Windows-only and both push the socket into non-blocking mode.

**Memorise this one line:** both are Windows-only, both make the socket non-blocking — they differ only in **how** they notify you (a window message vs an event object).

---
**Marking scheme (5 marks):** what both are = **1**, difference table = **3**, when to use each = **1**.

---

### Q38 🟡★ WSAPoll vs select. (asked: Gandaki Q6b alt)

**Plain idea:** both are "check many sockets at once" tools — you say "tell me which of these sockets are ready", and the call blocks until at least one is. The difference is *how* you describe the list of sockets to watch.

| | `select` | `WSAPoll` |
|---|---|---|
| You give it | an `fd_set` — a **fixed-size list** (max 64 on Windows) | an **array of `WSAPOLLFD`** — you choose the size |
| Size limit | stuck at `FD_SETSIZE` (64) | **no fixed limit** |
| After the call | it **overwrites** your sets — you must rebuild them for every call | keeps your array — just read each socket's `.revents` |
| Reading results | check bits with `FD_ISSET` | each socket's `revents` field |
| Portability | works on Windows AND Unix | Windows only (mirror of Unix `poll`) |

**When it matters:**
- 100 sockets with `select` → **impossible** on Windows (limit 64); you would need several threads.
- `WSAPoll` → one array of 100 `WSAPOLLFD`, one call, done.

**The `WSAPOLLFD` struct (just 3 fields):**
```c
typedef struct pollfd {
    SOCKET fd;        // which socket to watch
    SHORT  events;    // what you want to know (POLLIN = read-ready, POLLOUT = write-ready)
    SHORT  revents;   // filled by WSAPoll: what actually happened
} WSAPOLLFD;
```

- Many sockets (on Windows) → **WSAPoll**; portability to Unix or few sockets → **select**.

---
**Marking scheme (5–6 marks):** what both are = **1**, comparison table = **3**, WSAPOLLFD struct = **1**, when to use = **0–1**.

---

### Q41 🟡★ 5 Winsock I/O models.

**The question all five answer:** "how do I find out when my socket is ready to read or write — without wasting time?" Each model is a different *answer*, from simple to super-scalable.

1. **select / WSAPoll** — *"wait and count."* Give the OS a list of sockets; it blocks until any is ready, then tells you. Simple and works everywhere; Windows `select` caps at 64 sockets.

2. **WSAAsyncSelect** — *"the window will ring me."* Bind a socket to a window; when it's ready, Windows drops a `WM_SOCKET` message in that window's queue. Needs a GUI/message loop — the old GUI-server model.

3. **WSAEventSelect** — *"the bell will ring me."* Bind a socket to a Win32 **event**; when it's ready, the event is signaled, and you wait with `WSAWaitForMultipleEvents`. No window needed; good for console apps/services; up to 64 events/thread.

4. **Overlapped I/O** — *"fire many, get told later."* Launch many `WSASend`/`WSARecv` at once with `WSAOVERLAPPED`; the kernel finishes them in the background and signals via **event or callback**. Best single-thread throughput; one thread runs many operations.

5. **I/O Completion Ports (IOCP)** — *"the OS manages the workers."* Attach sockets to a completion port; the OS runs a **thread pool** that automatically picks up completed operations. Handles **tens of thousands** of connections — the top scalability model.

**Choosing one (exam-ready):**
| Need | Choose |
|---|---|
| simple & few sockets | select / WSAEventSelect |
| GUI app | WSAAsyncSelect |
| high throughput, many connections | overlapped I/O |
| very large scale (thousands) | IOCP |

**The graduation metaphor (memorise the order):** select = asking each socket "ready?" one by one; WSAAsyncSelect/WSAEventSelect = sockets ring *you*; overlapped = you throw out lots of work and collect results late; IOCP = the OS owns the workers and hands completed work to idle threads. Each step scales further than the last.

---
**Marking scheme (7–8 marks):** each model = **1** (total 5), which-to-use = **2–3**.

---

### Q42 🟡★ Is a common Unix+Windows app possible? How? (asked: NCIT Q5b alt)

**Yes.** The socket calls themselves (`socket`, `bind`, `listen`, `connect`, `send`, `recv`) are identical on both — only the *surroundings* (headers, socket type, close function, errors, init) differ. So you write the shared logic ONCE and hide the small differences behind `#ifdef _WIN32`.

**The 5 differences to hide:**
| Thing | UNIX | Windows |
|---|---|---|
| header | `<sys/socket.h>` | `<winsock2.h>` |
| socket type | `int` | `SOCKET` |
| close | `close()` | `closesocket()` |
| errors | `errno` | `WSAGetLastError()` |
| init | none | `WSAStartup`/`WSACleanup` |

**The wrapper pattern (`#ifdef` splits the differences):**
```c
#ifdef _WIN32
  #include <winsock2.h>
  #define close_socket closesocket
#else
  #include <sys/socket.h>
  #include <netinet/in.h>
  #include <unistd.h>
  #define close_socket close
#endif

/* --- common code below — identical on both --- */
int s = socket(AF_INET, SOCK_STREAM, 0);
/* bind, listen, connect, send, recv ... */
close_socket(s);    // a name that works on both
```

**Why is this even possible (the insight):** the socket API is a *convention*, not UNIX-specific. Windows implements the very same calling convention — so the socket functions behave the same; only the OS-specific glue differs. That means ~90% of your networking code can be shared.

**Rules of thumb:**
1. Put every platform difference in `#ifdef _WIN32 ... #else ... #endif`.
2. Give `close` a common name (e.g. `close_socket`) so the shared code stays clean.
3. On Windows, errors always via `WSAGetLastError()`, never `errno`.
4. Use only what exists on BOTH (the socket API, `select`) in the shared code — Windows-only IOCP and Linux-only epoll each stay inside their own `#ifdef`.

---
**Marking scheme (5–6 marks):** yes + the 5 differences = **2**, wrapper code = **2**, key practices = **1–2**.

---

## Unit 6 — Utilities, Trends & Security

### Q43 🟡★ Name & describe network utilities. (asked: short notes — telnet, ipconfig/ifconfig, remote login, iperf, netstat)

**Plain meaning:** network utilities are **command-line tools** you run in a terminal to **test, diagnose, and see how the network is doing**. Each one answers a different practical question ("can I reach that machine?", "is my link fast enough?", "which ports are open?").

**1. `ping` — "Is that machine alive, and how fast?"** (ICMP, Layer 3)
- Sends **ICMP Echo Request** packets to a target host and waits for **Echo Reply**.
- Shows **round-trip time (latency)** in milliseconds, packet-loss percentage, and TTL.
- `ping -c 4 google.com` — send 4 pings.
- If `ping` works, you have IP connectivity to that host. If it fails, the host is unreachable or blocking ICMP.

**2. `telnet` — "Open a remote terminal / test a TCP port"** (TCP 23)
- Connects to a remote host on a specified port (default 23) and gives you a terminal shell.
- **Modern use:** TCP port testing — `telnet host 80` connects to port 80 and shows the response (great for testing HTTP, SMTP, SSH).
- **Limitation:** sends *everything* (including passwords) in **cleartext** — replaced by SSH for real remote login.

**3. `ip` / `ifconfig` — "What does my network card know?"**
- Shows/sets the IP address, netmask, MAC address, MTU, and interface status (up/down).
- `ifconfig eth0` (older Linux) or `ip addr show` (modern Linux); `ipconfig` (Windows).
- Used to answer: "is my interface even up? what IP do I have?"

**4. `iperf` — "How fast can data actually flow?"**
- **Client-server tool:** one machine runs the server (`iperf -s`), another runs the client (`iperf -c server_ip`) to measure real TCP/UDP **throughput/bandwidth** between them.
- Reports bandwidth in Mbits/sec, jitter (for UDP), and packet loss.
- Essential for testing performance: "is the link actually delivering the 100 Mbps we paid for?"

**5. `netstat` — "What's connected right now?"**
- `netstat -tlnp` — show all **TCP** ports in **listening** state (with PIDs).
- `netstat -an` — show all established connections.
- Also shows the routing table (`netstat -r`), interface statistics (`netstat -i`).
- Replaced by `ss` on modern Linux (`ss -tlnp` is faster).

**6. Remote login (`rlogin` / `ssh`):**
- `rlogin` — the old UNIX remote-terminal tool (insecure, cleartext, port 513).
- `ssh` — **Secure Shell** (port 22) — **encrypts all traffic** (including passwords), authenticates via keys or passwords. The modern replacement for `rlogin`, `telnet` and `rsh`.
- `ssh user@host` gives a secure terminal on the remote host.

**How to remember all six (grab one fact each):** ping = alive+speed; telnet = test a port / old remote shell; ifconfig/ipconfig = my own IP; iperf = measured speed; netstat = open connections; ssh = secure remote login.

---
**Marking scheme (6 marks):** each utility = **1** (6 marks).

---

### Q44 🔴★ HTTP vs WebSocket + simple server. (asked: NCIT Q7b — 8 marks)

**Plain meaning:** HTTP is like **asking questions one at a time** — you ask, the server answers, and that's it (the server can never talk to you unless you ask first). **WebSocket is a persistent phone line** — once connected, both sides can talk at any time, with almost no overhead. That's why live chat and games use WebSocket.

**HTTP (Hypertext Transfer Protocol):**
- **Request–response model:** client sends a request; server sends a response; then the connection is typically closed (or kept alive briefly).
- **Half-duplex in practice:** the client talks first, then the server replies. The server **cannot** push data unprompted.
- **Higher overhead:** each request carries full HTTP headers (cookies, User-Agent, Accept...) even for a tiny payload.
- **Stateless:** each request is independent; no built-in state between requests (use cookies/sessions for state).

**WebSocket:**
- **Full-duplex:** both client and server can send messages **at any time** — true two-way communication.
- **Persistent:** one TCP connection stays open for the whole session — no repeated connect/disconnect overhead.
- **Lower overhead:** after the handshake, frames are tiny (2–14 byte headers vs hundreds of bytes of HTTP headers).
- **Server push:** the server can send data to the client **without being asked** — essential for real-time apps.

**Comparison table (write this):**
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
2. Server replies: `HTTP/1.1 101 Switching Protocols` + `Upgrade: websocket` + `Sec-WebSocket-Accept: <hash>`.
3. After the `101` response, the connection is **upgraded to WebSocket** — both sides now exchange frames, not HTTP requests.

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

**The one-liner to remember:** HTTP = question→answer (client always first); WebSocket = upgrade the HTTP connection once, then both talk freely forever.

---
**Marking scheme (8 marks):** HTTP vs WebSocket table = **3**, handshake diagram/explanation = **2**, server pseudo-code = **2**, frame structure (bonus) = **1**.

---

### Q45 🟡★ What is gRPC? (short note)

**Plain meaning:** **gRPC** = "Remote Procedure Call, done fast". It lets your program **call a function on another machine** as if it were a local function, using **binary data** over the modern **HTTP/2** protocol. It's Google's high-performance, open-source RPC framework.

**Key features:**
- **Language-agnostic:** a Python client can call a C++ server — code is *generated* from `.proto` service definitions.
- **Four call models:** unary (one request, one response), server streaming, client streaming, **bidirectional streaming**.
- **Binary serialization (Protocol Buffers):** much faster and smaller than JSON/XML. Define services and messages in a `.proto` file; the `protoc` compiler generates client/server stubs.
- **Built on HTTP/2:** multiplexed streams, header compression, TLS by default — fast and secure.
- **Used in:** microservices, mobile↔backend, Kubernetes, Google Cloud.

**How gRPC works (brief):**
1. Define the service in a `.proto` file:
   ```protobuf
   service Greeter { rpc SayHello (HelloRequest) returns (HelloReply); }
   message HelloRequest { string name = 1; }
   message HelloReply { string message = 1; }
   ```
2. Generate code with `protoc` (generates stubs for client and server).
3. Server implements the service; client calls the generated stub → a transparent RPC over HTTP/2.

**vs REST/HTTP:** gRPC is faster (binary, HTTP/2), supports streaming, generates type-safe code; REST is simpler, human-readable, more widely supported. Use gRPC for internal microservice-to-microservice calls; REST for public APIs.

**The one-sentence memory hook:** gRPC = "write a `.proto` file once, get instant client+server stubs in any language, talking fast binary over HTTP/2."

---
**Marking scheme (5–6 marks):** what gRPC is = **1**, HTTP/2 + protobuf = **2**, four call models = **1**, proto example = **1**, vs REST = **1**.

---

### Q46 🟡★ TLS/SSL + cryptography concepts. (short note — asked)

**Plain meaning:** **TLS (Transport Layer Security)**, formerly SSL, is the "secure envelope" wrapped around normal socket data. It does **three jobs** — keep data secret (**encryption**), prove who you're talking to (**authentication**), and detect tampering (**integrity**). It sits between your application and TCP.

**The three pillars (learn each):**

1. **Encryption** — makes data unreadable to eavesdroppers:
   - **Symmetric encryption** (AES, ChaCha20): same key encrypts and decrypts. **Fast** — used for the actual data after the handshake.
   - **Asymmetric encryption** (RSA, ECC): a public+private key pair. **Slow** — used *only* to safely exchange the symmetric key during the handshake.

2. **Authentication** (certificates):
   - The server proves its identity by presenting a **digital certificate** signed by a trusted **Certificate Authority (CA)**.
   - The client verifies the CA's signature using its built-in CA certificate store.
   - Optionally the client also presents a certificate (mutual TLS) so the server can verify the client too.

3. **Integrity** (hashing / HMAC):
   - **Hash functions** (SHA-256) turn data into a fixed-size digest — used for integrity checks.
   - **HMAC** (Hash-based Message Authentication Code) = hash + secret key — proves data wasn't tampered with *and* that it came from the expected party.

**TLS handshake (simplified):**
1. Client → Server: "ClientHello" (supported cipher suites, TLS version).
2. Server → Client: "ServerHello" (chosen cipher suite) + **server certificate**.
3. Client verifies the certificate against its CA store.
4. Both agree on a **session key** (via key exchange — ECDHE for forward secrecy).
5. Both switch to **symmetric encryption** using that session key — all subsequent data is encrypted.

**Forward secrecy:** modern TLS uses **Ephemeral Diffie-Hellman (DHE/ECDHE)** — the session key is freshly generated per connection and **never stored on disk**, so even if the server's private key is later compromised, past sessions can't be decrypted.

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

**The one-liner to remember:** symmetric = fast but needs a shared key; asymmetric = slow but no shared key needed — TLS uses each for what it's best at, and certificates prove you're not talking to an impostor.

---
**Marking scheme (6 marks):** encryption = **1**, certificates/authentication = **1**, integrity/HMAC = **1**, handshake = **1**, forward secrecy = **1**, OpenSSL code = **1**.

---

### Q47 🟡★ What is SDN? Key advantages. (asked: NCIT Q6b — 8 marks)

**Plain meaning:** in a normal network, every switch/router is a **self-contained box** that both *thinks* (decides where packets go) and *does* (moves the packets). **SDN (Software-Defined Networking)** **separates the brain from the muscle**: the *thinking* moves into a central **software controller**, and the switches become simple "forwarding machines" that just follow its instructions. The network is then programmed like software, not configured switch-by-switch.

**Traditional networking vs SDN:**
- Traditional: each switch has its own control plane (routing logic). Decentralised, configured individually, one by one.
- SDN: the control plane is **extracted** from the switches and put into a **centralised SDN controller** — the "brain". The switches become simple forwarding devices that obey **flow rules** installed by the controller.

**The three layers:**
| Layer | What it does | Example |
|---|---|---|
| **Application layer** | Network apps and policies (firewall, load balancing, routing) | Custom Python app, network management UI |
| **Control layer** | Centralised **SDN controller** — makes all routing/policy decisions | OpenDaylight, ONOS, Floodlight |
| **Data/Infrastructure layer** | Switches just **forward packets** according to installed **flow rules** | OpenFlow switches |

**OpenFlow protocol** = the standard way the SDN controller talks to the switches:
- Controller installs **flow rules** (match fields + actions) into each switch's **flow table**.
- When a packet arrives, the switch checks its flow table:
  - **Match found →** do the action (forward, drop, modify).
  - **No match →** send the packet up to the controller (packet-in).
- Controller computes the path, installs flow rules on all relevant switches (packet-out).

**Key advantages of SDN (the "why" to write):**
1. **Centralised control** — one controller sees the whole network and makes globally optimal decisions (unlike distributed protocols that converge slowly).
2. **Programmability** — the network is controlled by software, not manual CLI config. Enables automation and rapid deployment.
3. **Agility / rapid innovation** — new policies deploy in seconds (write a script → push rules via the controller) instead of reconfiguring hundreds of devices.
4. **Better resource utilisation** — the controller sees all traffic patterns and can optimise paths globally.
5. **Vendor independence** — switches just speak OpenFlow; the controller doesn't care whose hardware it is.

**Memory hook:** *"traditional = every switch thinks for itself; SDN = one brain commands all switches."* Draw the three-layer diagram (app / control / data) for the exam.

---
**Marking scheme (8 marks):** what SDN is = **1**, three layers table = **3**, OpenFlow explanation = **2**, advantages list = **2**.

---

### Q48 🟡★ OpenFlow, P4, Frenetic. (short notes — asked: Gandaki "P4 and frenetic programming")

**Plain meaning:** OpenFlow, P4, and Frenetic are three different *levels* of programming an SDN network: **OpenFlow** writes the switch's decision table, **P4** rewrites what a switch can do to a packet, and **Frenetic** writes the controller's policies (and turns them into OpenFlow for you).

**OpenFlow — the controller ↔ switch protocol:**
- The **standard protocol** by which the SDN controller programs the **flow tables** of switches.
- The controller installs rules: "if a packet matches [src IP, dst IP, port...], then [forward to port X / drop / modify / send to controller]."
- Switches are simple: they look up packets in their flow table and follow the rules — no routing logic in the switch.
- Used in data centres, research networks, and SDN deployments.

**P4 — programming the data plane:**
- **P4 (Programming Protocol-independent Packet Processors)** is a high-level **domain-specific language (DSL)** for programming **what the switch does** — not just forwarding, but custom packet parsing, processing, and modification.
- OpenFlow can only match a fixed set of fields (IP, MAC, port). P4 lets you define your **own headers and processing logic** (process custom protocols, implement new load-balancing schemes).
- You write a P4 program → compile it → deploy to a P4-programmable switch → the switch processes packets per your custom logic.
- Key use: **custom data-plane logic** that OpenFlow cannot express.

**Frenetic — programming the controller:**
- **Frenetic** is a high-level **DSL for programming the SDN controller**. You write network policies in Frenetic (a functional language), and the compiler turns them into **OpenFlow rules** for the switches.
- Advantage: you write high-level policies ("forward HTTP to the load balancer, drop all other traffic") without manually computing which OpenFlow rules go on which switch.
- Frenetic handles the hard parts: composing policies, computing flow tables across switches, handling packet-in events.
- Part of the broader **"network programming languages"** research area (with Pyretic, NetKAT, etc.).

**Summary table:**
| Tool | Programs what? | Level |
|---|---|---|
| **OpenFlow** | Switch flow tables (match + action) | Low-level rules |
| **P4** | Switch packet processing logic | Data-plane DSL |
| **Frenetic** | Controller policies (compiles to OpenFlow) | Controller DSL |

**Memory hook:** the three tools stack: *Frenetic (policy) → produces → OpenFlow rules → installed in switches, and P4 (re)programs what the switch hardware can do to packets.*

---
**Marking scheme (5–6 marks):** OpenFlow = **2**, P4 = **2**, Frenetic = **1**, comparison table = **1**.

---

### Q49 🟡★ WebSockets short note. (asked: Gandaki short note)

**Plain meaning:** **WebSocket** is a protocol for **persistent, two-way (full-duplex) communication** over a single TCP connection — the "hotline" that lets both a web page and its server talk freely at any moment, unlike HTTP's one-question-one-answer style.

**Key characteristics:**
- **Full-duplex:** both client and server can send messages at any time, even simultaneously.
- **Persistent:** one TCP connection stays open for the session — no repeated connect/disconnect.
- **Low overhead:** frame headers are 2–14 bytes (vs hundreds of bytes of HTTP headers per request).
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

**Use cases:** chat apps, multiplayer games, live dashboards, stock tickers, IoT real-time push, collaborative editing (Google Docs), live sports scores.

**vs HTTP polling:** WebSocket is far more efficient — no repeated HTTP headers, no polling overhead, true real-time updates pushed from the server.

---
**Marking scheme (5–6 marks):** what it is = **1**, key characteristics = **2**, handshake = **1**, frames = **1**, use cases = **1**.

---

### Q50 🟢 Daemonizing techniques in Unix.

**Plain meaning:** to turn a normal program into a **daemon** (a background process with no terminal), you run a fixed set of steps that *detach* it: give it its own session, move it somewhere safe, and cut off its input/output.

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

**Step-by-step explanation (understand each):**
1. **`fork()`** — create a child; the parent exits so your shell prompt returns immediately.
2. **`setsid()`** — create a **new session** (new process group, no controlling terminal). The child is now a session leader.
3. **Second `fork()` (double-fork)** — after `setsid()` the process is a session leader, and a session leader *could* re-acquire a controlling terminal if it opens a terminal device. By forking again (and the parent exiting), the grandchild is **not** a session leader — it can **never** get a controlling terminal.
4. **`chdir("/")`** — avoid holding a busy/mounted filesystem as the current directory (could prevent unmounting).
5. **`umask(0)`** — clear the file-creation mask so the daemon can create files with full intended permissions.
6. **Redirect stdin/stdout/stderr to `/dev/null`** — with no terminal, reading stdin would fail and writing to stdout/stderr would be meaningless. Sending them to `/dev/null` makes both harmless (output discarded, reads return EOF).

**Result:** a fully detached background process — no controlling terminal, no parent (orphaned and adopted by `init`/`systemd`), writing only to logs (via `syslog`).

**Optional extras:** change the process name (`prctl(PR_SET_NAME, "mydaemon")`), write a PID file (`/var/run/mydaemon.pid`), set up signal handlers for graceful shutdown.

**Memory hook:** *"fork, setsid, fork again, chdir, umask, /dev/null"* — say it in that order and you have the recipe.

---
**Marking scheme (5 marks):** code/6 steps = **3** (0.5 each), double-fork explanation = **1**, redirect to /dev/null = **1**.

---

## Full 6-Part 8-Mark Model Answers

> Every answer below follows the structure **①Definition → ②Diagram → ③Full concept → ④Example/code → ⑤Common errors/limits → ⑥Conclusion**. Practise writing each one by hand with a pencil. The diagrams are worth the most — an examiner who sees a clear diagram knows you understand immediately. Read the **①Definition (plain)** line first — it gives you the one-sentence picture in everyday words, so you always know what the answer is really about before you dig into the details.

### M1. Compare TCP, UDP and SCTP  [🔴★ NCIT 2025 Q1a]

**① Definition** — **Plain meaning:** the transport layer is a *delivery company* that carries your data between programs on different machines. TCP, UDP and SCTP are three delivery options with different guarantees: **TCP** = registered/tracked courier (everything must arrive, in order, at the cost of speed), **UDP** = mailbox letter (no tracking, no order, instant), **SCTP** = tracked courier with *two vehicles and multiple addresses* so a phone call never drops. All three sit **between your application and IP**: they break your data into pieces, add a port (so it reaches the right app), and hand the result to IP.

**② Diagram**
```
 APPLICATION (HTTP, FTP, DNS, VoIP ...)
        │
        ▼
 ┌─────────────┬──────────────┬──────────────┐
 │     TCP     │     UDP      │     SCTP     │
 │ connection  │ connection-  │ connection-  │
 │ oriented,   │ less, no ack,│ oriented, ack│
 │ ack+re-xmit │ no order     │ + msg order, │
 │ stream      │ datagram     │ multi-homed  │
 └─────────────┴──────────────┴──────────────┘
        └────────────┬──────────────┘
                     ▼
                    IP (network layer)
```

**③ Full concept**
- **TCP (Transmission Control Protocol)** — reliable, ordered, connection-oriented **byte stream**. It does handshake (3-way), ACKs every segment, retransmits lost data, and applies flow + congestion control. Because it preserves only a *stream of bytes* (no message boundaries), the receiver may get your data in different-sized pieces. Used for: web (HTTP), email (SMTP/IMAP), file transfer (FTP), remote shell (SSH).
- **UDP (User Datagram Protocol)** — connectionless, best-effort **datagrams** (each message stays whole). No ACK, no retransmit, no ordering, no handshake — so it is fast and has tiny headers. Used for: DNS (one quick query), live video/audio, real-time games, VoIP, streaming, SNMP.
- **SCTP (Stream Control Transmission Protocol)** — reliable like TCP, **but keeps message boundaries** (so each "message" arrives whole, in order) **and supports multi-homing** (one connection can span multiple IP addresses). Because the handshake uses a **4-way INIT/INIT-ACK/COOKIE-ECHO/COOKIE-ACK** sequence with a cookie, it is also harder to SYN-flood. Used for: telephone signalling (SIGTRAN), IP telephony — anything where a dropped call is unacceptable.

**④ Example — where each is used in real life:**
```
TCP:   ssh user@host      (must be 100% correct)
UDP:   a live video call  (a lost frame is better than a delay)
SCTP:  phone network signalling (SIGTRAN) — call must not drop if one link dies
```

**⑤ Common errors/limits**
- Confusing "unreliable" UDP with "broken" — UDP is the *right* tool for low-latency one-shot data (DNS). Reliability isn't free.
- Saying SCTP is "just TCP" — the key extra features are **message boundaries** and **multi-homing**, both absent in TCP.
- Forgetting that TCP is a *byte stream*: one `send` may arrive as several `recv`s. That is by design, not a bug.

**⑥ Conclusion** — Choose the delivery guarantee the data needs: **must-not-lose → TCP (or SCTP); must-be-fast-and-loss-tolerant → UDP; carrier-grade telephony → SCTP**. The table is most of the marks.

---

### M2. TCP Three-Way Handshake + Why ISN should not start from 0  [🔴★ multiple papers]

**① Definition** — **Plain meaning:** before sending data, TCP performs an *introduction ceremony* of exactly **three packets** so both sides can prove "I'm here and ready" and agree on their **starting sequence numbers**. If every connection started at 0, an old lost packet could be mistaken for fresh data in a new connection — so the start number (ISN) is made **random and unpredictable**.

**② Diagram**
```
CLIENT                                      SERVER
   │  1. SYN  (seq = x)                        │
   │ ───────────────────────────────────────▶  │   "I want to connect; my start is x"
   │  2. SYN+ACK (seq = y, ack = x+1)          │
   │ ◀───────────────────────────────────────  │   "OK; I got x, my start is y"
   │  3. ACK (ack = y+1)                       │
   │ ───────────────────────────────────────▶  │   "I got y"
   ▼                                           ▼
        CONNECTION ESTABLISHED — data flows
```

**③ Full concept**
- **Step 1 (active open):** client sends a segment with the **SYN** flag and its initial sequence number `x`.
- **Step 2 (passive open):** server replies with one packet holding **both** SYN and ACK: `ack = x+1` says "I received your x", `seq = y` says "my numbering starts at y".
- **Step 3:** client ACKs with `ack = y+1`. Now **both sides** know the other is alive and the sequence numbers agree.
- **Why not two packets:** with only two packets, the server could never be sure the client received its reply (the "two-army problem"). The third ACK removes the doubt. Step 2 combining SYN+ACK is why it is exactly three packets, not four.

**Why the ISN should NOT start from 0 (three reasons):**
1. **Stale-packet collision:** an old delayed segment from a previous connection could carry a sequence number that *matches* the new connection — the receiver would accept ancient garbage as valid new data.
2. **Security:** a predictable ISN (0) lets an attacker **guess sequence numbers and inject fake data** (sequence-number prediction attack).
3. **Fix:** random, unpredictable ISN **+ TIME_WAIT** (2×MSL) so old duplicates die before numbers can be reused; the classic ISN clock advances ~every 4 µs.

**④ Example/code**
```c
// What the two endpoints agree on during the handshake:
// client ISN = x   (random)
// server ISN = y   (random)
// first data byte from client is numbered x+1
// first data byte from server is numbered y+1
```

**⑤ Common errors/limits**
- Drawing SYN+ACK as two separate packets (wrong — it is one packet in step 2).
- Saying the ISN must be 0 to "simplify" — this is exactly the insecure/stale-matching behaviour the protocol is designed to avoid.
- Confusing the handshake with the connection **termination** (which is a 4-packet FIN/ACK dance — see M-series/Q8).

**⑥ Conclusion** — The three-way handshake establishes reachability, readiness, and agreed sequence numbers in exactly three packets; the random ISN (plus TIME_WAIT) protects against stale-packet confusion and sequence-number guessing.

---

### M3. TCP State-Transition Diagram (all 11 states)  [🔴★ NCIT 2025, Gandaki]

**① Definition** — **Plain meaning:** a TCP connection behaves like a tiny *state machine* — it is always in one of **11 states**, and each event (send SYN, receive FIN, timeout) pushes it to another state. Understanding who is in which state and what moves it out is the deepest TCP question in the syllabus. The **active** participant (the one who contacts) and the **passive** participant (the one who listens) follow different paths.

**② Diagram**
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
```

**③ Full concept — the four journeys to memorise:**
- **Server connection path:** `CLOSED → LISTEN → SYN_RCVD → ESTABLISHED`
- **Client connection path:** `CLOSED → SYN_SENT → ESTABLISHED`
- **Active close (the side that sends FIN first):** `ESTABLISHED → FIN_WAIT_1 → FIN_WAIT_2 → TIME_WAIT → CLOSED`
- **Passive close (the side that receives FIN first):** `ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED`
- (Rare `CLOSING` state: both sides send FIN almost simultaneously and both wait for the other's ACK.)

**The 11 states (one line each):**
- `CLOSED` — no connection at all (start/finish).
- `LISTEN` — **passive** side, waiting for a SYN.
- `SYN_SENT` — **active** side, sent SYN, awaiting SYN+ACK.
- `SYN_RCVD` — **passive** side, got SYN, sent SYN+ACK, awaiting the ACK.
- `ESTABLISHED` — data can flow.
- `FIN_WAIT_1` — active closer sent FIN, awaiting ACK.
- `FIN_WAIT_2` — active closer got the ACK of its FIN, awaiting the peer's FIN.
- `CLOSE_WAIT` — passive side got the peer's FIN and is deciding whether to close (app decides).
- `LAST_ACK` — passive side sent its FIN, awaiting the final ACK.
- `TIME_WAIT` — active closer after receiving the peer's FIN; waits **2×MSL** before final CLOSED.
- `CLOSING` — both sent FIN, both await ACK (simultaneous close).

**④ Example — "which event moves which state" (quick revision):**
```
LISTEN    --recv SYN--> SYN_RCVD
SYN_SENT  --recv SYN+ACK--> ESTABLISHED
ESTABLISHED --send FIN--> FIN_WAIT_1
ESTABLISHED --recv FIN--> CLOSE_WAIT
FIN_WAIT_2 --recv FIN--> TIME_WAIT
CLOSE_WAIT --send FIN--> LAST_ACK
LAST_ACK   --recv ACK--> CLOSED
TIME_WAIT  --2*MSL--> CLOSED
```

**⑤ Common errors/limits**
- Forgetting the passive server also has `SYN_RCVD` — it is not just the client that goes through intermediate states.
- Drawing TIME_WAIT on the wrong side: it is the side that **sent the first FIN** (the active closer).
- Mixing up CLOSE_WAIT (got FIN, passive side) with FIN_WAIT_1 (sent FIN, active side). Hint: the word "WAIT" tells you what it's waiting *for*.

**⑥ Conclusion** — Draw the full diagram, then annotate the four journeys (server, client, active close, passive close) with one sentence each; that combination earns the full marks.

---

### M4. Value-Result Arguments  [🔴★ NCIT 2025 Q3a, Gandaki Q2b]

**① Definition** — **Plain meaning:** a **value-result argument** is one variable used in *both* directions: your program writes a **value in** ("my buffer is this big"), the kernel does its work and **writes a new value back** ("I actually used this much"). Filling a form with your capacity and getting it back stamped with the real number is the picture. Used whenever the **kernel produces** the address (accept, recvfrom, getsockname, getpeername).

**② Diagram**
```
Process ── (value in: len = sizeof(buf)) ──▶ Kernel  "here is how big my buffer is"
Process ◀── (result out: len = real size) ── Kernel  "here is how much I actually used"

vs (by-value, one-way):
Process ── (bind/connect: address_len) ──▶ Kernel   "here is how big the address is"
            (no write-back — the kernel only reads it)
```

**③ Full concept**
- **Two directions of length passing:**
  - **Process → kernel** (`bind`, `connect`, `sendto`): you *supply* the address → length passed **by value** (`socklen_t`), read-only.
  - **Kernel → process** (`accept`, `recvfrom`, `getsockname`, `getpeername`): the kernel *produces* the address → length passed **by pointer** (`socklen_t *`) — input value = your buffer size (prevents overflow), output value = actual size stored.
- **Why needed:**
  1. **Safety:** the kernel must know your buffer size or it could write past it (overflow).
  2. **Variable sizes:** IPv4 = 16 B, IPv6 = 28 B. Reporting the real size lets you tell *which family* actually connected.

**④ Example/code**
```c
struct sockaddr_in cli;
socklen_t len = sizeof(cli);            /* VALUE: "my buffer is this big" */
int cfd = accept(listenfd, (SA*)&cli, &len);  /* kernel writes real size → RESULT */
printf("peer address used %u bytes\n", (unsigned)len);  /* len updated by kernel */
```

**⑤ Common errors/limits**
- Passing `&len` **uninitialised** — the kernel reads garbage → overflow/wrong data. Always set it first.
- Using by-value style on a by-reference function (and vice versa) — compile warnings or silent memory errors.
- Forgetting the cast to `(struct sockaddr *)` on the address, which the functions require.

**⑥ Conclusion** — Value-result = one variable doing input and output duty; it exists for safety (no overflow) and to report variable address sizes. Pass by **value** when you supply the address, by **pointer** when the kernel supplies it.

---

### M5. Socket Address Structures  [🔴★ NCIT Q2b, Gandaki Q2a/Q2b]

**① Definition** — **Plain meaning:** a socket address structure is a small C struct holding "who to talk to" — the **family** (IPv4 / IPv6 / Unix), the **port**, and the **address**. You fill it and hand it to every socket call; behind the scenes all calls see it through one **generic pointer type** (`struct sockaddr *`).

**② Diagram**
```
 struct sockaddr_in (IPv4, 16 bytes)
 ┌──────────┬───────┬──────────────┬──────────────┐
 │ sa_family│ port  │  addr.s_addr │  sin_zero[8] │
 │ (AF_INET)│(htons)│  (htonl/pton)│  zero pad    │
 └──────────┴───────┴──────────────┴──────────────┘
struct sockaddr_storage (generic, >= 128 bytes) — big enough & aligned for ANY family
```

**③ Full concept**
- **`struct sockaddr`** — generic (16 B): `sa_family` + `sa_data[14]`. All socket functions are declared with `(struct sockaddr *)` regardless of actual family.
- **`struct sockaddr_in`** (IPv4, 16 B): `sin_family` (AF_INET), `sin_port` (network byte order → `htons`), `sin_addr.s_addr` (network byte order → `htonl`/`inet_pton`), `sin_zero[8]` (padding, must be zeroed).
- **`struct sockaddr_in6`** (IPv6, 28 B): `sin6_*` fields including `sin6_addr` (128-bit) and `sin6_scope_id`.
- **`struct sockaddr_storage`** (≥128 B): large enough for the biggest family type, with the strictest alignment — declare one, pass it to `accept`/`recvfrom`, then inspect `ss_family` to learn which family arrived. This is how you write **family-neutral** IPv4/IPv6 servers.
- **The casting trick:** every call takes a generic pointer; you *declare* a concrete struct and *cast* it: `bind(s, (struct sockaddr *)&serv, sizeof(serv))`. The kernel reads `sa_family` inside to know which real struct it points to.
- **Naming hint:** generic = `sa_*`, IPv4 = `sin_*`, IPv6 = `sin6_*`, storage = `ss_family`.

**④ Example/code**
```c
struct sockaddr_in serv;                          /* concrete IPv4 struct */
bzero(&serv, sizeof(serv));                       /* zero the padding too */
serv.sin_family      = AF_INET;
serv.sin_port        = htons(8080);
serv.sin_addr.s_addr = htonl(INADDR_ANY);
bind(sockfd, (struct sockaddr *)&serv, sizeof(serv));  /* cast to generic */

struct sockaddr_storage cli;                      /* family-neutral buffer */
socklen_t clen = sizeof(cli);
int cfd = accept(sockfd, (struct sockaddr *)&cli, &clen);
if (((struct sockaddr*)&cli)->sa_family == AF_INET6) { /* IPv6 arrived */ }
```

**⑤ Common errors/limits**
- Forgetting `htons`/`htonl` — port/addr must be network byte order.
- Not zeroing `sin_zero` (uninitialised bytes leak into kernel-visible data).
- Declaring `sockaddr_in` but passing it without the cast (compile error/warning) or assuming `sockaddr` is "the struct to fill" (it is the generic pointer, not the IPv4 one).

**⑥ Conclusion** — Know the 4 structs' sizes/fields, always convert byte order, and remember the cast-to-generic rule. `sockaddr_storage` is the modern answer for writing IPv4/IPv6-agnostic code.

---

### M6. The Five I/O Models  [🔴★ Gandaki Q4a]

**① Definition** — **Plain meaning:** reading from a socket needs two phases — (1) **wait** for data to arrive, (2) **copy** it into your buffer. The five I/O models differ only in *what the process does while waiting*: sleep (blocking), keep asking (non-blocking), sleep-but-watch-many (multiplexing), get interrupted (signal-driven), or delegate everything to the kernel (asynchronous). Only model 5 is truly asynchronous; models 1–4 are all **synchronous** because the actual `recvfrom` still blocks.

**② Diagram**
```
         Phase 1              Phase 2
         ─────────            ─────────
Model 1: [BLOCK: sleep]──────[BLOCK: kernel copies] ──▶ return
Model 2: [poll: EWOULDBLOCK]─[BLOCK: kernel copies] ──▶ return
Model 3: [BLOCK: select()]───[BLOCK: kernel copies] ──▶ return
Model 4: [SIGIO handler runs][BLOCK: kernel copies] ──▶ return
Model 5: [Kernel does everything]───────────────return  (never blocks user)
```

**③ Full concept**
1. **Blocking I/O** — `recvfrom` sleeps until data arrives *and* is copied. Simplest, but one thread per connection.
2. **Non-blocking I/O** — returns `EWOULDBLOCK` if no data; the app polls. Alone = busy-wait (wastes CPU); useful only with `select`.
3. **I/O multiplexing (`select`/`poll`)** — block once in `select` watching *many* fds; receive on each ready fd. Standard scalable server model.
4. **Signal-driven I/O (`SIGIO`)** — kernel sends `SIGIO` when an fd is ready; the handler runs `recvfrom`. Never blocks the main loop, but signals can merge/be missed.
5. **Asynchronous I/O (`aio_read`)** — the kernel waits *and* copies, then notifies you. The process never blocks at all.

**Sync vs async (the exam trick):** models 1–4 are **synchronous** — in all of them `recvfrom` blocks until the copy is done (the process is active, "blocked" in the call). Only model 5 is asynchronous: the *entire* operation (wait + copy) happens without the process waiting.

**Which for real servers:** model 3 (select/poll/epoll) — portable, mature, one process handles thousands of sockets. Model 5 exists but is rarely the practical winner.

**④ Example/code**
```c
// Model 1 blocking:
recvfrom(fd, buf, n, 0, ...);          // sleeps until data copied

// Model 3 multiplexing:
select(maxfd+1, &readfds, NULL, NULL, NULL);   // blocks watching all fds
for (fd = 0; fd <= maxfd; fd++)
    if (FD_ISSET(fd, &readfds)) recv(fd, buf, n);  // immediate (data ready)

// Model 5 async:
aio_read(&iocb);                        // kernel waits + copies, notifies later
```

**⑤ Common errors/limits**
- Calling non-blocking "asynchronous" — it is synchronous (still blocks during the copy).
- Drawing model 5 with a user-side loop — model 5 has *no* user-side polling at all.
- Saying select is async — it blocks the process (that's the point of its efficiency).

**⑥ Conclusion** — The diagram plus a one-line "what the process does while waiting" per model scores the most. Then the clean sentence: *models 1–4 synchronous, model 5 asynchronous.*

---

### M7. Mechanisms to Handle Multiple Clients (fork / select / threads)  [🔴★ Gandaki Q3b, NCIT Q5a]

**① Definition** — **Plain meaning:** one server program must talk to *many clients at once*. Three standard tools exist: give **each client its own copy of the process** (`fork`), watch **all sockets in one loop** (`select`), or give **each client its own thread** (`pthread_create`). Each is a different trade-off between simplicity, memory, and scalability.

**② Diagram**
```
 Clients:  c1  c2  c3  c4   c5 ...
                │
       ┌────────┼─────────────────────────────┐
       ▼        ▼                             ▼
   fork()   select()/poll()              pthread_create()
   = one PROCESS  = ONE process watches   = one THREAD per
     per client      all sockets in a     client, sharing
     (isolated)      loop (low memory)    memory (risky)
```

**③ Full concept**
- **Approach 1 — `fork()` (process-per-connection):** after `accept`, fork a child that handles that client; the parent returns to `accept`. Simple and isolated (child crash doesn't kill the server), but heavy (full process copy per client) and needs zombie reaping.
- **Approach 2 — `select()`/`poll` (single-threaded event loop):** one process watches all sockets; when any is ready, it reads. Very scalable (thousands of idle connections), low memory, but complex and one slow handler blocks everything → combine with non-blocking reads.
- **Approach 3 — threads (`pthread_create`):** like fork but light — threads share the address space. Great concurrency with shared state, but **race conditions** need mutexes; one crashing thread can kill the whole process.

**④ Example/code (fork pattern)**
```c
for (;;) {
    connfd = accept(listenfd, (SA*)&cliaddr, &clilen);
    if ((pid = fork()) == 0) {        /* child services this client */
        close(listenfd);
        doit(connfd);
        close(connfd);
        exit(0);
    }
    close(connfd);                    /* parent keeps listening */
}
```

**⑤ Common errors/limits**
- fork: forgetting to close `connfd` in the parent and `listenfd` in the child (file-descriptor leaks).
- fork: not reaping zombies (`waitpid` in a `SIGCHLD` handler).
- select: not rebuilding `fd_set` every loop (select overwrites it).
- threads: accessing shared state without a mutex → race conditions.

**⑥ Conclusion** — fork = simple/isolated but heavy; select = scalable/low-memory but complex; threads = light but need locks. Choosing one depends on client count and whether processes must share state.

---

### M8. Socket Options (SO_REUSEADDR, SO_BROADCAST, SO_KEEPALIVE, SO_LINGER)  [🔴★ NCIT Q5b, Gandaki Q4b]

**① Definition** — **Plain meaning:** a socket is a kernel object with configurable **switches**. `setsockopt`/`getsockopt` flip or read those switches. Four options matter for the exam: reuse a TIME_WAIT port (SO_REUSEADDR), allow broadcasting (SO_BROADCAST), detect dead peers (SO_KEEPALIVE), and control what `close()` does with unsent data (SO_LINGER).

**② Diagram**
```
 ┌─────────────┐  setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on))  ┌────────┐
 │ application │ ──────────────────────────────────────────────────────────▶ │ kernel │
 │             │ ◀───────────────────────────────────────────────────────── │ socket │
 └─────────────┘  getsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &val, &len)       └────────┘
```

**③ Full concept**
- **`SO_REUSEADDR`** — lets you bind a port that is in **TIME_WAIT** (2×MSL). Without it, a server that restarts fails with `EADDRINUSE`. Set *before* `bind()`; essential for restarting servers.
- **`SO_BROADCAST`** — allows UDP sockets to send to broadcast addresses; refused by default (`EACCES`). Routers don't forward broadcast (local subnet only).
- **`SO_KEEPALIVE`** — TCP sends keepalive probes after 2 hours idle. ACK → alive; RST → peer crashed (`ECONNRESET`); no response → 8 probes × 75 s → `ETIMEDOUT`. Detects dead peers on long-lived connections.
- **`SO_LINGER`** — controls `close()`:
  ```
  struct linger { int l_onoff; int l_linger; };
  l_onoff=0                → close() returns immediately (default; data sent in background)
  l_onoff≠0, l_linger=0    → TCP ABORTS: discard buffer, send RST, no FIN
  l_onoff≠0, l_linger≠0    → close() BLOCKS until data ACKed or timeout
  ```

**④ Example/code**
```c
int on = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));  // before bind
setsockopt(udpfd,   SOL_SOCKET, SO_BROADCAST,  &on, sizeof(on));  // before sendto
setsockopt(connfd,  SOL_SOCKET, SO_KEEPALIVE,  &on, sizeof(on));

struct linger li = {1, 5};    /* block up to 5 s waiting for ACK */
setsockopt(connfd,  SOL_SOCKET, SO_LINGER, &li, sizeof(li));
struct linger li2 = {1, 0};   /* abort now: RST + discard unsent data */
setsockopt(connfd,  SOL_SOCKET, SO_LINGER, &li2, sizeof(li2));
```

**⑤ Common errors/limits**
- `SO_REUSEADDR` semantics differ across Linux/BSD/Windows; Linux may also need `SO_REUSEPORT`.
- `SO_KEEPALIVE`'s 2-hour default is too slow for most apps — tune `TCP_KEEPIDLE` or use app-level heartbeats.
- `SO_LINGER` with a timeout **blocks** `close()` — dangerous if the peer is slow; can stall a server thread.
- Setting buffer options after `connect`/`listen` has no effect.

**⑥ Conclusion** — These four options fix real server problems (restart, broadcast, dead-peer detection, graceful vs abrupt close). The `struct linger` table is a memorisation favourite.

---

### M9. How is Winsock different from UNIX sockets? + static vs dynamic linking  [🔴★ NCIT Q6a]

**① Definition** — **Plain meaning:** Winsock is the *same socket idea rewritten for Windows*. The calls are alike, but Windows needs a **SOCKET handle instead of an int**, **`closesocket` instead of `close`**, **`WSAGetLastError()` instead of `errno`**, and — the biggest difference — **`WSAStartup()`/`WSACleanup()` before/after**, because the network code lives in a DLL (`ws2_32.dll`) that must be loaded first.

**② Diagram**
```
 UNIX                              WINDOWS (Winsock)
 socket()                          WSAStartup()  ← load DBLL first
 connect()/bind()/listen()         socket()/connect()/...
 read()/write()/send()/recv()      send()/recv() only
 close(fd)                         closesocket(s)
 errno                             WSAGetLastError()
 (nothing to unload)               WSACleanup()

 library: inside kernel            library: ws2_32.dll (loaded at run time)
```

**③ Full concept**
- UNIX socket functions need **no setup** — the code is always in the kernel. Windows keeps networking in a **DLL** you must load and version-check first — that is `WSAStartup`. That's why Windows has a "step 0" that UNIX doesn't.
- **Static vs dynamic linking:**
  - **Dynamic (DLL):** library code in a separate file, loaded at run time. Small executables, easy updates, shared across apps. Fails if the DLL is missing/wrong version ("DLL hell").
  - **Static:** library code copied into the `.exe`. Bigger, always runs, but updating means recompiling everyone.

**④ Example/code**
```c
#include <winsock2.h>
#pragma comment(lib, "ws2_32.lib")
WSADATA wd;
WSAStartup(MAKEWORD(2,2), &wd);           /* STEP 0: UNIX has no equal */
SOCKET s = socket(AF_INET, SOCK_STREAM, 0);
/* ... send/recv ... */
closesocket(s);                            /* not close() */
WSACleanup();                              /* unload the DLL */
```

**⑤ Common errors/limits**
- Calling a Winsock function before `WSAStartup` → every call fails with `WSANOTINITIALISED`.
- Using `close()` instead of `closesocket()` → won't compile on Windows.
- Forgetting the `winsock2.h` header vs `windows.h` (which pulls in the old 1.1 API) — use `<winsock2.h>`.

**⑥ Conclusion** — Same socket concepts, different glue: SOCKET/closesocket/WSAGetLastError/WSAStartup. The DLL setup step is why Windows has extra init/cleanup the UNIX API lacks. Dynamic linking = smaller + updatable; static = self-contained.

---

### M10. Winsock TCP and UDP Client-Server Sequences with Code  [🔴★ Gandaki Q5b]

**① Definition** — **Plain meaning:** this is the exact call *recipe* a Winsock program follows. TCP = "call, wait in line, pick up caller, talk on a new line"; UDP = "no line, no pickup — just claim an address and take datagrams". Memorise the order, then each call's one-line job.

**② Diagram**
```
 TCP SERVER: WSAStartup → socket → bind → listen → accept → recv/send → closesocket → WSACleanup
 TCP CLIENT: WSAStartup → socket → connect → send/recv → closesocket → WSACleanup
 UDP SERVER: WSAStartup → socket → bind → recvfrom → closesocket → WSACleanup     (no listen/accept)
 UDP CLIENT: WSAStartup → socket → sendto → closesocket → WSACleanup              (no bind/connect)
```

**③ Full concept**
- **TCP server:** `listen` marks the socket "ready, queue callers"; `accept` picks up the *next completed* connection and returns a **new socket** for talking to that client (the original listener keeps listening). Data flows on the new socket via `recv`/`send`.
- **TCP client:** the kernel auto-assigns the source port (ephemeral) — no `bind` needed. `connect` performs the 3-way handshake.
- **UDP:** no connection at all — server only `bind`s, then `recvfrom` each datagram; client `sendto` names the destination *on every datagram*.
- Both start with `WSAStartup` (load DLL) and end with `WSACleanup` (unload it).

**④ Example/code (TCP server)**
```c
WSADATA w; WSAStartup(MAKEWORD(2,2), &w);
SOCKET s = socket(AF_INET, SOCK_STREAM, 0);
SOCKADDR_IN sa; sa.sin_family = AF_INET; sa.sin_port = htons(5150);
sa.sin_addr.s_addr = htonl(INADDR_ANY);
bind(s, (SOCKADDR*)&sa, sizeof(sa));
listen(s, 5);
SOCKADDR_IN cli; int clen = sizeof(cli);
SOCKET cs = accept(s, (SOCKADDR*)&cli, &clen);   /* blocks until a client calls */
char buf[1024]; int n = recv(cs, buf, sizeof(buf), 0);
send(cs, buf, n, 0);
closesocket(cs); closesocket(s);
WSACleanup();
```

**⑤ Common errors/limits**
- Forgetting `WSAStartup` → `WSANOTINITIALISED`.
- Using `INADDR_ANY` without `htonl` (though `INADDR_ANY` is already 0, always convert for correctness).
- On the server, forgetting that `accept` returns a NEW socket — talk to the client on `cs`, not `s`.

**⑥ Conclusion** — TCP: service has bind→listen→accept then talk on the returned socket; UDP: bind→recvfrom only. Wrap both in WSAStartup/WSACleanup and they work.

---

### M11. Overlapped I/O in Winsock  [🔴★ NCIT Q7a]

**① Definition** — **Plain meaning:** blocking I/O = *do one op, wait, do the next*; **overlapped I/O = fire off many socket operations at once** and collect the results later, with the kernel doing the work in the background. Your thread is never stuck — that's what makes it "asynchronous" from the program's point of view.

**② Diagram**
```
 BLOCKING:   send1 ─wait─▶ send2 ─wait─▶ send3 ─wait─▶ ... (thread idle while waiting)
 OVERLAPPED: issue send1 ─┐
             issue send2 ─┤  (thread moves on to other work)
             issue send3 ─┘
             ... later, each op signals: "done!" → WSAGetOverlappedResult()
```

**③ Full concept**
1. Create the socket as overlapped with `WSASocket(..., WSA_FLAG_OVERLAPPED)` — without this flag, overlapped calls fail.
2. Launch operations with `WSASend`, `WSARecv`, `WSARecvFrom`, `WSAIoctl`, `AcceptEx`, each passing a **`WSAOVERLAPPED`** struct (a "job box": Win32 event + status).
3. Each call either finishes instantly (TRUE) or returns `SOCKET_ERROR` with **`WSA_IO_PENDING`** — *not a failure*, it means "job queued".
4. Completion is reported by an **event object** (check with `WaitFor*`) or a **completion routine** (callback).
5. `WSAGetOverlappedResult()` returns how many bytes actually moved.

**Why async:** the thread issues many ops then does useful work — one thread can manage hundreds of outstanding operations, giving the best single-thread throughput of all Winsock I/O models. Attach to an **IOCP** and the OS runs a thread pool that hands completed ops to idle threads automatically.

**④ Example/code**
```c
SOCKET s = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
WSAOVERLAPPED ov = {0};
char buf[1024]; WSABUF wb = { sizeof(buf), buf };
DWORD flags = 0, bytes = 0;
int rc = WSARecv(s, &wb, 1, &bytes, &flags, &ov, NULL);
if (rc == SOCKET_ERROR && WSAGetLastError() == WSA_IO_PENDING) {
    /* job queued — do other work; completion will be reported */
}
```

**⑤ Common errors/limits**
- Forgetting `WSA_FLAG_OVERLAPPED` on the socket.
- Treating `WSA_IO_PENDING` as a failure — it is the normal "in progress" result.
- Not keeping the buffer valid until the operation completes (the kernel writes into it *later*).
- Buffers must remain untouched until completion — reuse too early corrupts data.

**⑥ Conclusion** — The power of overlapped I/O is *concurrency without threads*: issue many operations, get told late, collect with `WSAGetOverlappedResult`. Top-scale servers pair it with IOCP.

---

### M12. HTTP vs WebSocket + Simple Server  [🔴★ NCIT Q7b]

**① Definition** — **Plain meaning:** HTTP is *one-question-one-answer* (the client always asks first; the server can never speak unless asked). WebSocket is a *persistent hotline* — after an initial HTTP upgrade, both sides talk freely at any time with tiny frames. Real-time apps (chat, games, dashboards) use WebSocket; documents/APIs use HTTP.

**② Diagram**
```
 HTTP (request/response):   C ─request─▶ S   C ◀─response─ S   (connection may close)
 WebSocket (persistent):    C ───Upgrade──▶ 101 ──────────────► both talk freely, anytime
```

**③ Full concept**
- **HTTP:** stateless, half-duplex in practice (client first, then server), large repeated headers, closed after a response unless keep-alive. Server cannot push.
- **WebSocket:** full-duplex, one persistent TCP connection (no re-dialing), 2–14 byte frame headers, server can push anytime. Started via an HTTP **Upgrade** request.
- **Handshake:** client sends `GET ... Upgrade: websocket` + `Sec-WebSocket-Key`; server answers `101 Switching Protocols` + `Sec-WebSocket-Accept` (hash of key + GUID). After that, frames flow.
- **Frame:** FIN (1 bit) + opcode (0x1 text, 0x2 binary, 0x8 close, 0x9 ping, 0xA pong) + MASK (1 bit, client→server) + payload length + masking key (if masked) + payload.

**④ Example — simple WebSocket server**
```
socket() → bind() → listen()                  // ordinary TCP server setup
conn = accept()                               // client Upgrade request arrives
read HTTP request; verify "Upgrade: websocket"; compute Sec-WebSocket-Accept
send "HTTP/1.1 101 Switching Protocols" + Upgrade + Sec-WebSocket-Accept headers
loop {
    read frame from conn                      // full-duplex: client can send anytime
    broadcast frame to all clients            // server can also push anytime
}
```

**⑤ Common errors/limits**
- Forgetting the server must reply **`101 Switching Protocols`**, not `200 OK`.
- Treating WebSocket as a replacement for HTTP — it *starts* as HTTP and needs port 80/443 + proxies that allow upgrades.
- Overlooking MASK bit: client→server frames MUST be masked (RFC 6455), or the server must close the connection.

**⑥ Conclusion** — HTTP = request/response for documents; WebSocket = upgraded, persistent, full-duplex channel for real-time apps. The comparison table + handshake diagram earn the marks.

---

### M13. SDN: Concept, Architecture and Advantages  [🔴★ NCIT Q6b]

**① Definition** — **Plain meaning:** in a normal network each switch is a self-contained box that both *thinks* (decides where packets go) and *acts* (moves packets). **SDN (Software-Defined Networking)** pulls the *thinking* into a central **software controller** and leaves the switches as simple "forwarding machines" that obey flow rules. The network is programmed like software instead of configured device by device.

**② Diagram**
```
  APPLICATION LAYER  (firewall, LB, routing policies)
         │   northbound API (REST etc.)
         ▼
  CONTROL LAYER      SDN CONTROLLER (the "brain")  — OpenDaylight, ONOS
         │   southbound API (OpenFlow)
         ▼
  DATA LAYER         switches: flow tables, just forward packets
```

**③ Full concept**
- **Traditional:** every switch owns its control plane (routing logic). Decentralised, individually configured, slow to converge.
- **SDN:** control plane extracted to a central controller. Switches obey **flow rules** (match fields + actions) installed by the controller via **OpenFlow**.
- **OpenFlow flow:** packet arrives → switch looks in its flow table:
  - match → do the action (forward/drop/modify);
  - no match → **packet-in** to the controller, which computes the path, installs rules on the relevant switches (**packet-out**), and forwards.
- **Advantages:** centralised control (globally optimal routing), programmability (automation), agility (deploy policies in seconds), better utilisation, vendor independence (switches merely speak OpenFlow).

**④ Example — an OpenFlow rule**
```
match:  ip_src = 10.0.0.1, eth_dst = aa:bb:cc:dd:ee:ff
action: output -> port 3          (forward to port 3)
```

**⑤ Common errors/limits**
- Claiming SDN removes switches — it removes their *intelligence*, not their forwarding role.
- Assuming one controller must be a single box — real deployments use distributed/HA controllers.
- OpenFlow fixed match fields cannot express arbitrary header processing (that's where P4 comes in, see Q48).

**⑥ Conclusion** — SDN = separate the brain (controller) from the muscle (switches); OpenFlow is the protocol between them. The three-layer diagram plus five advantages is a complete 8-mark answer.

---

### M14. TLS/SSL  [🔴 8-mark]

**① Definition** — **Plain meaning:** **TLS (Transport Layer Security)**, formerly SSL, is the *secure envelope* around socket data sitting between the application and TCP. It does three jobs: keep data secret (**encryption**), prove identity (**certificates/authentication**), and detect tampering (**integrity via hashes**). As a programmer you swap `read`/`write` for `SSL_read`/`SSL_write` and the library does the rest.

**② Diagram**
```
 APPLICATION              (HTTP, SMTP, ...)
        │
        ▼
       TLS layer           ← encryption + certificates + integrity
        │
        ▼
        TCP
```
```
 TLS Handshake (simplified):
 CLIENT                                          SERVER
   │                                                │
   │──① ClientHello (version, ciphers, random)──▶ │
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
- Three building blocks:
  1. **Encryption** — **symmetric** (AES-256) is fast; used for bulk data. **Asymmetric** (RSA, Diffie-Hellman) is slow; used only to safely establish the symmetric session key. Ephemeral DH (**ECDHE**) gives **forward secrecy** — past traffic stays secret even if the server key leaks.
  2. **Hashing** — SHA-256 (+ HMAC) detects tampering: any change to the ciphertext breaks the hash and is rejected.
  3. **Certificates** — bind a public key to an identity, digitally signed by a **Certificate Authority (CA)**. Client checks: CA signature valid, hostname matches, not expired.
- **Handshake (7 steps):** ClientHello → ServerHello → Certificate → KeyExchange → client verifies cert → both derive the same session key → Finished messages confirm success. After that, symmetric encryption covers everything.
- **TLS 1.3:** one round trip (0-RTT for resumption), forward-secret key exchange only.

**④ Example — OpenSSL on a socket:**
```c
SSL_CTX *ctx = SSL_CTX_new(SSLv23_method());           /* or TLS_*_method() */
SSL_CTX_use_certificate_file(ctx, "server.pem", SSL_FILETYPE_PEM);
SSL_CTX_use_PrivateKey_file(ctx, "server.key", SSL_FILETYPE_PEM);
SSL *ssl = SSL_new(ctx);
SSL_set_fd(ssl, sockfd);                /* attach to existing connected socket */
SSL_accept(ssl);                        /* server-side handshake */
SSL_write(ssl, "Hello", 5);            /* encrypted write */
SSL_read(ssl, buf, sizeof(buf));        /* encrypted read */
SSL_shutdown(ssl); SSL_free(ssl); SSL_CTX_free(ctx);
```

**⑤ Common errors/limits**
- **Certificate expired / hostname mismatch / untrusted CA** → handshake fails or browser warns.
- On **non-blocking** sockets, `SSL_read`/`SSL_write` return `SSL_ERROR_WANT_READ`/`WANT_WRITE` — not real errors; retry later.
- TLS 1.0/1.1 deprecated — use TLS 1.2+.
- No certificate = no authentication → MITM possible. Prefer ECDHE/DHE cipher suites for forward secrecy.

**⑥ Conclusion** — TLS = encryption + authentication + integrity around your sockets. Attach OpenSSL to the fd and swap read/write for SSL_read/SSL_write; the library handles the hard parts.

---

### M15. gRPC  [🔴 8-mark]

**① Definition** — **Plain meaning:** **gRPC (Google Remote Procedure Call)** lets a client program call a *method on a remote server as if it were a local function*. It is built on **HTTP/2** and uses **Protocol Buffers** for fast binary serialisation. You write a `.proto` file once, generate stubs in any language, and call remote methods like local ones.

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
 │ Server streaming │  request ──▶ server ──▶ response, response... │
 │                  │  (1:N — server sends a stream)                │
 ├──────────────────┼───────────────────────────────────────────────┤
 │ Client streaming │  request, request… ──▶ server ──▶ response   │
 │                  │  (N:1 — client sends a stream)                │
 ├──────────────────┼───────────────────────────────────────────────┤
 │ Bidi streaming   │  request, request… ◀──▶ response, response…  │
 │                  │  (N:N — both sides stream simultaneously)     │
 └──────────────────┴───────────────────────────────────────────────┘
```

**③ Full concept**
- Define the service in a `.proto` file (`syntax="proto3"`; `service Greeter { rpc SayHello(...) returns (...); }` with `message` definitions).
- `protoc` + the gRPC plugin generate client and server **stubs** in many languages. The client calls a generated method; it serialises the request to protobuf, sends it over HTTP/2; the server stub deserialises, runs your real method, and sends the response back.
- **HTTP/2 gives:** multiplexing (many calls on one connection), header compression (HPACK), bidirectional streaming.
- **Protobuf gives:** compact binary messages (much smaller/faster than JSON), strong typing at compile time, backward/forward compatibility (field numbers, optional fields).
- Extra: **deadlines/timeouts**, cancellation, interceptors (middleware), load balancing.

**④ Example — complete flow:**
```bash
# 1. define service
cat > greeter.proto <<EOF
syntax = "proto3";
package greeter;
service Greeter { rpc SayHello (HelloRequest) returns (HelloReply); }
message HelloRequest { string name = 1; }
message HelloReply   { string message = 1; }
EOF
# 2. generate code
protoc --cpp_out=. --grpc_out=. --plugin=protoc-gen-grpc=$(which grpc_cpp_plugin) greeter.proto
# 3. implement + run server; 4. client calls generated stub
Greeter::Stub stub(channel);
HelloRequest req; req.set_name("World");
HelloReply reply; ClientContext ctx;
Status status = stub.SayHello(&ctx, req, &reply);   // reply.message() == "Hello World"
```

**⑤ Common errors/limits**
- **Binary payloads** aren't human-readable — debugging needs `grpcurl`, Wireshark + protobuf dissector, or gRPC reflection.
- **Toolchain dependency:** `protoc` + language plugins must be installed; proto version mismatch between client and server can fail silently.
- **HTTP/2 required** — proxies/LBs without HTTP/2 break gRPC; need L7 LB or a service mesh.
- Heavier than raw TCP for trivial one-off calls.

**⑥ Conclusion** — gRPC is the modern default for microservices/streaming needing performance, strong typing, and multi-language support: hand-written JSON/REST replaced by contract-driven binary RPC over HTTP/2.

---

### M16. WebSockets (short note, also see M12)

**① Definition** — **Plain meaning:** WebSocket is a **full-duplex, persistent** protocol over a single TCP connection — one upgrade (via HTTP) turns a normal request/response link into a channel where both sides send messages anytime, with almost no overhead.

**② Handshake** — starts as an HTTP request: client sends `GET /chat HTTP/1.1` with `Upgrade: websocket` and `Sec-WebSocket-Key: <base64>`; server replies `101 Switching Protocols` with `Sec-WebSocket-Accept: <hash>`; after this the connection is upgraded to WebSocket and both sides exchange frames.

**③ Frame format** — each frame starts with: FIN (1 bit, final fragment?) + opcode (4 bits: 0x1 text, 0x2 binary, 0x8 close, 0x9 ping, 0xA pong, 0x0 continuation) + MASK (1 bit, must be 1 for client→server) + payload length (7 bits; 126→2 extra bytes; 127→8 extra bytes) + masking key (4 bytes, if masked) + payload.

**④ Use cases** — chat applications, multiplayer games, live dashboards, stock tickers, IoT real-time push, collaborative editing.

**⑤ Limits** — idle connections may be killed by proxies after ~60 seconds (need ping/pong keepalive); masking adds overhead; server must handle many concurrent connections (select/poll/async).

**⑥ Conclusion** — WebSocket replaces HTTP's request/response with a persistent, low-overhead, full-duplex channel — the standard for real-time web applications.

---

### M17. Blocking vs Non-blocking I/O  [🔴★ Gandaki Q4a]

**① Definition** — **Plain meaning:** when a `recv` has no data yet, what happens? **Blocking:** the call *sleeps* until data is ready and copied. **Non-blocking:** the call *returns immediately* with `EWOULDBLOCK`, so your program can do other things and try again later. Non-blocking alone wastes CPU (polling); it shines when combined with `select`.

**② Diagram**
```
 BLOCKING MODE (default):
 app calls recvfrom → kernel: data not ready, process sleeps → data arrives → copied → return
 (thread tied up the entire time)

 NON-BLOCKING MODE:
 app calls recvfrom → kernel: not ready → EWOULDBLOCK → app does other work → try again
 (thread free between calls, but polling alone wastes CPU)
```

**③ Full concept**
- **Blocking (default):** `recvfrom` does not return until data arrives *and* is copied. Simple code, low CPU, but one blocking `recv` ties up the whole thread — a blocking server needs thread/fork per client.
- **Non-blocking:** set with `fcntl(F_SETFL, O_NONBLOCK)` (Unix) or `ioctlsocket(FIONBIO)` (Windows). `recvfrom` returns immediately: -1 + `EWOULDBLOCK` if no data, else the byte count.
- **The practical pattern:** non-blocking **+ `select`** — select blocks efficiently until some fd is ready, then non-blocking reads on the ready fds give data immediately (handling premature "spurious readiness" gracefully).
- Both modes' real `recvfrom` copy still blocks the process — both are **synchronous**.

**④ Example/code**
```c
// Unix
int flags = fcntl(sockfd, F_GETFL, 0);
fcntl(sockfd, F_SETFL, flags | O_NONBLOCK);
ssize_t n = recvfrom(sockfd, buf, MAXLINE, 0, NULL, NULL);
if (n == -1 && errno == EWOULDBLOCK) { /* no data yet — try later */ }
// Windows
unsigned long mode = 1;
ioctlsocket(s, FIONBIO, &mode);        /* recv returns SOCKET_ERROR + WSAEWOULDBLOCK */
```

**⑤ Common errors/limits**
- Non-blocking without select/poll → busy-waiting at 100% CPU.
- Confusing `recv` returning 0 (peer closed **orderly**) with the -1/EWOULDBLOCK "no data yet" case.
- `connect()` on non-blocking sockets returns immediately with `EINPROGRESS` — use select to learn when the connection completed.

**⑥ Conclusion** — Blocking = simple but thread-heavy; non-blocking = thread-free but wasteful alone; the standard is **non-blocking + select/poll** for efficient one-thread-many-sockets servers.

---

### M18. Signal-driven I/O vs I/O Multiplexing  [🔴★ Gandaki Q4a]

**① Definition** — **Plain meaning:** both answer *"how do I know when a socket has data?"* — **I/O multiplexing**: block in `select()`/`poll()` watching many descriptors. **Signal-driven I/O**: the kernel *interrupts* you with **SIGIO** when a descriptor is ready, so the main loop is never blocked.

**② Diagram**
```
 I/O MULTIPLEXING:
   app: select(sockfd+1, &readfds, ...) → kernel waits → select returns when readable
   app: recvfrom(sockfd, ...)            (blocks briefly here)

 SIGNAL-DRIVEN I/O:
   app setup: sigaction(SIGIO, handler); fcntl(F_SETOWN); fcntl(F_SETFL, O_ASYNC)
   main loop runs free (other work)
   kernel: socket readable → sends SIGIO → signal handler: recvfrom(sockfd, ...)
```

**③ Full concept**
- **Multiplexing:** call `select`/`poll` with many fds; it blocks; on return you know which are ready and `recvfrom` them (that call now returns immediately). Two system calls per read (select + recvfrom), supports many fds at once, reliable.
- **Signal-driven:** enable the socket with `F_SETOWN` + O_ASYNC, install a `sigaction` handler. When readable, the kernel sends `SIGIO`; the handler runs `recvfrom`. Main loop never blocks — free for other work.
- **Key differences:** signal-driven never blocks the main thread but signals can merge/be missed and handlers need reentrancy care; multiplexing is reliable and returns *all* ready fds at once but blocks the thread in `select`.
- **Both synchronous:** the actual `recvfrom` still blocks briefly in both; only POSIX `aio_*` (model 5) is truly asynchronous.

**④ Example/code**
```c
void sigio_handler(int signo) {
    ssize_t n = recvfrom(sockfd, buf, MAXLINE, 0, NULL, NULL);
    printf("Received %zd bytes\n", n);
}
int main() {
    struct sigaction sa; sa.sa_handler = sigio_handler;
    sigemptyset(&sa.sa_mask); sigaction(SIGIO, &sa, NULL);
    fcntl(sockfd, F_SETOWN, getpid());
    int flags = fcntl(sockfd, F_GETFL);
    fcntl(sockfd, F_SETFL, flags | O_ASYNC);
    while (1) { /* main loop: do other work; SIGIO interrupts when data arrives */ }
}
```

**⑤ Common errors/limits**
- Multiple simultaneous SIGIOs may **merge/lost** (signals aren't queued).
- Handlers must be **reentrant** (no mutex/printf/malloc in them).
- `FD_SETSIZE` limits multiplexing; signal-driven has no such limit but needs careful signal management.
- Only one signal is delivered per fd readiness event — drain all data non-blocking inside the handler.

**⑥ Conclusion** — Multiplexing = reliable, blocks in `select`; signal-driven = main loop free but fragile. Prefer `select`/`poll` for servers unless very low latency is needed.

---

### M19. Daemonizing a Process (with code)  [🔴★ NCIT Q4a]

**① Definition** — **Plain meaning:** a **daemon** is a long-running background process with **no controlling terminal** — it survives logout and runs until shutdown (sshd, httpd, crond). **Daemonizing** detaches a program from the terminal, working directory, umask, and standard file descriptors so nothing accidental kills it or clutters it.

**② Diagram — the full daemonization sequence**
```
 parent (shell) ──fork()──▶ child ── parent exits, child orphaned/adopted by init
                                  ── setsid(): new session, detached from controlling TTY
                                  ── (2nd fork optional): can never re-acquire a TTY
                                  ── chdir("/"): don't hold a mount busy
                                  ── umask(0): full control over file creation
                                  ── fd 0,1,2 → /dev/null: no stray output
                                  ▼
                       DAEMON RUNNING IN BACKGROUND
```

**③ Full concept — why each step:**
1. **`fork()` + parent `exit()`:** child becomes an orphan adopted by init (PID 1); shell regains its prompt; the child is not a session leader (needed for `setsid`).
2. **`setsid()`:** new session + process group, no controlling terminal. Without it, logout sends SIGHUP and kills the daemon.
3. **(Optional) second `fork()`:** a session leader can still re-acquire a controlling terminal by opening a terminal device without `O_NOCTTY`; the second fork (parent exits) makes the grandchild a non-session-leader, preventing that. This is what `daemon(3)` does.
4. **`chdir("/")`:** the daemon's inherited working directory may be on a removable/mounted filesystem; moving to `/` frees it.
5. **`umask(0)`:** clears the file-mode creation mask so the daemon can create files with the permissions it actually requests.
6. **fd 0/1/2 → `/dev/null`:** stdin/stdout/stderr inherited from the shell point to the terminal; pointing them to `/dev/null` makes reads EOF and writes vanish.

**④ Example/code — complete daemonization**
```c
void daemonize(void) {
    pid_t pid;
    if ((pid = fork()) < 0) err_sys("fork error");
    if (pid != 0) exit(0);                // 1. parent (shell) exits
    setsid();                             // 2. new session, no controlling TTY
    if ((pid = fork()) < 0) err_sys("fork error");
    if (pid != 0) exit(0);                // 3. 2nd fork: not a session leader
    umask(0);                             // 4. clear create-mode mask
    if (chdir("/") < 0) err_sys("chdir error");   // 5. safe cwd
    int fd = open("/dev/null", O_RDWR);   // 6. redirect std fds
    if (fd >= 0) { dup2(fd, 0); dup2(fd, 1); dup2(fd, 2); if (fd > 2) close(fd); }
}
```

**⑤ Common errors/limits**
- **Forgetting `setsid()`** → daemon still has a controlling terminal → killed on logout.
- **Not redirecting fd 0/1/2** → `printf` writes to a dead terminal or the wrong user's screen.
- **Modern systemd:** daemons usually run as *foreground children of systemd*, which handles terminal/umask/cwd/signals — in that model, do NOT daemonize yourself.
- Closing inherited fds beyond 0/1/2 (DB connections, log files) is also good practice.

**⑥ Conclusion** — fork → setsid → fork → chdir → umask → redirect produces a proper daemon; on modern Linux systemd manages this for you. Key exam points: *why* each step, especially setsid (terminal detach) and the fd redirect.

---

### M20. Socket Options (SO_LINGER, SO_KEEPALIVE, SO_REUSEADDR, SO_BROADCAST)  [🔴★ NCIT Q5b, Gandaki Q4b]

**① Definition** — **Plain meaning:** socket options are kernel **configuration switches** for a socket, set via `setsockopt`/read via `getsockopt`. The four exam options solve classic server problems: reuse a port still in TIME_WAIT (SO_REUSEADDR), allow broadcasting (SO_BROADCAST), detect dead peers (SO_KEEPALIVE), and control what close does with unsent data (SO_LINGER).

**② Diagram**
```
 ┌─────────────┐  setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on))  ┌────────┐
 │ application │ ──────────────────────────────────────────────────────────▶ │ kernel │
 │             │ ◀───────────────────────────────────────────────────────── │ socket │
 └─────────────┘  getsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &val, &len)       └────────┘
```

**③ Full concept**
- **`SO_REUSEADDR`:** allows binding to a port in **TIME_WAIT** — otherwise a restarting server gets "Address already in use". Set **before** `bind()`. Essential for all TCP servers.
- **`SO_BROADCAST`:** lets UDP sockets send to broadcast addresses (e.g. 192.168.1.255). Off by default — without it you get `EACCES`. Routers do not forward broadcast (local subnet only).
- **`SO_KEEPALIVE`:** TCP sends keepalive probes after 2 hours idle. ACK → alive; RST → peer crashed (ECONNRESET); silence → 8 probes 75 s apart, then ETIMEDOUT. Detects dead peers on long-lived connections.
- **`SO_LINGER`:** controls `close()`:
  ```
  struct linger { int l_onoff; int l_linger; };
  l_onoff=0                  → close() returns immediately (default)
  l_onoff≠0, l_linger=0     → TCP ABORTS: discard buffer, send RST (no FIN)
  l_onoff≠0, l_linger≠0     → close() BLOCKS until data ACKed or timeout
  ```
- Buffer options (`SO_RCVBUF`/`SO_SNDBUF`) must be set **before** connect/listen.

**④ Example/code**
```c
int on = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));  // before bind
setsockopt(udpfd,   SOL_SOCKET, SO_BROADCAST,  &on, sizeof(on));  // before sendto
setsockopt(connfd,  SOL_SOCKET, SO_KEEPALIVE,  &on, sizeof(on));  // connected socket
struct linger li = {1, 5};
setsockopt(connfd,  SOL_SOCKET, SO_LINGER, &li, sizeof(li));      // wait up to 5 s
struct linger li2 = {1, 0};
setsockopt(connfd,  SOL_SOCKET, SO_LINGER, &li2, sizeof(li2));    // abort now (RST)
```

**⑤ Common errors/limits**
- `SO_REUSEADDR` semantics differ Linux/BSD/Windows; Linux may also want `SO_REUSEPORT` for multiple listeners.
- `SO_KEEPALIVE`'s 2-hour default is too long for most apps — tune `TCP_KEEPIDLE` or use app heartbeats.
- `SO_LINGER` with a timeout **blocks `close()`** — dangerous if the peer is slow; can stall a server.
- Buffer options set after connect/listen have no effect.

**⑥ Conclusion** — These four options solve the most common server problems (restart-without-conflict, broadcast send, dead-peer detection, graceful vs abrupt close). The `struct linger` table is a favourite exam question — memorise it.

---

### M21. WSAAsyncSelect vs WSAEventSelect  [🟡★ NCIT alt]

**① Definition** — **Plain meaning:** two Winsock async I/O models for telling a program "your socket is ready" — **WSAAsyncSelect** delivers that news as a **Windows message to a window** (GUI apps), **WSAEventSelect** sets a **Win32 event object** instead (console/services, no window needed). Same job, different delivery channel.

**② Diagram**
```
 WSAAsyncSelect (message-based):
 socket ── FD_READ ──▶ Winsock DLL ── posts WM_SOCKET message ──▶ WndProc (window)
 WSAEventSelect (event-based):
 socket ── FD_READ ──▶ Winsock DLL ── signals hEvent object ──▶ WSAWaitForMultipleEvents
```

**③ Full concept**
- **WSAAsyncSelect:** associates a socket with a **window handle (HWND)** and a Windows message (`WM_SOCKET`). On any event (FD_READ, FD_WRITE, FD_OOB, FD_ACCEPT, FD_CONNECT, FD_CLOSE), Winsock posts a message; the window procedure decodes `wParam` (socket) and `lParam` (event) and reacts. Calling it switches the socket to **non-blocking**. Needs a window + message pump — natural for GUI apps, unusable in console apps/services.
- **WSAEventSelect:** creates an **event object** (`WSACreateEvent`), associates it with the socket (`WSAEventSelect(s, hEvent, FD_READ|FD_CLOSE)`), waits with `WSAWaitForMultipleEvents`, then calls `WSAEnumNetworkEvents` to learn which socket/event fired. No window needed — good for console/background apps. Max **64 events per thread**.
- **Both are non-blocking but synchronous** — they tell you *when to read*; you still do the `recv`. The I/O is not done automatically by the kernel (only overlapped I/O/IOCP run I/O in the background).

**④ Example/code (WSAEventSelect)**
```c
WSAEVENT events[2]; SOCKET socks[2];
events[0] = WSACreateEvent(); WSAEventSelect(listenSock, events[0], FD_ACCEPT);
events[1] = WSACreateEvent(); WSAEventSelect(connSock,  events[1], FD_READ | FD_CLOSE);
while (1) {
    DWORD idx = WSAWaitForMultipleEvents(2, events, FALSE, WSA_INFINITE, FALSE);
    idx -= WSA_WAIT_EVENT_0;
    WSANETWORKEVENTS ne; WSAResetEvent(events[idx]);
    WSAEnumNetworkEvents(socks[idx], events[idx], &ne);
    if (ne.lNetworkEvents & FD_ACCEPT)  { SOCKET nc = accept(listenSock, 0, 0); /* add nc */ }
    if (ne.lNetworkEvents & FD_READ)    { recv(socks[idx], buf, sizeof(buf), 0); }
}
```

**⑤ Common errors/limits**
- WSAAsyncSelect **requires a window + message pump** — not usable in console apps/services.
- WSAEventSelect limited to **64 events per thread** — build your own mapping event→socket.
- Neither model does the I/O for you — you still call `recv`/`send` (only overlapped I/O/IOCP run I/O in the background).
- Both switch the socket to non-blocking automatically — use `ioctlsocket(FIONBIO, 0)` if you want blocking back.

**⑥ Conclusion** — WSAAsyncSelect = message-driven, for GUI apps; WSAEventSelect = event-object-driven, for console/services. Both are non-blocking notification models that tell you *when* to do I/O.

---

### M22. Securing a Network Application (hostname / IP / wrapper)  [🟡★ short note]

**① Definition** — **Plain meaning:** securing a network app = keeping unwanted clients out (**access control**: hostname, IP, wrapper) **and** protecting the data (**TLS**). Three access-control methods are asked about — check who connects by *hostname*, by *IP*, or through a *wrapper program* that gates every connection — plus encryption on top.

**② Diagram**
```
 CLIENT ──▶ [WRAPPER PROGRAM / in.tcpd]
               │ checks: trusted hostname? (DNS lookup)
               │ checks: IP in allow list? (/etc/hosts.allow)
               ├─ ALLOWED ──▶ start real service, relay data
               └─ DENIED ──▶ log the attempt, drop connection, close
```

**③ Full concept**
- **By hostname/domain:** resolve the client IP to a hostname (reverse DNS) and allow only trusted names. ⚠️ **DNS can be spoofed** — weak alone.
- **By IP number:** restrict with `/etc/hosts.allow` + `/etc/hosts.deny` (TCP wrappers) or firewall rules (`iptables`/`nftables`). Simple; IPs can be spoofed but it's harder than DNS spoofing.
- **Wrapper program (TCP wrappers / `in.tcpd`):** a small front-end intercepts the connection, checks the client against policy, and *only if allowed* launches the real service and relays data. Adds access control **without modifying the server application** — the server binary stays untouched.
- **TLS/SSL:** encrypts all data in transit, authenticates the server via certificates, detects tampering — essential *in addition* to access control (access control alone leaves data in clear text).

**④ Example — `/etc/hosts.allow` + `/etc/hosts.deny`:**
```
# /etc/hosts.allow
sshd: 192.168.1.0/24          # allow SSH from the local subnet
in.telnetd: .trusted.com      # allow telnet from the trusted.com domain
# /etc/hosts.deny
ALL: ALL                      # deny everything not explicitly allowed
```

**⑤ Common errors/limits**
- Trusting hostname alone (DNS spoofing); relying on source IP alone (IP spoofing).
- Forgetting that access control is **not encryption** — data is still in the clear. Pair with TLS.
- TCP wrappers are deprecated on modern Linux (systemd doesn't use them) — use firewall rules instead; the concept is identical.

**⑥ Conclusion** — A layered defence (hostname allowlist + IP firewall + wrapper + TLS) protects against unwanted connections and in-transit tampering; the wrapper concept is the key exam idea — gating access without touching the server code.

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
