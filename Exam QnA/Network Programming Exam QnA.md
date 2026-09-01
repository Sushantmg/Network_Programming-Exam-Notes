# Network Programming — Exam Questions & Answers

**CMP 380 Network Programming (Pokhara University)**
Compiled from the **actual NCIT Spring 2025** and **Gandaki College 2025** question papers (extracted from screenshots), the official syllabus, and the full study notes.

## How to use this guide
- Each question has a **priority rating** based on how often it appeared in real past papers:
  - 🔴 **HIGH** — appears in nearly every paper / asked repeatedly (study first, be able to write full 8-mark answer).
  - 🟡 **MEDIUM** — appears often, usually as 5–7 mark (know well).
  - 🟢 **LOW** — appears occasionally / as a short note.
- Questions asked almost **word-for-word** in the captured papers are marked **★ (asked in real paper)**.

---

## Unit 1 — Fundamentals

### Q1 🔴★ Compare and contrast TCP, UDP, and SCTP. (asked: NCIT 2025 Q1a)
All three are **transport-layer protocols**; they differ in connection, reliability, ordering, and message handling.

| Feature | TCP | UDP | SCTP |
|---|---|---|---|
| Connection | Connection-**oriented** | **Connectionless** | Connection-oriented |
| Reliability | Reliable (ACK + retransmit) | Unreliable | Reliable |
| Ordering | Byte stream, sequenced | No ordering | Message-ordered |
| Message boundaries | **No** (pure byte stream) | **Yes** (datagrams) | **Yes** (messages) |
| Multi-homing | No | No | **Yes** (multiple IPs) |
| Data transfer | Stream | Datagram | Message + stream |
| Usage | HTTP, FTP, SMTP | DNS, NFS, SNMP | Telephony/Signaling (SIGTRAN) |

### Q2 🔴★ Explain the TCP three-way handshake. (asked in multiple papers)
Three segments to open a connection:
```
CLIENT                                  SERVER
   │  SYN (seq = x)                        │
   │ ────────────────────────────────▶     │  (client wants to connect)
   │  SYN+ACK (seq = y, ack = x+1)         │
   │ ◀────────────────────────────────     │  (server agrees)
   │  ACK (ack = y+1)                      │
   │ ────────────────────────────────▶     │  (both sides now ready)
   │          CONNECTION ESTABLISHED       │
```
Explained:
1. Client sends **SYN** (synchronize) with an initial sequence number **x** (active open).
2. Server replies **SYN+ACK** — its own seq **y**, acknowledging **x+1** (passive open).
3. Client sends **ACK** acknowledging **y+1**.
Called *three-way* because at least three packets are exchanged (though the SYN+ACK is one segment).

### Q3 🔴★ Why should the initial sequence number NOT start from 0? Explain TCP state transition diagram. (asked: NCIT 2025 Q1a — 7 marks)
**Why not 0:** If ISNs always started at 0, an old, delayed packet from a previous connection (in the network) could carry a sequence number that **collides** with a live connection's data and be mistakenly accepted (it would look like valid new data). Starting from a **random/guessable-but-unpredictable ISN** and relying on **TIME_WAIT** lets old duplicates expire, preventing such mix-ups. (Also a guessed ISN has security implications.)

**TCP state-transition diagram** (simplified, draw this):
```
             ┌─────────────────────────────────────────────────┐
             │                                                 │
   CLIENT -----> CLOSED ------> LISTEN <-----------------------+--- (server)
                │    │ (passive open)                           │
      (active   │    ▼                                          │
       open)    │  SYN_SENT --->  (recv SYN+ACK) ---> ESTABLISHED
                │        ▲           │  (send ACK)              ▲
                │        └───────────┘   │                      │
                │                        │  (simultaneous open) │
   server path: │  LISTEN ---> (recv SYN) ---> SYN_RCVD           │
                │        (send SYN+ACK)   │                      │
                │                         ▼                      │
                │                     ESTABLISHED ---------------┘
                ▼
             (either side) FIN_WAIT_1 <---> CLOSE_WAIT
             FIN_WAIT_2                     LAST_ACK
             TIME_WAIT (2×MSL)              |
                                          CLOSED
```
**Key states:** `CLOSED` → `LISTEN` (server) / `SYN_SENT` (client) → `ESTABLISHED` (both) → on close: `FIN_WAIT_1/2` and `TIME_WAIT` (initiator), `CLOSE_WAIT` and `LAST_ACK` (receiver) → `CLOSED`.

### Q4 🟡★ What is a socket? What are the types of sockets? (asked: NCIT 2025 Q2 — differentiate TCP vs UDP socket)
A **socket** = an **endpoint for communication** = a combination of **IP address + port number** used to identify a connection.
- **Stream socket (SOCK_STREAM)** → TCP — connection-oriented, reliable, ordered **byte stream** (no message boundaries).
- **Datagram socket (SOCK_DGRAM)** → UDP — connectionless, unreliable, preserves **message (datagram) boundaries**.
- **Raw socket (SOCK_RAW)** → direct access to IP packets (for low-level tools, e.g., ping).

**TCP vs UDP socket (asked directly):**
| | TCP socket | UDP socket |
|---|---|---|
| Type | SOCK_STREAM | SOCK_DGRAM |
| Connection | must `connect` first | connectionless |
| Reliable | yes (ack/retransmit) | no |
| Ordering/messages | byte stream | datagram boundaries |
| Server calls | bind→listen→accept | bind→recvfrom only |
| Client send | `send`/`write` | `sendto` |

### Q5 🟢 What is IPC? List the evolution of UNIX IPC mechanisms.
IPC = **Interprocess Communication** — allowing processes to exchange data. Timeline:
```
Pipes → Named pipes/FIFOs → System V msg queues → POSIX msg queues → RPC
```

### Q6 🟢 What are the three ways two UNIX processes can share info?
1. **Through a file** in the filesystem (through the kernel) — needs synchronization.
2. **Through the kernel** (pipes, message queues, semaphores) — each is a system call.
3. **Through shared memory** — direct, no kernel per-access — still needs synchronization.

### Q7 🟡★ Define the port number ranges & socket pair.
- **0–1023**: well-known (e.g., 80 HTTP, 443 HTTPS).
- **1024–49151**: registered.
- **49152–65535**: dynamic/private (ephemeral, auto-assigned to clients).
**Socket pair** (for TCP) = the **4-tuple** *(local IP, local port, foreign IP, foreign port)* — uniquely identifies every TCP connection.

### Q8 🟡★ How is a TCP connection terminated? Why is TIME_WAIT needed?
Four segments: **FIN → ACK → FIN → ACK**.
**TIME_WAIT** (lasts **2× MSL**): 1) ensures the final ACK wasn't lost (reliable full-duplex close), and 2) lets **old duplicate segments** expire in the network so they can't contaminate a new connection.

---

## Unit 2 — Unix Basics

### Q9 🔴★ Explain value-result arguments in socket programming. Why are they needed? (asked: NCIT 2025 Q3a, Gandaki 2025 Q2b)
For functions that take a socket-address length:
- **Process → kernel** (`bind`, `connect`, `sendto`): you pass the **size by value** — the kernel knows how much to copy.
- **Kernel → process** (`accept`, `recvfrom`, `getsockname`, `getpeername`): you pass a **pointer to the size** (a `socklen_t *`). It is a **value** on input (tells the kernel the buffer size so it doesn't write past the end) and a **result** on output (tells you how many bytes were actually stored). ⇒ **value-result argument**.

**Why needed:** the kernel must know the buffer size to avoid overflow, and the caller must learn the real size afterwards.
```c
struct sockaddr_in cli;
socklen_t len = sizeof(cli);          /* value */
accept(listenfd, (SA*)&cli, &len);    /* len is updated to actual size → result */
```

### Q10 🔴★ Ways to pass the length of a socket structure — with function prototypes. (asked: NCIT 2025 Q2a)
Two ways as above, demonstrated by the two groups of functions:
```c
/* Group 1: length passed BY VALUE (process→kernel) */
int bind(int s, const struct sockaddr *addr, socklen_t addrlen);
int connect(int s, const struct sockaddr *addr, socklen_t addrlen);
ssize_t sendto(int s, const void *buf, size_t len, int flags,
               const struct sockaddr *to, socklen_t tolen);

/* Group 2: length passed BY REFERENCE, value-result (kernel→process) */
int accept(int s, struct sockaddr *addr, socklen_t *addrlen);
ssize_t recvfrom(int s, void *buf, size_t len, int flags,
                 struct sockaddr *from, socklen_t *fromlen);
int getsockname(int s, struct sockaddr *addr, socklen_t *addrlen);
int getpeername(int s, struct sockaddr *addr, socklen_t *addrlen);
```
In group 2 the caller sets `*addrlen` before the call; the kernel updates it afterward to the real length.

### Q11 🔴★ Explain the socket address structures (sockaddr, sockaddr_in, sockaddr_in6, sockaddr_storage). (asked: NCIT 2025 Q2b, Gandaki 2025 Q2a/Q2b)
| Structure | Family | Size | Purpose |
|---|---|---|---|
| `struct sockaddr` | generic | 16 B | generic/old casting form (`sa_family`, `sa_data[14]`) |
| `struct sockaddr_in` | AF_INET | 16 B | IPv4 (`sin_family`, `sin_port`, `sin_addr`, `sin_zero[8]`) |
| `struct sockaddr_in6` | AF_INET6 | 28 B | IPv6 (`sin6_port`, `sin6_flowinfo`, `sin6_addr`, `sin6_scope_id`) |
| `struct sockaddr_storage` | generic | ≥128 B | large enough for **any** family + strict alignment |

**Why `sockaddr_storage` is significant** (asked): it is big enough to hold **any** socket-address type the system supports (IPv4 or IPv6) and gives the strictest **alignment**, so you can pass it to `accept`/`recvfrom` without knowing which family will arrive.

### Q12 🔴★ Byte ordering & manipulation functions. (asked: NCIT 2025 Q2a)
**Endianness:** a 2-byte integer `0x1234` stored as `34 12` (little-endian, Intel) or `12 34` (big-endian). Internet protocols use **network byte order = big-endian**.
```c
uint16_t htons(uint16_t hostshort);  /* host → network, short  */
uint32_t htonl(uint32_t hostlong);   /* host → network, long   */
uint16_t ntohs(uint16_t netshort);   /* network → host, short  */
uint32_t ntohl(uint32_t netlong);    /* network → host, long   */
```
Apply to `sin_port` (htons) and `sin_addr` (htonl).
**Manipulation functions** (for binary data, not C strings): `bzero/bcopy/bcmp` (BSD) and `memset/memcpy/memcmp` (ANSI). Use them to zero/init address structures.

### Q13 🟡★ inet_aton / inet_addr / inet_ntoa / inet_pton / inet_ntop.
- `inet_aton("1.2.3.4",&addr)` — dotted → binary (preferred).
- `inet_addr("1.2.3.4")` — same, returns value.
- `inet_ntoa(addr)` — binary → dotted string.
- `inet_pton/ntop` — presentation↔numeric for **both IPv4 and IPv6**.

### Q14 🔴★ Write the TCP server & client system-call sequence. (asked multiple times)
```
SERVER: socket() → bind() → listen() → accept() → read()/write() → close()
CLIENT: socket() → connect() → read()/write() → close()          (bind optional)
```

### Q15 🔴 What happens if you call bind() in a TCP client? (asked: NCIT 2025 Q3b)
A TCP client normally does **not call bind()** — the kernel automatically assigns an **ephemeral port** at `connect()`. If you call `bind()`:
- You force a specific local IP/port instead of the automatic ephemeral one.
- It can fail with **EADDRINUSE** if that port is already taken.
- It's only needed when a client must bind to a particular source port/interface (e.g., some FTP modes).
**send/recv in UDP, sendto/recvfrom in TCP** (same paper): In principle you *could* use `sendto`/`recvfrom` once a TCP connection exists, but normal TCP uses `write`/`read` (or `send`/`recv`); `recvfrom`/`sendto` are the natural calls for **UDP datagrams** where you must specify/learn the peer address on each datagram.

### Q16 🟡★ What is a daemon? How do you daemonize a process in UNIX? (asked: NCIT 2025 Q4a)
A **daemon** = long-running background process with **no controlling terminal**, started at boot, runs until shutdown. **To daemonize:**
```c
fork();            /* 1. child = daemon, parent exits   */
setsid();          /* 2. new session; detach from tty   */
chdir("/");        /* 3. safe working directory         */
umask(0);          /* 4. clear file-mode mask           */
/* 5. redirect stdin/stdout/stderr to /dev/null */
```
Sample: `if (fork() > 0) exit(0); setsid(); chdir("/"); umask(0); open("/dev/null"); dup2(0,1); dup2(0,2);`

### Q17 🟡★ signal() and sigaction(), and signal handling in UNIX. (asked: Gandaki 2025 Q3a)
Signals are **async notifications** the kernel/program can send (SIGINT, SIGIO, etc.).
- `signal(signum, handler)` — simple; a handler is a function `void h(int)`.
- `sigaction(signum, &act, &old)` — **more powerful/portable**: you can block other signals during handling, control `SA_RESTART`, get flags — safer for network apps (avoids losing a signal or getting `EINTR`).
**Handling in network programs:** use `SIGIO` for signal-driven I/O readiness, `SIGCHLD` to reap zombie children in a fork-based concurrent server, and handle interrupted syscalls returning `EINTR` (or use `SA_RESTART`).

### Q18 🟡★ fork() and exec() — creating processes. (asked: NCIT 2025 Q4a)
- `fork()` — creates a **child process**, an exact **copy** of the parent (own PID; returns 0 in child, child's PID in parent; -1 on error). Parent & child continue concurrently.
- `exec()` — **replaces the current process image** with a new program and runs it from entry. Never returns on success. Variants: `execl, execlp, execle, execv, execvp, execve`.
- In servers: `fork()` after `accept()` so each client gets its own child; `exec()` to replace with a program (e.g., a shell).

### Q19 🟢★ UNIX domain sockets & socketpair. (asked: short note)
`AF_UNIX`/`AF_LOCAL` sockets for **same-host IPC** — faster than TCP. Stream/datagram/seqpacket forms. Used by X Window; **pass file descriptors** between processes via `sendmsg/recvmsg`. `socketpair()` creates two connected local sockets at once (full-duplex pipe for stream).

### Q19b 🟡 Hostname & service name resolution (gethostbyname / getservbyname / getaddrinfo).
Computers need **IP addresses**, but humans use **names**. Resolution converts between them:
- **`gethostbyname(name)`** → `struct hostent *` for a hostname: official name, aliases, address family, addr length, and `h_addr_list[]` (the IP addresses, network byte order). It reads **DNS**.
- **`gethostbyaddr(addr, len, type)`** → the reverse: IP → official hostname (PTR lookup).
- **`getservbyname(name, proto)`** → `struct servent *` giving the **port** for a well-known service from the `services` database, e.g. `getservbyname("http","tcp")` → port 80.
- **`getservbyport(port, proto)`** → reverse: port → service name.
- **Modern alternative `getaddrinfo()`** — one function does name → *address structures* for both IPv4/IPv6 (returns a linked list of `addrinfo`), and is preferred for new code because it's family-neutral and thread-safe.

```c
struct hostent *hp = gethostbyname("www.example.com");
/* hp->h_addr_list[0] is the first IP in network byte order */
```

> **Limit:** `gethostbyname` blocks (sync DNS) and only does IPv4; it is **not thread-safe** (returns pointer to static data). Use `getaddrinfo` for production code.

---

## Unit 3 — Advanced Unix

### Q20 🔴★ The five I/O models; which are synchronous? (asked: Gandaki 2025 Q4a)
Two phases of I/O: **waiting for data to be ready** + **copying data from kernel to process**.
1. **Blocking I/O** — process sleeps until data arrives AND is copied; then returns.
2. **Nonblocking I/O** — returns `EWOULDBLOCK` if not ready; must **poll** (wastes CPU).
3. **I/O multiplexing (select/poll)** — block on *select* until a descriptor is ready; then read it. (Needs 2 syscalls.)
4. **Signal-driven I/O (SIGIO)** — kernel signals us when the descriptor is *ready to read*.
5. **Asynchronous I/O (aio_/POSIX)** — kernel tells us when the operation is *complete* (data already copied).

**Synchronous vs async (POSIX):** A **synchronous** I/O op **blocks** the process until *that op* completes — models **1–4 are all synchronous** (they block on `recvfrom`). Only model **5 (asynchronous I/O)** is truly asynchronous. This is the key differentiator.

### Q21 🟡★ Blocking vs Non-blocking I/O. (asked: Gandaki 2025 Q4a)
- **Blocking:** `recvfrom` does not return until data is ready and copied; process sleeps; simple but ties up the thread.
- **Non-blocking:** kernel returns `EWOULDBLOCK` instead of sleeping; we keep asking (polling) — responsive to many sockets but wastes CPU; combine with `select` to avoid busy-waiting.

### Q22 🔴★ Explain signal-driven I/O, compare with I/O multiplexing. (asked: Gandaki 2025 Q4a, NCIT 2025 alternative)
- **Signal-driven:** enable socket for SIGIO + install handler (`sigaction`). When datagram ready (data in kernel), kernel sends **SIGIO**; handler reads it. You are notified that I/O *can be initiated*.
- **I/O multiplexing:** you block in `select`/`poll` waiting for readiness, then read.
- **Difference:** multiplexing = you wait on a set of descriptors; signal-driven = kernel *interrupts* you with a signal when one is ready (no blocking in select). Both still do **synchronous** `recvfrom`.

### Q23 🔴★ I/O multiplexing & select(). (asked: NCIT 2025 Q4b)
Lets one process watch **many** descriptors and be told which are ready.
```c
int select(int maxfdp1, fd_set *readset, fd_set *writeset,
           fd_set *exceptset, const struct timeval *timeout);
```
- `timeout`: NULL (wait forever), fixed time, or 0 (poll).
- Macros: `FD_ZERO`, `FD_SET`, `FD_ISSET`, `FD_CLR`.
- Use: a client handling stdin + socket; a single-threaded server handling many sockets.

### Q24 🔴★ Mechanisms to handle multiple clients in UNIX — with code. (asked: Gandaki 2025 Q3b, NCIT 2025 Q5a)
Approaches: 1) **fork per client**, 2) **select/poll multiplexing**, 3) **threads (pthread)**.
**fork() concurrent server (example):**
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
Parent keeps calling `accept()`; each child handles its own client. Parent should `waitpid()`/`SIGCHLD` to reap zombies.

### Q25 🟡★ Broadcast vs multicast. (asked: NCIT 2025 Q4a)
- **Broadcast** (UDP only): sends to **all** hosts on the local subnet — must set `SO_BROADCAST`; routers don't forward.
- **Multicast**: sends to a **group** that "subscribed" (joined) — set `IP_ADD_MEMBERSHIP`, `IP_MULTICAST_TTL`; NIC hardware filtering; more efficient; used for video conferencing, etc.
**When needed:** broadcast = network-wide discovery/announce; multicast = group communication where only subscribers receive.

### Q26 🟡★ Socket options: setsockopt / getsockopt + SO_REUSEADDR, SO_BROADCAST, SO_KEEPALIVE, SO_LINGER. (asked: NCIT 2025 Q5b, Gandaki 2025 Q4b)
`setsockopt()`/`getsockopt()` get/set options at various levels (SOL_SOCKET, IPPROTO_TCP…).
```c
int setsockopt(int s, int level, int optname, const void *optval, socklen_t optlen);
int getsockopt(int s, int level, int optname, void *optval, socklen_t *optlen);
```
- **SO_REUSEADDR** — reuse a local address/port (avoids "address already in use" during TIME_WAIT); lets a server restart quickly.
- **SO_BROADCAST** — allow sending to broadcast addresses (UDP).
- **SO_KEEPALIVE** — TCP sends keep-alive probes after 2 h idle to check if peer is still alive.
- **SO_LINGER** — control `close()` behaviour: default/immediate, abort (RST, discard data), or linger until data acked.

### Q27 🟡 SO_LINGER in detail.
```c
struct linger { int l_onoff; int l_linger; };
```
1. `l_onoff=0` → default: `close()` returns immediately.
2. `l_onoff≠0, l_linger=0` → TCP **aborts** (RST), discards unsent data.
3. `l_onoff≠0, l_linger≠0` → `close()` **blocks/lingers** until data is sent & acked or timeout expires.

### Q28 🟢 SO_KEEPALIVE in detail.
After 2 h idle, send a probe. ACK → wait another 2 h. RST → peer crashed (`ECONNRESET`). No response → 8 probes, 75 s apart (~11 min), then `ETIMEDOUT` (or `EHOSTUNREACH` if ICMP error).

### Q29 🟡★ Syslog & logging from network applications. (asked: NCIT 2025 Q4b — with block diagram)
A **centralised logging facility** so daemons/network apps can report errors without a terminal.
```
             ┌─────────────────────────┐
 kernel log ─┤                         │    /etc/syslog.conf
 syslog(3) ──┤      syslogd (daemon)   ├──▶  classify → write to /var/log files
 UDP port 514─┤                         │
             └─────────────────────────┘
```
Functions: `openlog(ident, options, facility)` → `syslog(priority, format, ...)` → `closelog()`. Priorities: DEBUG, INFO, WARNING, ERROR, CRIT. Logs help debugging/auditing/monitoring.

### Q30 🟡★ How to secure a network application. (asked: short note "Wrapper function…")
- **By hostname/domain** — allow only trusted names (DNS can be spoofed).
- **By IP number** — restrict by source IP.
- **Wrapper program** — a front-end that checks the client against a policy before handing over to the real service (TCP wrappers style). Plus **TLS/SSL** for encryption/auth/integrity.

---

## Units 4 & 5 — Winsock

### Q31 🔴★ How is Winsock different from UNIX sockets? + static vs dynamic linking. (asked: NCIT 2025 Q6a — 7 marks)
**Winsock = Windows implementation of BSD sockets.**
| Unix | Winsock |
|---|---|
| socket = `int` | socket = `SOCKET` handle |
| `close(fd)` | `closesocket(s)` |
| `errno` | `WSAGetLastError()` |
| no init | **must WSAStartup/WSACleanup** |
| include `<sys/socket.h>` | include `<winsock2.h>`, link `ws2_32.lib` |
| add I/O models | adds message/event/overlapped/IOCP I/O models |

**Static vs dynamic linking:**
- **Dynamic (DLL):** code in a shared `.dll`, loaded at run time. ✓ smaller, easy update (replace DLL), modular. ✗ dependency on the DLL being present/version-correct (program may fail to start).
- **Static:** code copied into the `.exe`. ✓ no external dependency, always runs. ✗ bigger file, needs re-link to update.

### Q32 🔴★ WSAStartup / WSACleanup; role of setup(), cleanup(). (asked: NCIT Q6b, Gandaki Q5a)
- `WSAStartup(MAKEWORD(2,2), &wsadata)` — **loads** the Winsock DLL and requests a version; `wsadata` returns the loaded version. Called **once before any socket call**.
- `WSACleanup()` — **unloads** the library and frees resources; must match each `WSAStartup` (reference counted).
- *(“setup()/cleanup()” in the paper = these same two functions: setup = WSAStartup, cleanup = WSACleanup.)*

### Q33 🔴★ Major DLLs needed for a Winsock app. (asked: NCIT Q6b — 8 marks)
| DLL | Role |
|---|---|
| `ws2_32.dll` | **Main Winsock 2.0 32-bit API** (the primary one you link) |
| `wsock32.dll` | Winsock 1.1 32-bit API |
| `winsock.dll` | Winsock 1.1 16-bit API |
| `mswsock.dll` | Microsoft extensions (AcceptEx, TransmitFile,…) |
| `wshtcpip.dll` | TCP/IP helper |
| `msafd.dll` | Winsock ↔ kernel interface |

### Q34 🔴★ Winsock TCP & UDP client-server sequences with code. (asked: Gandaki Q5b — 8 marks)
**TCP server:** `WSAStartup → socket → bind → listen → accept → recv/send → closesocket → WSACleanup`
```c
WSADATA w;  WSAStartup(MAKEWORD(2,2), &w);
SOCKET s = socket(AF_INET, SOCK_STREAM, 0);
SOCKADDR_IN sa; sa.sin_family=AF_INET; sa.sin_port=htons(5150);
sa.sin_addr.s_addr=htonl(INADDR_ANY);
bind(s, (SOCKADDR*)&sa, sizeof(sa));
listen(s, 5);
SOCKADDR_IN cli; int clen=sizeof(cli);
SOCKET cs = accept(s, (SOCKADDR*)&cli, &clen);
/* recv(cs,buf,..), send(cs,buf,..) */
closesocket(cs); closesocket(s); WSACleanup();
```
**TCP client:** `WSAStartup → socket → connect → send/recv → closesocket → WSACleanup`.
**UDP:** server = `WSAStartup→socket→bind→recvfrom→closesocket` (no listen/accept); client = `WSAStartup→socket→sendto→closesocket`.

### Q35 🔴★ What is overlapped I/O in Winsock? How does it support async? (asked: NCIT 2025 Q7a — 7 marks)
**Overlapped I/O** = issue **multiple I/O operations at once** and let the kernel run them in the background; you're told when each **completes** (so the thread isn't blocked).
- Create the socket overlapped: `WSASocket(..., WSA_FLAG_OVERLAPPED)`.
- Use `WSASend, WSARecv, WSARecvFrom, WSAIoctl, AcceptEx`.
- Pass a `WSAOVERLAPPED` structure; completion via **event object** (in `Overlapped.hEvent`) **or completion routine** (callback).
- If a call returns `SOCKET_ERROR` + `WSA_IO_PENDING`, the op was **queued** (not an error) — completion will come later.
- Advantage: best throughput; one thread manages many outstanding I/O ops.

### Q36 🟡★ Event-driven programming & WSAEventSelect. (asked: NCIT Q7a alt, Gandaki Q6b)
**Event-driven programming** = the flow of the program is controlled by **events** (input, I/O readiness, messages) rather than a linear sequence; a loop dispatches work when events occur.
**WSAEventSelect** supports it by associating a socket with an **event object**:
- `WSACreateEvent()` → create event.
- `WSAEventSelect(s, hEvent, lNetworkEvents)` → "signal hEvent when these socket events happen".
- `WSAWaitForMultipleEvents(...)` → wait/loop; when signaled, find which socket by `WSAEnumNetworkEvents`.
- No window needed (unlike WSAAsyncSelect). Max **64 events per thread**.

### Q37 🟡★ WSAAsyncSelect vs WSAEventSelect. (asked: NCIT alt)
- **WSAAsyncSelect** → notifications delivered as **Windows messages** to a **window procedure**; needs a window; socket becomes non-blocking.
- **WSAEventSelect** → notifications delivered by **setting an event object**; **no window needed**; suited to console/background apps.

### Q38 🟡★ WSAPoll vs select. (asked: Gandaki Q6b alt)
- `select` uses fixed **fd_set** (limited size, e.g. 64 on Windows) and overwrites the sets (must reset each time).
- **`WSAPoll`** uses an **array of `WSAPOLLFD`** structures → **no size limit**, and returns an **event bitmask** without destroying your descriptors. Easier/more scalable for many sockets (like Unix `poll`).

### Q39 🟡 Graceful close in Winsock.
`shutdown(s, SD_SEND)` tells peer "no more data" (sends TCP FIN), ensures sent data is delivered, then `closesocket(s)` fully releases.

### Q40 🟢 WSAEnumProtocols / WSAAccept / WSAConnect (Winsock extensions).
- `WSAEnumProtocols` — list installed protocols + capabilities (`dwServiceFlags1`).
- `WSAAccept` — accept with a **condition function** to reject/defer.
- `WSAConnect` — connect with caller/callee data + QoS.

### Q41 🟡★ 5 Winsock I/O models.
1. **select** — check fd_set readiness (cross-platform).
2. **WSAAsyncSelect** — Windows message to a window.
3. **WSAEventSelect** — event objects (≤64/thread).
4. **Overlapped I/O** — many I/O at once, event/callback completion (best performance).
5. **Completion port (IOCP)** — thread pool over a completion port; best for hundreds–thousands of sockets (NT/2000 only).

### Q42 🟡★ Is a common Unix+Windows app possible? How? (asked: NCIT Q5b alt)
Yes — keep the core in portable sockets and wrap OS differences:
```c
#ifdef _WIN32
  #include <winsock2.h>
  #define close closesocket
#else
  #include <sys/socket.h>
  #include <unistd.h>
#endif
/* call WSAStartup/WSACleanup only under _WIN32; use WSAGetLastError() on Windows,
   errno on Unix */
```

---

## Unit 6 — Utilities, Trends & Security

### Q43 🟡★ Name & describe network utilities. (asked: short notes — telnet, ipconfig/ifconfig, remote login, iperf, netstat)
- **ping** — ICMP reachability + RTT.
- **telnet** — remote terminal login / test TCP port.
- **ip / ifconfig** — configure/inspect interfaces (IP, MAC, netmask).
- **iperf** — throughput/bandwidth measurement between client & server.
- **netstat** — connections, routing table, listening ports, interface stats.
- **remote login (rlogin / ssh)** — log into a remote host (ssh = secure/encrypted).

### Q44 🔴★ HTTP vs WebSocket + simple server. (asked: NCIT Q7b — 8 marks)
| | HTTP | WebSocket |
|---|---|---|
| Communication | request–response | **full-duplex** |
| Connection | closed after response (unless keep-alive) | **persistent** |
| Overhead | repeated headers | small frames |
| Server push | no | **yes** |
| Use | web pages/APIs | chat, gaming, live dashboards |

**Simple WebSocket server (pseudo-code):**
```
socket() → bind() → listen()        /* normal TCP server */
conn = accept();                    /* client's Upgrade request arrives  */
read HTTP request; verify "Upgrade: websocket";
send "HTTP/1.1 101 Switching Protocols" + Sec-WebSocket-Accept;
loop { read frame from conn; broadcast to all clients; }   /* full-duplex */
```
Handshake: client sends `GET / Upgrade: websocket`, server replies **101 Switching Protocols**; after that both sides push at will.

### Q45 🟡★ What is gRPC? (short note)
High-performance open-source **RPC framework** (Google) built on **HTTP/2** using **Protocol Buffers** (binary serialization from `.proto` files). Language-agnostic, supports **bidirectional streaming**; used in microservices and mobile↔backend.

### Q46 🟡★ TLS/SSL + cryptography concepts. (short note — asked)
TLS/SSL = **encryption + authentication + integrity** layer between app and TCP.
- **Encryption** — plaintext→ciphertext (symmetric AES; asymmetric/RSA public-key).
- **Hashing** — one-way digest (SHA-256) for integrity checks.
- **Certificates** — bind public key to identity, signed by a **CA**, used to authenticate + exchange keys.
Handshake: negotiate ciphers → verify cert → agree session key → encrypted channel (`https://`). Use OpenSSL (`SSL_CTX`, `SSL_connect`, `SSL_read/write`).

### Q47 🟡★ What is SDN? Key advantages. (asked: NCIT Q6b — 8 marks)
SDN **separates the control plane (brain) from the data plane (muscle)**:
- **Control plane** → centralized **SDN controller** (software) decides routing/policy.
- **Data plane** → switches just forward per installed flow rules.
- **OpenFlow** = the protocol by which the controller programs switch **flow tables**.
**Advantages:** centralized control, programmability, agility/automation, better utilization, vendor independence.

### Q48 🟡★ OpenFlow, P4, Frenetic. (short notes — asked: Gandaki "P4 and frenetic programming")
- **OpenFlow:** standard protocol controller↔switch to install forwarding rules.
- **P4:** high-level DSL to program the **data plane** (what switches do).
- **Frenetic:** DSL to program the **controller** (compile policies into OpenFlow rules).

### Q49 🟡★ WebSockets short note. (asked: Gandaki short note)
Full-duplex, persistent, single-TCP messaging protocol; HTTP Upgrade handshake → 101; `ws://`/`wss://`; tiny frames; ideal for chat/gaming/dashboards/stock tickers/IoT.

### Q50 🟢 Daemonizing techniques in Unix. (asked: Gandaki short note)
`fork()` (parent exits) → `setsid()` → `chdir("/")` → `umask(0)` → redirect std fds to `/dev/null`. Result = detached background process with no controlling terminal.

---

## Full 6-Part 8-Mark Model Answers (maximum-depth, exam-ready)

> These are complete model answers for the **highest-weight 8-mark questions**, each following the structure **①Definition → ②Diagram → ③Full concept → ④Example/code → ⑤Common errors/limits → ⑥Conclusion**. Practise writing these under exam conditions. (Answers to every other question are in the sections above.)

> **How to turn any model answer into a full-mark 8-mark answer (exam technique):**
> - **① Definition (≈1 mark):** open with *the* one-line definition in bold. Name the category it belongs to (protocol / function / model).
> - **② Diagram (≈2 marks):** every 8-mark answer MUST have a hand-drawn diagram — handshake, state machine, layered stack, call order, or comparison table. Label every arrow/box.
> - **③ Full concept (≈3 marks):** this is the body — expand with *why* and *how*, define the terms you use (every acronym: MSL, AM, ISN…), and give the rules/tables verbatim.
> - **④ Example/code (≈1 mark):** write a short real C snippet or a worked example (a port number, a specific protocol mapping). A code block scores even if small.
> - **⑤ Common errors/limits (≈1 mark):** this is what separates full marks from near-full marks — mention a real gotcha, a deprecated feature, or a limitation.
> - **⑥ Conclusion (≈0.5–1 mark):** 1–2 sentences: when to use it + what problems it solves. Don't repeat ①.
> - **Scaling rule:** for a genuine full 8-mark answer, write each section 1–2 sentences longer than shown here, and always draw the biggest, clearest diagram you can — diagrams are the cheapest marks.

---

### M1. Transport Protocols: Compare TCP, UDP, SCTP  [🔴★ NCIT 2025 Q1a]

**① Definition** — TCP, UDP and SCTP are all **transport-layer protocols** that deliver application data over IP. TCP = reliable byte-stream; UDP = best-effort datagram; SCTP = reliable, message-oriented with extra stream & path features for telephony.

**② Diagram**
```
APP (HTTP, FTP, ...)   APP (DNS, NFS, ...)   APP (SIGTRAN, ...)
       TCP                   UDP                  SCTP
                     ┌──────── IP ────────┐
                     └      (IPv4/IPv6)   ┘
```

**③ Full concept**
- **TCP**: connection-oriented; reliable (ACK + retransmission); ordered byte stream (no message boundaries); flow & congestion control.
- **UDP**: connectionless; unreliable (may lose/reorder/duplicate); preserves datagram boundaries; low latency, low overhead.
- **SCTP**: association (like TCP's connection); reliable; **message-oriented**; **multistreaming** (independent ordered streams inside one association, avoiding head-of-line blocking); **multi-homing** (several IPs per end for fault tolerance); **4-way handshake with a cookie** (anti-SYN-flood); no half-open state.

| Feature | TCP | UDP | SCTP |
|---|---|---|---|
| Connection | oriented | connectionless | oriented (association) |
| Reliability | reliable | unreliable | reliable |
| Ordering | byte stream | none | per-stream |
| Message boundaries | no | yes | yes |
| Multi-homing | no | no | yes |
| Handshake | 3-way | none | 4-way + cookie |

**④ Example** — TCP→HTTP/FTP; UDP→DNS/NFS/SNMP; SCTP→SIGTRAN (telephony/SS7), Diameter.

**⑤ Common errors/limits** — mixing them up: UDP is *not* always better for video (loss tolerated); SCTP's message orientation is why it isn't a drop-in replacement for raw TCP streams; UDP has no congestion control (can cause congestion collapse).

**⑥ Conclusion** — Choose TCP when reliability/order matter, UDP when low latency and loss-tolerance matter, SCTP when you need reliability + message boundaries + multi-stream/multi-home (telephony).

---

### M2. TCP Three-Way Handshake + ISN not zero  [🔴★ multiple papers]

**① Definition** — The three-way handshake establishes a TCP connection by **synchronising both sides' sequence numbers** before any data flows.

**② Diagram**
```
CLIENT (active)                        SERVER (passive)
  │① SYN(seq=x, SYN=1)                      │
  │───────────────────────────────▶         │  client→ LISTEN, sends SYN
  │② SYN+ACK(seq=y, ack=x+1)                │
  │◀───────────────────────────────         │  LISTEN→SYN_RCVD, sends SYN+ACK
  │③ ACK(ack=y+1)                           │
  │───────────────────────────────▶         │  SYN_RCVD→ESTABLISHED
  │          ✓ ESTABLISHED                  │
```

**③ Full concept** — (1) active opener sends SYN with ISN=x. (2) server replies SYN+ACK with its own ISN=y and ack=x+1 (a 1-byte sequence). (3) client sends ACK ack=y+1; both enter ESTABLISHED. Three segments are needed because each direction's sequence number must be acknowledged independently — it is full-duplex synchronisation. **ISN should not be 0:** an old delayed segment from a *previous* closed connection could fall inside the new connection's window and be accepted as valid data (the "wandering duplicate"); a predictable ISN is also a session-hijacking risk. Random/unpredictable ISN + TIME_WAIT (2×MSL) let old duplicates expire.

**④ Example/code** — `connect()` triggers the handshake; the server's `accept()` returns only after ESTABLISHED. In Wireshark you see the three packets SYN → SYN+ACK → ACK.

**⑤ Common errors/limits** — a **SYN flood** (attacker sends SYNs without completing) exhausts the *incomplete queue*; that's why `listen`'s backlog and SYN cookies exist (SCTP's cookie handshake resists this by design).

**⑥ Conclusion** — The 3-way handshake is TCP's way of reliably opening a full-duplex, sequence-synchronised connection; avoiding a guessable/fixed ISN and using TIME_WAIT prevent duplicate/corruption and hijacking attacks.

---

### M3. TCP State-Transition Diagram  [🔴★ NCIT 2025, Gandaki]

**① Definition** — TCP is a **finite state machine**; the state-transition diagram shows the **11 states** a connection passes through from creation to destruction (as seen in `netstat`/`ss`).

**② Diagram**
```
             (passive open=listen)
 CLOSED ─────────────────────────▶ LISTEN
   │ (active open=connect/send SYN)  │(recv SYN, send SYN+ACK)
   ▼                                 ▼
 SYN_SENT ──(recv SYN+ACK,send ACK)──▶  ESTABLISHED
   │ (simultaneous open: also reachable   │  ▲  ◀─ data transfer
   │  via SYN_SENT→SYN_RCVD→...)          │  │
   └────────────▶ SYN_RCVD ──(recv ACK)───┘  │
                                              │ active close (send FIN)
 ESTABLISHED ──(send FIN)──▶ FIN_WAIT_1 ◀──(recv FIN,send ACK)── CLOSE_WAIT
   │ ◀──(recv ACK)── enter         FIN_WAIT_2 ◀──(recv FIN,send ACK)── LAST_ACK
   │   FIN_WAIT_2                 │
   │ ◀──(recv FIN,send ACK)───────┘  (initiator)
   ▼
 TIME_WAIT (2×MSL) ──▶ CLOSED       (receiver) LAST_ACK ──(recv ACK)──▶ CLOSED
```

**③ Full concept** — The 11 states: `CLOSED, LISTEN, SYN_SENT, SYN_RCVD, ESTABLISHED, FIN_WAIT_1, FIN_WAIT_2, CLOSE_WAIT, CLOSING, LAST_ACK, TIME_WAIT`. Establishment: active `CLOSED→SYN_SENT→ESTABLISHED`, passive `CLOSED→LISTEN→SYN_RCVD→ESTABLISHED`. Termination: initiator `ESTABLISHED→FIN_WAIT_1→FIN_WAIT_2→TIME_WAIT→CLOSED`; receiver `ESTABLISHED→CLOSE_WAIT→LAST_ACK→CLOSED`. `CLOSING` occurs on simultaneous close. `TIME_WAIT` (2×MSL, where MSL=Maximum Segment Lifetime) lets lost final ACKs be retransmitted and lets old duplicates expire.

**④ Example** — `netstat -tan` shows sockets in `LISTEN`, `ESTABLISHED`, `TIME_WAIT`. A restarting server hitting "Address already in use" is because the old socket lingers in `TIME_WAIT` — fixed with `SO_REUSEADDR`.

**⑤ Common errors/limits** — forgetting that the **active closer** (not the receiver) enters `TIME_WAIT`; confusing `CLOSE_WAIT` (we received FIN, haven't closed) with `FIN_WAIT_2` (we sent FIN, waiting for theirs). `TIME_WAIT` is not a bug — it's required for safe port reuse.

**⑥ Conclusion** — Drawing the full state diagram (with both the client and server paths for open and close) is one of the highest-value, most reliable 8-mark answers in the course.

---

### M4. Value-Result Arguments  [🔴★ NCIT 2025 Q3a, Gandaki Q2b]

**① Definition** — A **value-result argument** is an argument that is a *value* when passed to a function but becomes a *result (output)* when the function returns. In sockets this applies to the **address length** in calls where the **kernel fills in** the address.

**② Diagram / comparison**
```
Process → kernel (bind/connect/sendto):        Kernel → process (accept/recvfrom/getsockname/getpeername):
   addr pointer + integer size                     addr pointer + pointer-to-size
   kernel only reads.                              length = VALUE on input (buffer size)
                                                   length = RESULT on output (bytes written)
```

**③ Full concept** — When the kernel writes a socket address *into your* structure, it must know (a) how big your buffer is (so it doesn't overflow it = value on input) and (b) you must learn how much was actually written (result on output). Hence the parameter is `socklen_t *` — a **pointer to** the length, treated both ways.

**④ Example/code**
```c
struct sockaddr_in cli;  socklen_t len;
len = sizeof(cli);               /* VALUE: "I have this much space"      */
getpeername(fd, (SA*)&cli, &len);/* on return len = RESULT: bytes stored */
```

**⑤ Common errors/limits** — For fixed-size structs (16-byte IPv4, 28-byte IPv6) the returned length is constant; for variable-size `sockaddr_un` it can be smaller. Forgetting to initialise `len = sizeof()` before the call is a classic bug (garbage length). The kernel **truncates** if `len` is too small.

**⑥ Conclusion** — The value-result length is how socket calls safely copy kernel-filled addresses into caller buffers while reporting the true size; it appears in all kernel→process socket functions.

---

### M5. Socket Address Structures  [🔴★ NCIT Q2b, Gandaki Q2a/Q2b]

**① Definition** — Socket address structures define **where a process can be reached** (address family + address + port) and are passed by pointer to socket functions.

**② Diagram**
```
   struct sockaddr  (generic, 16B) ── cast target
   struct sockaddr_in  (IPv4, 16B): sin_family|sin_port|sin_addr|sin_zero
   struct sockaddr_in6 (IPv6, 28B): sin6_family|sin6_port|sin6_flowinfo|sin6_addr|sin6_scope_id
   struct sockaddr_un  (AF_LOCAL):  sun_family | sun_path
   struct sockaddr_storage (≥128B, fits any + strict alignment)
```

**③ Full concept** — Each protocol family has its own struct named `sockaddr_<family>`. Everyone has `sin/sun_family`. Values (IP/port) must be in **network byte order**. `sockaddr` is only a generic container to **cast** specific pointers in calls like `bind()`. `sockaddr_storage` is large enough and aligned for **any** family, so you can use it when the family is unknown at compile time.

**④ Example/code**
```c
struct sockaddr_in serv;
memset(&serv, 0, sizeof(serv));
serv.sin_family = AF_INET;
serv.sin_port   = htons(8080);
serv.sin_addr.s_addr = htonl(INADDR_ANY);
bind(s, (struct sockaddr*)&serv, sizeof(serv));
```

**⑤ Common errors/limits** — Forgetting byte order (`htons`/`htonl`); not zeroing `sin_zero`; using the wrong size with `bind`; `sockaddr` is too small for IPv6 (use `sockaddr_storage`).

**⑥ Conclusion** — Address structures + byte-order + casting form the foundation every socket call relies on; mastering them is essential for both Unix and Winsock.

---

### M6. The 5 I/O Models  [🔴★ Gandaki Q4a]

**① Definition** — The five ways a process can wait for and perform I/O, differing in how the two phases (**waiting for data ready** and **copying kernel→process**) are handled.

**② Diagram**
```
Blocking     Nonblocking      Multiplexing   Signal-driven    Async
[recvfrom    [recvfrom        [select then   [SIGIO then     [aio_read
 blocks]      →EWOULDBLOCK    recvfrom]       recvfrom]       completion]
              poll loop]                                    ← truly async
───── all 4 are synchronous (block at the real recvfrom) ────▶
```

**③ Full concept**
1. **Blocking** — blocks until data ready *and* copied.
2. **Nonblocking** — returns `EWOULDBLOCK`; must poll (wastes CPU).
3. **Multiplexing (select/poll)** — block in `select`, then read the ready one.
4. **Signal-driven (SIGIO)** — kernel signals when *ready to read*; handler reads.
5. **Asynchronous (POSIX aio_)** — kernel notifies when *completely done* (datagram already copied). **Only model 5 is truly asynchronous**; models 1–4 block on the actual `recvfrom`, so all are synchronous.

**④ Example** — A chat server uses `select` (model 3) to watch many clients; `io_uring`/`aio_read` (model 5) for high-throughput.

**⑤ Common errors/limits** — Confusing "ready for I/O" (signal-driven) with "I/O done" (async); thinking nonblocking "fixes" blocking latency when it actually just wastes CPU unless paired with `select`.

**⑥ Conclusion** — All models share the same two phases; the choice is a trade-off between **throughput, CPU, and programming complexity**, with only async I/O running genuinely in the background.

---

### M7. Unix Concurrent Server (fork / select / threads)  [🔴★ Gandaki Q3b, NCIT Q5a]

**① Definition** — A **concurrent server** handles **many clients at once** (an iterative server handles one at a time), via processes, select-multiplexing, or threads.

**② Diagram**
```
fork():  MAIN──────────▶ accept() ──▶ fork() ── child handles client, parent accepts next
select(): MAIN: watch listenfd + all connfds in one select(); service whichever are ready
threads:  MAIN: accept() → pthread_create(handler, connfd)  (thread per client)
```

**③ Full concept** — (1) **fork** clones the process so the child serves the accepted socket while the parent continues accepting. (2) **select** lets one process wait on *many* descriptors and service only the ready ones. (3) **threads** share the address space and are lighter than processes; a thread pool avoids per-request creation cost. After fork both processes share the connected socket, so child closes the listener and parent closes the connected socket.

**④ Example/code**
```c
for (;;) {
  connfd = accept(listenfd, &cli, &clen);
  if ((pid = fork()) == 0) { close(listenfd); doit(connfd); close(connfd); exit(0); }
  close(connfd);          /* parent */
  /* handle SIGCHLD → waitpid(-1,...WNOHANG) to reap children */
}
```

**⑤ Common errors/limits** — **zombie children** if the parent never `wait()`s (fix with SIGCHLD + `waitpid(WNOHANG)`); resource exhaustion with thousands of processes; shared-data races with threads (need mutexes) and races between network event loops.

**⑥ Conclusion** — Choose fork for isolation, select for a single-process multiplexer, and threads/thread-pool when thousands of light connections and shared state are needed; always reap children and synchronise shared data.

---

### M8. Socket Options (getsockopt/setsockopt + SO_*)  [🔴★ NCIT Q5b, Gandaki Q4b]

**① Definition** — Socket options tune a socket's behaviour (reuse, buffer sizes, keepalive, lingering) via `getsockopt`/`setsockopt` at different levels (`SOL_SOCKET`, `IPPROTO_TCP`).

**② Diagram**
```
 app ──setsockopt(sock, SOL_SOCKET, SO_KEEPALIVE, on)──▶　kernel socket
 app ◀──getsockopt(sock, ...)──────────────────────────　reads current value
```

**③ Full concept**
- **SO_REUSEADDR**: allow rebinding an address/port held by a socket in **TIME_WAIT** → servers restart instantly.
- **SO_BROADCAST**: permit sending to broadcast addresses (UDP only).
- **SO_KEEPALIVE**: TCP probes an idle peer after 2 h; detects dead peers.
- **SO_LINGER**: controls `close()` — immediate / abort (RST, discard data) / linger until ACKed.
- **SO_RCVBUF/SO_SNDBUF**: kernel buffer sizes (set **before** connect/listen).

**④ Example/code**
```c
int on = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
struct linger li = {1, 5};  /* linger up to 5s */
setsockopt(s, SOL_SOCKET, SO_LINGER, &li, sizeof(li));
```

**⑤ Common errors/limits** — `SO_REUSEADDR` on Linux lets two sockets bind the same port only with `SO_REUSEPORT` (and only if neither is listening in a broken way); `SO_KEEPALIVE` default is 2 h (long); `SO_LINGER` with timeout blocks — a risk on shutdown. Set buffers before connect/listen.

**⑥ Conclusion** — These options solve practical server problems (restart, dead-peer detection, graceful/abrupt close) and are a staple 8 & 5-mark question.

---

### M9. WinSock vs UNIX + DLLs + static/dynamic linking  [🔴★ NCIT Q6a]

**① Definition** — WinSock (Windows Sockets) is the **Windows implementation of the BSD/Berkeley socket API**, with Windows-specific setup, types, error handling, and I/O models.

**② Diagram**
```
UNIX: kernel syscalls      Windows: DLL library (ws2_32.dll)
ABI: socket() ...           app calls Winsock API which calls ws2_32.dll
no init                     must WSAStartup first / WSACleanup last
```

**③ Full concept** — Differences: type `SOCKET` (handle) vs `int`; `closesocket` vs `close`; `WSAGetLastError` vs `errno`; mandatory `WSAStartup`/`WSACleanup`; link `ws2_32.lib`; extra async I/O models (WSAAsyncSelect, WSAEventSelect, overlapped, IOCP).

| Feature | Unix / Berkeley | WinSock |
|---|---|---|
| Socket type | `int` (file descriptor) | `SOCKET` (handle) |
| Close | `close(fd)` | `closesocket(s)` |
| Errors | `errno` | `WSAGetLastError()` |
| Init needed? | no | **WSAStartup / WSACleanup** |
| Include | `<sys/socket.h>` | `<winsock2.h>` + link `ws2_32.lib` |
| Byte order | `htons/htonl/...` | same |
| Extra I/O models | select, poll, epoll | WSAAsyncSelect, WSAEventSelect, overlapped, IOCP |

**Dynamic vs static linking** — `Dynamic (DLL)`: code in a shared `.dll`, loaded at run time → ✓ small, easy update (replace the DLL), modular; ✗ dependency on presence/version (may fail at start). **Static**: code copied into the `.exe` → ✓ self-contained, always runs; ✗ bigger, needs re-link to update.

**④ Example** — The main Winsock DLLs: `ws2_32.dll` (Winsock 2.0), `wsock32.dll` (1.1 32-bit), `winsock.dll` (1.1 16-bit), `mswsock.dll` (extensions), `wshtcpip.dll`, `msafd.dll`.

**⑤ Common errors/limits** — Forgetting `WSAStartup` (functions return WSAStartup-not-called / WSAENETDOWN); mixing `close`/`closesocket`; not matching WSACleanup counts (reference counting).

**⑥ Conclusion** — WinSock reuses the Berkeley model but adapts it to Windows (DLLs, handles, setup, and richer I/O models); the two are interoperable in concept, so Unix code ports with small `#ifdef` wrappers.

---

### M10. WinSock TCP & UDP client-server with code  [🔴★ Gandaki Q5b]

**① Definition** — WinSock client/server applications use the familiar sequence wrapped by `WSAStartup`/`WSACleanup`.

**② Diagram / call order**
```
TCP SERVER: WSAStartup → socket(AF_INET,SOCK_STREAM) → bind → listen → accept → recv/send → closesocket → WSACleanup
TCP CLIENT: WSAStartup → socket → connect → send/recv → closesocket → WSACleanup
UDP SERVER: WSAStartup → socket(AF_INET,SOCK_DGRAM) → bind → recvfrom → closesocket → WSACleanup
UDP CLIENT: WSAStartup → socket → sendto → closesocket → WSACleanup
```

**③ Full concept** — Same as Unix: TCP needs `listen`+`accept` on the server and `connect` on the client (bind is optional for clients); UDP uses `bind`+`recvfrom` (receiver) and `sendto` (sender) with no listen/accept/connect. Errors come from `WSAGetLastError()`.

**④ Example/code** — see Q34 above (full TCP server/client snippet).

**⑤ Common errors/limits** — no `bind` on a UDP receiver; calling `listen` without `bind` (WSAEINVAL); blocking `recv` stalling the server (use non-blocking/select/events); port already in use (WSAEADDRINUSE).

**⑥ Conclusion** — The Winsock sequences mirror Berkeley exactly except for the mandatory `WSAStartup`/`WSACleanup`; memorise both TCP and UDP orders for credits in the WinSock 8-mark question.

---

### M11. Overlapped I/O in WinSock  [🔴★ NCIT Q7a]

**① Definition** — Overlapped I/O lets a program **initiate multiple I/O operations at once** and be notified when each **completes**, so the thread is **not blocked** waiting — WinSock's highest-performance single-socket model.

**② Diagram**
```
 app: WSARecv(s, buf, ..., &ovl, completion_routine)  ──▶ returns (not blocked)
      WSARecv(s, ..., &ovl2, ...)  (another)
          kernel performs both in background
      each finishes → signals event (ovl.hEvent) or calls completion routine
      your thread was free to do other work meanwhile
```

**③ Full concept** — Socket must be opened overlapped: `WSASocket(..., WSA_FLAG_OVERLAPPED)`. Use `WSARecv/WSASend/WSARecvFrom/WSASendTo/WSAIoctl/AcceptEx`. A `WSAOVERLAPPED` struct identifies each op. Completion via (1) an **event object** in `ovl.hEvent` waited with `WSAWaitForMultipleEvents`, or (2) a **completion routine** called by the thread in alertable wait. If a call returns `SOCKET_ERROR` with `WSA_IO_PENDING`, the op is **queued, not an error**.

**④ Example/code**
```c
WSAOVERLAPPED ovl; WSARecv(s, &wsabuf, 1, &nrecv, 0, &ovl, completionRoutine);
/* if WSAGetLastError()==WSA_IO_PENDING -> will complete asynchronously */
```

**⑤ Common errors/limits** — treating `WSA_IO_PENDING` as failure; reusing an `OVERLAPPED`/buffer before completion (data race); not keeping buffers alive across the async operation.

**⑥ Conclusion** — Overlapped I/O is WinSock's way to run many non-blocking I/O ops concurrently with callbacks/events; it is the foundation of completion-port (IOCP) high-scale servers.

---

### M12. HTTP vs WebSocket + simple server  [🔴★ NCIT Q7b]

**① Definition** — **HTTP** is a request–response protocol; **WebSocket** upgrades one HTTP connection into a **persistent, full-duplex** channel where either side can push at any time.

**② Diagram**
```
CLIENT : GET /chat HTTP/1.1   Upgrade: websocket   Sec-WebSocket-Key: xxx
        ───────────────────────────────────────────────────────▶ SERVER
CLIENT ◀──── HTTP/1.1 101 Switching Protocols + Sec-WebSocket-Accept ──
        │   (now both sides exchange small frames freely)             │
        ◀──── push─────  ...  ─────push─────▶
```

**③ Full concept**
| | HTTP | WebSocket |
|---|---|---|
| Direction | request–response | full-duplex |
| Connection | closed after response (unless keep-alive) | persistent |
| Overhead | headers repeat | tiny frames |
| Server push | no | yes |
| Use | pages, APIs | chat, gaming, live dash |

After the 101 upgrade, data travels in tiny **frames**: a header byte with **FIN** (final frame), **opcode** (0x1 text, 0x2 binary, 0x8 close, 0x9 ping, 0xA pong, 0x0 continuation), a **MASK** bit (must be 1 client→server) and a payload length (7 bits; 126→2-byte, 127→8-byte length). **Ping/pong** keep idle connections alive and detect dead peers. Close frames carry a **close code** (1000 normal, 1001 going away, 1008 policy, 1011 server error).

**④ Simple server (pseudo)**
```
socket→bind→listen()           # normal TCP server
conn = accept(); read HTTP request
verify header: Upgrade: websocket
send "HTTP/1.1 101 Switching Protocols" + Sec-WebSocket-Accept
loop { read frame (opcode → handle text/close/ping);
       broadcast loopback to all clients (echo) }   # full-duplex
```

**⑤ Common errors/limits** — app-level ping/pong needed to keep idle connections alive; masking is mandatory client→server; concurrently serving many WebSocket connections requires poll/async on the server side.

**⑥ Conclusion** — WebSocket replaces HTTP's request/response with a persistent two-way channel, enabling real-time push apps; the 101-upgrade handshake is the key transition.

---

### M13. SDN: concept + advantages  [🔴★ NCIT Q6b, Gandaki]

**① Definition** — **SDN (Software-Defined Networking)** separates the **control plane** (deciding where traffic goes) from the **data/forwarding plane** (moving packets), centralising the "brain" in software.

**② Diagram**
```
                ┌──────────────────────────────┐
                │     SDN CONTROLLER (brain)   │  ← programmatic, centralized
                └──────────────┬───────────────┘
                               │ OpenFlow (install flow rules)
        ┌──────────┬───────────┴──────┬─────────┐
       switch1    switch2          switch3   ...  (data plane: just forward)
```

**③ Full concept** — Traditional routers/switches bundle both planes in closed firmware. SDN **moves control logic to a central controller** (with a network-wide view) and leaves switches as simple programmable forwarders. **OpenFlow** is the standard controller↔switch protocol that installs **flow rules** ("if packet matches X → forward to Y"). Benefits: **centralized control, programmability, agility/automation, better utilization, vendor independence**.

**④ Example** — A controller can push a new path/load-balancing rule to all switches in seconds without configuring each device; datacenter & campus networks use this for dynamic traffic engineering.

**⑤ Common errors/limits** — single point of failure (controller — needs high availability); controller-switch latency for very fast flow setup; matching/rule table size limits in switches.

**⑥ Conclusion** — SDN decouples intelligence from hardware to make networks programmable, agile and centrally manageable; OpenFlow is its core enabler.

---

### M14. TLS/SSL  [🔴 8-mark, in addition to the 🟡 short-note]

**① Definition** — **TLS** (Transport Layer Security, successor of SSL) is a security protocol layered between the **application** and **TCP** that provides the three security goals — **confidentiality** (encryption), **authenticity** (server/client identity) and **integrity** (tamper detection) — for all data in transit. Plain sockets send data **in the clear**; TLS wraps that data so only ciphertext reaches the network.

**② Diagram**
```
 Application (HTTP/WebSocket)   ← plaintext
        ▼
      TLS layer                  ← encrypt + authenticate + integrity
        ▼
        TCP → network            ← ciphertext only
(HTTPS = HTTP over TLS; WSS = WebSocket over TLS; FTPS = FTP over TLS)

Handshake (simplified):
 CLIENT                                   SERVER
  │ ClientHello(version, ciphers, rand)        │
  │ ───────────────────────────────────────▶    │
  │ ServerHello(chosen cipher, rand);          │
  │ Certificate; (KeyExchange/ServerKeyExch)   │
  │ ◀───────────────────────────────────────    │
  │ [verify cert via CA]                       │
  │ derive same session key                   │
  │ ── (both now send traffic with this key) ──│
```

**③ Full concept** — Three building blocks:
1. **Encryption** — **symmetric** (same key both ways, e.g. **AES**) is fast and used for bulk data; **asymmetric/public-key** (e.g. **RSA**) is slow but used to safely establish the symmetric **session key**; ephemeral **Diffie-Hellman** keys give **forward secrecy** (even if the private key leaks later, past traffic stays secret).
2. **Hashing** — **one-way** functions (**SHA-256**) produce a fixed-size digest used with an **HMAC** to verify **integrity** (detect tampering / bit flips).
3. **Certificates** — bind a **public key to an identity**; a certificate is **digitally signed by a Certificate Authority (CA)**. The client checks the signature and name against its **trusted-CA store** → this authenticates the server and safely delivers its public key.

**The TLS 1.2 handshake (detail):**
1. Client **ClientHello** — TLS version + list of **cipher suites** + client random.
2. Server **ServerHello** — chosen suite + server random; then sends its **certificate** and key-exchange material.
3. Client **verifies** the certificate (CA signature, hostname, validity period) — this authenticates the server.
4. Both sides independently **derive the same session key** (from RSA/DH key exchange + both randoms).
5. A brief **Finished** exchange double-checks agreement; from then on every record is encrypted + MAC'd.
- **TLS 1.3** streamlines this (1 round trip, no RSA key transport, only forward-secret key exchange).

**④ Example/code** — OpenSSL on a socket:
```c
SSL_CTX *ctx = SSL_CTX_new(TLS_server_method());     /* or client_method */
SSL_CTX_use_certificate_file(ctx, "cert.pem", SSL_FILETYPE_PEM);
SSL_CTX_use_PrivateKey_file(ctx, "key.pem",  SSL_FILETYPE_PEM);
SSL *ssl = SSL_new(ctx);
SSL_set_fd(ssl, sockfd);          /* attach to an existing connected socket */
SSL_accept(ssl);                  /* server handshake */
SSL_write(ssl, "hello", 5);       /* encrypted I/O instead of write()/read() */
SSL_read(ssl, buf, sizeof(buf));
```

**⑤ Common errors/limits** — certificate **expired / name mismatch / not a trusted CA** → handshake fails or browser warns; on **non-blocking** sockets `SSL_read`/`SSL_write` return `SSL_ERROR_WANT_READ`/`SSL_ERROR_WANT_WRITE` (retry later — *not* a hard error); **SSL/TLS 1.0/1.1 are deprecated/unwanted** on modern servers; no certificate ⇒ **no authentication** (attackers can MITM); RSA key-transport gives **no forward secrecy**.

**⑥ Conclusion** — TLS is the standard answer to "how do I make my sockets secure": it combines symmetric+asymmetric encryption, hashing and CA-signed certificates to give confidentiality, authenticity and integrity. As a network programmer you don't reinvent it — you attach a library like OpenSSL to your socket and swap `read`/`write` for `SSL_read`/`SSL_write`.

---

### M15. gRPC  [🔴 8-mark, in addition to the 🟡 short-note]

**① Definition** — **gRPC** (Google Remote Procedure Call) is a high-performance, open-source **RPC framework** letting a client program call a **method on a server** over the network **as if it were a local function call**. It is built on **HTTP/2** and serialises data with **Protocol Buffers (protobuf)**.

**② Diagram**
```
 .proto file (interface contract)
      │  protoc (compiler)
      ▼
  ┌──────────┐                 ┌──────────┐
  │ Client   │   HTTP/2        │ Server   │
  │ stub     │◀──────────────▶ │ stub     │
  └──────────┘  (binary        └──────────┘
   call f()      protobuf)       dispatch to
                                actual service

  The 4 call models:
  Unary:  req ─▶ │ ─▶ resp   (1:1)
  Server stream: req ─▶ │ ─▶ resp, resp, resp…   (1:N)
  Client stream: req,req… ─▶ │ ─▶ resp            (N:1)
  Bidi stream:   req,req… ◀─▶ │ ◀─▶ resp,resp…    (N:N)
```

**③ Full concept** — You first write an **interface contract** in a `.proto` file (messages + service). The **protoc** compiler generates **stub code in many languages** (C++, Java, Go, Python, C#, Node). The **HTTP/2** base gives **multiplexing** (many calls share one connection), header compression, and true **bidirectional streaming**. **Protobuf** gives compact, fast, **strongly typed** binary messages (far smaller than JSON/XML). Because stubs are generated for both sides, calling `stub.SayHello(...)` sends the RPC and delivers the reply as a typed value. gRPC also supports **deadlines/timeouts, cancellation, authentication (TLS/mTLS), load balancing, and interceptors**.

**④ Example/code** — `greeter.proto`:
```proto
syntax = "proto3";
service Greeter {                       // the interface contract
  rpc SayHello (HelloRequest) returns (HelloReply);
  rpc Chat    (stream ChatMsg) returns (stream ChatMsg);   // bidi streaming
}
message HelloRequest { string name = 1; }
message HelloReply   { string message = 1; }
```
```bash
protoc --cpp_out=. greeter.proto    # → greeter.grpc.pb.cc/h (stubs)
```
```c++ // client, produced code
Greeter::Stub stub(channel);            // channel = gRPC connection to "host:port"
HelloReply reply;
grpc::ClientContext ctx;
stub.SayHello(&ctx, req, &reply);       // looks like a normal function call
```

**⑤ Common errors/limits** — payloads are **binary, not human-readable ⇒ harder to debug** (needs grpcurl/Wireshark); the protobuf **toolchain must be installed** and versions kept matched across languages; relies on **HTTP/2** (proxies that don't support HTTP/2 break it); heavier than a hand-written TCP protocol for trivial cases; debugging/curl tooling is less mature than for REST.

**⑥ Conclusion** — gRPC is the modern default for **microservices, mobile↔backend and streaming** workloads that need performance, strong typing and multi-language support — it replaces hand-written JSON/REST with a fast, contract-driven, streaming RPC system over HTTP/2.

### M16. WebSockets short note (6-part micro answer / also see M12)
**① Definition** — WebSocket is a **full-duplex, persistent** messaging protocol over a single TCP connection, giving the server the ability to **push** to clients.
**② Handshake** — client sends `GET / … Upgrade: websocket` + `Sec-WebSocket-Key`; server replies **HTTP/1.1 101 Switching Protocols** + `Sec-WebSocket-Accept`; after this either side sends frames at will (`ws://`; `wss://` = over TLS).
**③ Concept** — frames: FIN + opcode (text/binary/close/ping/pong/continuation) + MASK + length; low overhead vs repeated HTTP headers; ping/pong keep it alive.
**④ Use** — chat, gaming, live dashboards, stock tickers, IoT push.
**⑤ Limits** — idle connections may be dropped (need ping/pong); server must handle many concurrent connections (select/poll/async).
**⑥ Conclusion** — WebSocket enables low-latency two-way push that plain HTTP cannot, making it the standard for real-time web apps.

---

### M17. Blocking vs Non-blocking I/O  [🔴★ Gandaki Q4a]

**① Definition** — Blocking and non-blocking are two ways a socket handles an operation that cannot complete immediately (e.g., `recv` when no data has arrived).

**② Diagram**
```
BLOCKING:   recvfrom ──▶ [sleep until data ready & copied] ──▶ return data
NONBLOCK:   recvfrom ──▶ EWOULDBLOCK (immediately) ──▶ (poll/retry) ──▶ data
```

**③ Full concept** — In **blocking** mode the call does not return until the operation completes; the process sleeps (thread is tied up, simple to write). In **non-blocking** mode the call returns **immediately** with `EWOULDBLOCK` (`WSAEWOULDBLOCK` on Windows) if it would block; the application must **poll** (repeatedly retry) or combine with `select`/events. Polling without select **wastes CPU** but lets one thread manage many sockets responsively.

**④ Example/code** (UNIX)
```c
int flags = fcntl(sock, F_GETFL, 0);
fcntl(sock, F_SETFL, flags | O_NONBLOCK);   /* make it non-blocking */
n = recv(sock, buf, len, 0);
if (n == -1 && errno == EWOULDBLOCK) /* no data yet; retry later */
```
(Winsock: `ioctlsocket(s, FIONBIO, &mode)` with `mode=1`.)

**⑤ Common errors/limits** — thinking non-blocking "fixes" latency — it doesn't; it just stops blocking (you must poll or select, or you can miss events). Confusing `EWOULDBLOCK` with a real error. `recv` can also return 0 = orderly close (different from -1/EWOULDBLOCK).

**⑥ Conclusion** — Blocking is simple but ties up a thread; non-blocking enables one thread to serve many sockets at the cost of polling — combine with `select`/`poll`/events for efficiency.

---

### M18. Signal-driven I/O vs I/O Multiplexing  [🔴★ Gandaki Q4a, NCIT alt]

**① Definition** — Two of the five I/O models used to learn when a descriptor is ready: **multiplexing** waits on a set of descriptors in `select`/`poll`; **signal-driven I/O** gets interrupted by the **SIGIO** signal when a descriptor is ready.

**② Diagram**
```
MULTIPLEXING:   select(fds) ──▶ [block in select] ──▶ "fd ready" ──▶ recvfrom
SIGNAL-DRIVEN:  sigaction(SIGIO) → main loop runs free → SIGIO delivered → handler reads
```

**③ Full concept** — I/O **multiplexing**: the process blocks in `select`/`poll` watching *many* descriptors; when one becomes readable it returns and the process reads it (still two syscalls, synchronous). **Signal-driven**: enable the socket for SIGIO (`fcntl(F_SETOWN)` + `O_ASYNC`), install a handler; the kernel sends **SIGIO** when the descriptor is *ready to read*; the handler (or notified main loop) does the `recvfrom`. Key: signal-driven means you are notified that I/O **can be initiated**; the actual `recvfrom` still blocks briefly, so **both are synchronous** (only POSIX async `aio_*` is truly asynchronous where the kernel signals *completion*).

**④ Example** — multiplexing is used by single-threaded servers watching many clients; signal-driven is good when you also want to do other work in the main loop and only be interrupted when I/O is possible.

**⑤ Common errors/limits** — SIGIO can be lost / needs re-arming and careful `EINTR` handling; multiplexing repeatedly rebuilds fd_sets and has size limits (FD_SETSIZE); on most systems only **one** SIGIO per descriptor is delivered at a time while the handler runs.

**⑥ Conclusion** — both are synchronous models that "tell you when to read"; multiplexing scales to many descriptors with `select`, signal-driven avoids polling but requires careful signal handling.

---

### M19. Daemonizing a Process (with code)  [🔴★ NCIT Q4a, Gandaki short note]

**① Definition** — A **daemon** is a long-running background process with **no controlling terminal**; daemonizing means detaching a program from the terminal, working directory, umask and std fds so it survives logout and runs until shutdown.

**② Diagram**
```
fork ─▶ parent exits
   ▼  child = orphan
setsid() ─▶ new session, no controlling terminal
   ▼
(optional 2nd fork) ─▶ can never re-grab a controlling TTY
   ▼
chdir("/")   umask(0)
   ▼
redirect stdin/stdout/stderr → /dev/null
   ▼
(optional) write pidfile → daemon running
```

**③ Full concept** — The steps fix real problems: `fork`+exit lets init adopt us and ensures we're not a session leader; `setsid()` creates a new session and detaches the controlling terminal; a second `fork()` stops us ever re-acquiring a TTY; `chdir("/")` avoids holding a mount point busy; `umask(0)` gives full control of file creation; redirecting fds 0/1/2 to `/dev/null` keeps stray output from crashing the daemon or writing to a terminal.

**④ Example/code**
```c
if ((pid = fork()) != 0) exit(0);   /* 1. parent exits */
setsid();                            /* 2. new session  */
/* 3. (optional 2nd fork) */
chdir("/");  umask(0);              /* 4,5 */
int fd = open("/dev/null", O_RDWR); /* 6. redirect std fds */
dup2(fd,0); dup2(fd,1); dup2(fd,2); if (fd>2) close(fd);
```
(Or use glibc's `daemon(0,0)` on Linux.)

**⑤ Common errors/limits** — forgetting `setsid()` (still has a terminal → killed on logout/hangup); closing fds but not re-pointing 0/1/2 (a later `printf` may crash); not handling `SIGPIPE`/`SIGHUP`. **systemd** modern approach: don't daemonize manually — run as a foreground child of systemd and let it manage cwd/umask/signals.

**⑥ Conclusion** — The fork→setsid→chdir→umask→redirect sequence produces a proper daemon detached from the terminal; on modern Linux, systemd handles this for you.

---

### M20. Socket Options: SO_LINGER, SO_KEEPALIVE, SO_REUSEADDR, SO_BROADCAST  [🔴★ NCIT Q5b, Gandaki Q4b]

**① Definition** — Socket options let you change how a socket behaves (port reuse, keepalive, close behaviour, broadcast) through `setsockopt`/`getsockopt`, and are read with `getsockopt`.

**② Diagram**
```
setsockopt(sock, SOL_SOCKET, SO_*, value)  ──▶  kernel socket (behaviour change)
getsockopt(sock, SOL_SOCKET, SO_*, buf)   ◀──  read current / pending error
```

**③ Full concept** —
- **SO_REUSEADDR**: allow re-binding a port/address held by a socket stuck in **TIME_WAIT**, so a server can restart immediately without "Address already in use".
- **SO_BROADCAST**: permit a UDP socket to send to broadcast addresses.
- **SO_KEEPALIVE**: TCP probes an idle peer after **2 hours** (probe → ACK = fine; RST = peer crashed `ECONNRESET`; no reply = 8 probes 75 s apart then `ETIMEDOUT`), detecting dead peers.
- **SO_LINGER**: controls `close()` (see table below), letting you choose graceful, abort, or wait-until-acked.

| l_onoff | l_linger | close() result |
|---|---|---|
| 0 | — | return immediately (default; data sent in background) |
| ≠0 | 0 | **abort**: discard send buffer, send RST |
| ≠0 | ≠0 | **linger/block** until data acked or timer expires |

**④ Example/code**
```c
int on = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
struct linger li = {1, 5};
setsockopt(s, SOL_SOCKET, SO_LINGER, &li, sizeof(li));
int keep = 1;
setsockopt(s, SOL_SOCKET, SO_KEEPALIVE, &keep, sizeof(keep));
```

**⑤ Common errors/limits** — `SO_LINGER` with a timeout **blocks** `close()` (risky at shutdown); `SO_REUSEADDR` semantics differ between BSD/Linux/Windows; `SO_KEEPALIVE`'s 2-hour default is too long for many apps; set `SO_RCVBUF`/`SO_SNDBUF` **before** connect/listen.

**⑥ Conclusion** — These four options are the most exam-relevant and solve real server problems (restart, dead-peer detection, graceful close, broadcasting); know their exact semantics and the `struct linger` table.

---

### M21. WSAAsyncSelect vs WSAEventSelect  [🟡★ NCIT alt]

**① Definition** — Two Winsock async (non-blocking) I/O models. **WSAAsyncSelect** delivers socket-event notifications as **Windows messages** to a window; **WSAEventSelect** signals an **event object** instead.

**② Diagram**
```
WSAAsyncSelect: socket event (FD_READ) ─▶ Winsock posts a WM_ message to a window's WndProc
WSAEventSelect: socket event ─▶ sets an event object (WSACreateEvent), wait with
                                 WSAWaitForMultipleEvents
```

**③ Full concept** — Both register interest in network events (`FD_READ, FD_WRITE, FD_OOB, FD_ACCEPT, FD_CONNECT, FD_CLOSE`) and both switch the socket to **non-blocking**.
- **WSAAsyncSelect** needs a **window handle** (`HWND`); the window procedure decodes `wParam`/`lParam`. Natural for GUI apps.
- **WSAEventSelect** uses a **`WSAEVENT`** object created by `WSACreateEvent`; `WSAEventSelect(s,hEvent,events)`; `WSAWaitForMultipleEvents(...)` waits (max **64 events per thread**); then `WSAEnumNetworkEvents` identifies which socket fired. No window needed ⇒ good for console/background apps and libraries.

**④ Example/code** (WSAEventSelect)
```c
WSAEVENT ev = WSACreateEvent();
WSAEventSelect(s, ev, FD_READ | FD_CLOSE);
WSAWaitForMultipleEvents(1, &ev, FALSE, WSA_INFINITE, FALSE);
/* then WSAEnumNetworkEvents(s, ev, &netEvents) to find which event */
```

**⑤ Common errors/limits** — WSAAsyncSelect requires a message pump/window → not usable in console apps; WSAEventSelect is limited to 64 events per thread (must build event-socket mapping yourself); both are still synchronous at the actual `recv` (only overlapped/IOCP give completion-based async).

**⑥ Conclusion** — Choose WSAAsyncSelect for GUI apps (message-based) and WSAEventSelect when no window exists; both tell you *when to do I/O*, not *that I/O finished*.

---

### M22. Securing a Network Application (hostname / IP / wrapper)  [🟡★ short note]

**① Definition** — Securing a network application means restricting who can connect and protecting the data, using **hostname checks**, **IP access control**, and **wrapper programs**, plus **TLS** for encryption.

**② Diagram**
```
CLIENT ──▶ [wrapper/access-check: hostname? IP? in hosts.allow?] ──allow──▶ real service
                                    │
                                    └── deny ──▶ reject / log
```

**③ Full concept** — (1) **By hostname/domain**: allowlist trusted names resolved via **DNS**; ⚠️ DNS can be **spoofed**, so this is a weak filter alone. (2) **By IP number**: restrict by **source IP** (`/etc/hosts.allow` + `hosts.deny`, or firewall rules); simple but IPs can be forged. (3) **Wrapper program** (e.g., **TCP wrappers / in.tcpd**): a small front-end that checks the client against a **policy** before launching the real service — enforces security **without modifying the server**. For data protection add **TLS/SSL** (encryption + authentication + integrity).

**④ Example** — `hosts.allow`: `sshd: 192.168.1.0/24` allows only that subnet; a `tcpd` wrapper relays allowed connections and drops others.

**⑤ Common errors/limits** — trusting hostname alone (spoofing); relying on source IP alone (forgery); forgetting that filtering is only access control — it does **not** encrypt data (pair with TLS).

**⑥ Conclusion** — A layered defence (hostname + IP allowlist + wrapper program + TLS) protects network services from both unwanted connections and in-transit tampering/eavesdropping.

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
