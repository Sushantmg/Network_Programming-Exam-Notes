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
