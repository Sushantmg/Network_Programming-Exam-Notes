# Unit 3 — Advanced Unix Network Programming

**Subject:** Network Programming (CMP 380) · **Unit 3** | **Priority: HIGH**

> This is the "meat" of the course — I/O models, concurrent servers, socket options, broadcast/multicast, security. Draw the diagrams and explain each so the examiner sees deep understanding.

---

## 3.1 The Five I/O Models (very frequent — draw the diagrams!)

### Every input operation has two distinct phases
1. **Waiting for data to be ready** — data arrives over the network into the **kernel's buffer** (received from the NIC/stack).
2. **Copying data** — moving it from the **kernel buffer into the process (application) buffer**.

These two phases are what distinguish the five models.

```
The five models:
1. Blocking I/O
2. Nonblocking I/O
3. I/O multiplexing  (select / poll / pselect)
4. Signal-driven I/O (SIGIO)
5. Asynchronous I/O  (POSIX aio_*)
```

### 1) Blocking I/O (default)
```
 Application                        Kernel
  recvfrom(sock, buf, ...)  ────▶   (blocked: waiting for a datagram)
                                    ... datagram arrives ...
                                    (copies datagram into app buffer)
  ◀──── return OK ─────────────     (copy complete)
  (process blocked the whole time)
```
- Sockets are **blocking by default**. The call **does not return until data both arrives AND is copied** into the app buffer.
- The process **sleeps (is blocked)** the entire wait; if the app has nothing else to do, this wastes the CPU only in the sense that it can't do other work, but it is the **simplest** and most common model.

### 2) Nonblocking I/O
```
 App: recvfrom ──▶ EWOULDBLOCK  (no datagram yet)
 App: recvfrom ──▶ EWOULDBLOCK  (no datagram yet)     ← "polling" loop
 App: recvfrom ──▶ datagram ready → copy → return OK
```
- The kernel returns **`EWOULDBLOCK`** instead of sleeping the process when no data is ready.
- The process must **poll** (keep calling `recvfrom`), which **wastes CPU** because the process burns cycles even while nothing is available, and there is **latency** between an event and when the poll notices it.
- Used mainly on systems **dedicated to a single function** where constant polling is acceptable.

### 3) I/O Multiplexing (select / poll / pselect)
```
 App: fd_set readfds; FD_ZERO(&readfds); FD_SET(sockfd,&readfds);
     select(sockfd+1, &readfds, ...)   ──▶  [kernel waits until sockfd readable]
     (process blocked in select, not recvfrom)
     datagram ready → select returns, saying sockfd is readable
 App: recvfrom(sock, buf, ...)  ──▶ copy → return OK
```
- The process asks the kernel to **watch many descriptors at once** and **tell us when any is ready**. Then it does the (blocking) `recvfrom`, which now returns immediately.
- Used when a process must wait on **multiple** descriptors:
  - a client handling **multiple descriptors** (stdin + a socket),
  - a client handling **multiple sockets** (a web client for several pages),
  - a TCP server handling **listening socket + connected sockets**,
  - a server handling **both TCP and UDP**,
  - a server handling multiple services/protocols.

### 4) Signal-driven I/O (SIGIO)
```
 App: sigaction(SIGIO, handler, ...);  install handler → return (nothing blocks)
     ... main loop keeps running, doing other work ...
 Kernel: datagram arrives → delivers SIGIO signal to the process
 App: the signal handler is invoked → calls recvfrom (or signals main loop)
```
- The kernel uses **`SIGIO`** to notify the process that the descriptor is *ready*.
- It requires setup: `fcntl(sock, F_SETOWN, getpid())` (which process to signal) and enabling O_ASYNC.
- **Advantage:** the main loop runs freely and is interrupted by the signal only when I/O is actually possible — no polling, no blocking in a select loop.

### 5) Asynchronous I/O (POSIX aio_*)
```
 App: aio_read(...)  ────▶  returns immediately (NOT blocked)
     ... main loop continues ...
 Kernel: the WHOLE read completes (data copied into app buffer) → delivers signal/SIGEV
 App: told "the I/O is DONE"
```
- The kernel performs the operation and **notifies when the whole operation has completed** (data already in the app buffer).
- **Crucial difference vs signal-driven:** signal-driven says **"you may *start* the I/O now"** (data ready but not copied); async says **"the I/O is *done*"** (data copied). In async, the kernel does both Phase 1 and Phase 2.

### Synchronous vs Asynchronous — the key exam point
- **Synchronous I/O** blocks the requesting process until the operation *completes*.
- **Asynchronous I/O** does **not** block.
- **The first four models are ALL synchronous** (each ends with a blocking `recvfrom`). **Only async I/O (`aio_*`) is truly asynchronous.**

```
Blocking          Nonblocking        Multiplexing       Signal-driven      Async
[recvfrom blocks] [recvfrom polls]   [select+recvfrom]  [SIGIO, then recv] [aio notifies on DONE]
 ─────────────── all four are synchronous ────────────────────▶  ◀──── truly asynchronous
```

### select vs poll vs epoll (modern depth)
- **`select`**: works with `fd_set` (fixed size, often 1024 via `FD_SETSIZE`); the kernel scans all fds each call → O(n) and limited.
- **`poll`**: no fixed size (`pollfd` array), still O(n) scan.
- **`epoll`** (Linux) / **`kqueue`** (BSD/macOS): event-driven, scalable to thousands of connections — an `epoll_create` returns an epoll object, `epoll_ctl` registers interest, `epoll_wait` returns only the *ready* fds (O(ready)). Many production servers use these.

---

## 3.2 `select()` in Detail

```c
#include <sys/select.h>
#include <sys/time.h>
int select(int maxfdp1, fd_set *readset, fd_set *writeset,
           fd_set *exceptset, const struct timeval *timeout);
```
- `maxfdp1` = **highest descriptor number + 1** in the sets (lets the kernel only scan up to that index).
- `readset` / `writeset` / `exceptset` = sets of descriptors to watch for **readable**, **writable**, and **exceptional** conditions. Any may be NULL.
- The `timeout`:
  - **NULL** → wait forever until a descriptor is ready (may be interrupted by a signal).
  - **fixed time value** → return when ready *or* when the timeout expires.
  - **both 0** → **polling**: check and return immediately (returns 0 if nothing ready).

### `fd_set` macros
```c
fd_set rset;
FD_ZERO(&rset);          /* clear all bits */
FD_SET(sockfd, &rset);   /* set sockfd's bit */
FD_CLR(sockfd, &rset);   /* clear sockfd's bit */
if (FD_ISSET(sockfd, &rset)) /* test whether sockfd is set */
```
- **`select` modifies the descriptor sets** — it clears the bits that are *not* ready. So you must **re-build the sets before each `select` call** in a loop.

### Return values
- Returns the **number of ready descriptors** (across all sets), **0** if the timeout expired, **-1** on error (or if interrupted by a signal — handle `EINTR`).

### Typical server loop with select
```c
for (;;) {
    FD_ZERO(&rset);
    FD_SET(listenfd, &rset);      /* watch the listening socket */
    for (i = 0; i < nclients; i++)
        FD_SET(client[i], &rset); /* watch all connected sockets */
    maxfd = (max of listenfd and all client fds) + 1;
    select(maxfd, &rset, NULL, NULL, NULL);
    if (FD_ISSET(listenfd, &rset)) { /* new connection */
        connfd = accept(listenfd, ...);
        store connfd in client[]; /* add to the watched set next loop */
    }
    for (i = 0; i < nclients; i++)
        if (FD_ISSET(client[i], &rset)) /* a client has data */
            handle(client[i]);
}
```

---

## 3.3 Concurrent Server Design (handling many clients)

### Why concurrency?
A **simple/iterative server** handles **one client at a time** — if that client is slow, everyone else waits. **Concurrent servers** serve **many clients at once**.

### Option A — `fork()` per client (most classic)
```
SERVER main:
  listenfd = socket(); bind(); listen();
  for (;;) {
      connfd = accept(listenfd, ...);      // wait for, then accept a client
      pid = fork();
      if (pid == 0) {                       // CHILD
          close(listenfd);                  // child doesn't need the listener
          serve_client(connfd);             // handle THIS one client
          close(connfd);
          exit(0);
      }
      close(connfd);                        // PARENT: doesn't need this connection
                                            // (loop and accept the next client)
  }
```
- After `fork()`, the **connected socket is shared** between parent and child (they hold the same descriptor). The child closes the listening socket copy, the parent closes the connected-socket copy — then they never interfere.
- The parent must handle `SIGCHLD` to **reap zombie children** (see Unit 2.9).

### Option B — `select()` many descriptors (single process)
- One process monitors **the listening socket + all connected sockets** in a single `select`. When any becomes readable, it services that one. It **never blocks on a single client**, so it serves many while still single-threaded.

### Option C — Threads (pthreads)
- **Thread per client**: `pthread_create` spawns a thread that handles a client; all threads **share the same address space and global variables**. Lighter and faster to create than processes, but needs **synchronization** (mutexes) to protect shared data.
- **Thread pool**: pre-create a fixed number of worker threads that take jobs from a queue — avoids the cost of creating a thread per request.

### Process vs Thread
| Process | Thread |
|---|---|
| **Separate** address space | **Shares** address space / global variables |
| Heavier (more overhead to create) | Lightweight, faster to create |
| Created with `fork`/`exec` | Created with `pthread_create` |
| Strong isolation (a crash is contained) | Easy sharing, but needs synchronization (mutex) |

---

## 3.4 Broadcast & Multicast (protocol-level depth)

### Unicast vs Broadcast vs Multicast
- **Unicast**: one sender → one receiver.
- **Broadcast**: one sender → **ALL** hosts on the **local network segment (LAN)**.
- **Multicast**: one sender → a **selected GROUP** of hosts that joined that group.

```
UNICAST        BROADCAST          MULTICAST
[1→1]          [1→ALL on LAN]     [1→ group members]
```

### Broadcast — details
- **UDP/datagram only** (TCP is point-to-point; broadcast is not allowed on stream sockets).
- Must set **`SO_BROADCAST`** before sending, or the kernel rejects it:
  ```c
  int on = 1;
  setsockopt(sockfd, SOL_SOCKET, SO_BROADCAST, &on, sizeof(on));
  ```
- To receive, a process must **bind the wildcard address** and its port.
- **Every machine's NIC on the LAN receives the frame**; each host's stack checks whether an application wants it. Because every host must process every broadcast, **heavy broadcast traffic slows the whole LAN** (the "broadcast storm" problem).
- **Routers generally do NOT forward broadcast packets** across networks (broadcast is limited to the local subnet; the special IPv4 broadcast address is 255.255.255.255 or subnet-directed x.x.x.255).
- Example: ARP requests, DHCP, NBNS.

### Multicast — details (video conferencing as a classic example)
- A host that wants to receive sender-whatever **joins a multicast group**; the NIC adds a **hardware filter** so only frames for joined groups are delivered.
- The sender **does not need to be a member** of the group — it just sends to the group's address.
- **IPv4 multicast addresses** are in **Class D**: `224.0.0.0 – 239.255.255.255`.
  - `224.0.0.x` are **link-local** (not routed), e.g., 224.0.0.1 = all hosts, 224.0.0.2 = all routers.
  - `224.0.1.x` and up are global scope.
- **TTL** controls scope: larger TTL lets multicast travel farther across routers.
- Joining a group (Unix): `struct ip_mreq { in_addr imr_multiaddr; in_addr imr_interface; }` then `setsockopt(sock, IPPROTO_IP, IP_ADD_MEMBERSHIP, ...)`.
- Applications: **video/audio conferencing**, streaming a live event to many, LAN gaming, network time.

---

## 3.5 Socket Options (very common — memorise this table)

There are **three ways** to affect a socket's behaviour:
1. **`getsockopt` / `setsockopt`**
2. **`fcntl`**
3. **`ioctl`**

```c
int getsockopt(int sockfd, int level, int optname, void *optval, socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);
```
- `level`: **`SOL_SOCKET`** (generic socket code) or protocol-specific, e.g. **`IPPROTO_TCP`**, `IPPROTO_IP`.
- Pass the option value by pointer and its size; for `getsockopt` the size is a **value-result**.

### Generic options (SOL_SOCKET)
| Option | Meaning / behaviour |
|---|---|
| **SO_REUSEADDR** | Allow **reuse** of a local address/port. Solves the **"Address already in use"** error a restarted server hits because the old connection is stuck in **TIME_WAIT**. |
| **SO_REUSEPORT** | Allow multiple sockets (processes) to bind the **same port** (load balancing). |
| **SO_BROADCAST** | Enable sending **broadcast** messages (UDP/datagram only). |
| **SO_KEEPALIVE** | Send keep-alive **probes** if the connection is idle (TCP; default 2 hours). |
| **SO_LINGER** | Control what `close()` does: normal / abort (RST) / linger-and-wait. |
| **SO_DEBUG** | (TCP) record detailed packet/event information. |
| **SO_DONTROUTE** | Bypass the normal routing table; send to the local interface. |
| **SO_ERROR** | **Read** a pending socket error (and clear it) — avoids the process being killed by the error. |
| **SO_RCVBUF / SO_SNDBUF** | Set the kernel's **receive / send buffer** sizes. **Very important:** on Linux you can only set these **up to twice** the default; you must set them **before `connect`/`listen`**. |
| **SO_RCVLOWAT / SO_SNDLOWAT** | Minimum number of bytes before `read`/`write` returns (low-water mark) — used by `select`. |
| **SO_TYPE** | Return the socket's type (SOCK_STREAM/SOCK_DGRAM) — useful after `exec` (`getpeername` with SO_TYPE). |

### SO_KEEPALIVE in detail
If the connection is idle for **2 hours**, TCP sends a **keep-alive probe**. Three outcomes:
1. Peer **ACKs** the probe → connection fine; **no notification**; probe again after another 2 h.
2. Peer sends **RST** → peer crashed/rebooted; socket returns error **`ECONNRESET`** and is closed.
3. **No response** → TCP sends **8 probes, 75 s apart** (~11 min 15 s total). If still no response → **`ETIMEDOUT`**; if an ICMP "host unreachable" arrives → **`EHOSTUNREACH`**.

### SO_LINGER in detail
```c
struct linger { int l_onoff; int l_linger; };
```
| `l_onoff` | `l_linger` | Result of `close()` |
|---|---|---|
| 0 | (ignored) | Return **immediately** (default). TCP will still try to send queued data in the background. |
| ≠0 | **0** | **Abort**: discard the send buffer and send **RST** (not a graceful FIN close). |
| ≠0 | ≠0 | **Linger**: `close()` **blocks** until queued data is sent and ACKed, or until `l_linger` time expires. |

### TCP-specific options (IPPROTO_TCP)
- **`TCP_NODELAY`** — disables **Nagle's algorithm**, so small segments are sent immediately. Used for **low-latency interactive** apps (SSH). (Nagle coalesces small writes while unacked data is in flight; this reduces packet count but adds latency to chatty apps.)
- **`TCP_MAXSEG`** — set/get the maximum segment size.

### Making sockets nonblocking / async with `fcntl`
```c
fcntl(sock, F_SETOWN, getpid());   /* set which process receives SIGIO/SIGURG */
fcntl(sock, F_SETFL, FASYNC);      /* enable SIGIO (signal-driven/async notify) */
fcntl(sock, F_SETFL, FNDELAY);     /* make the socket nonblocking → EWOULDBLOCK  */
/* modern/each applicable: */
flags = fcntl(sock, F_GETFL); fcntl(sock, F_SETFL, flags | O_NONBLOCK);  /* nonblocking */
```

---

## 3.6 Logging — Syslog

- **Syslog** = the UNIX **centralised logging facility** used by network/system applications.
- A special daemon (**`syslogd`** / `rsyslog`) collects messages from all programs and writes them to log files (often `/var/log/syslog`, `/var/log/messages`).

### Functions
```c
openlog(const char *ident, int logopt, int facility); /* identify the program */
syslog(int priority, const char *format, ...);        /* log a message (printf-like) */
closelog(void);                                       /* close the log */
```
- **`ident`** = a program-name string prepended to each message.
- **`priority`** = `facility` combined with a **level** (see below).

### Message priorities (highest → lowest)
`LOG_EMERG` (panic) → `LOG_ALERT` (action required now) → `LOG_CRIT` (critical) → `LOG_ERR` (error) → `LOG_WARNING` (warning) → `LOG_NOTICE` (normal but significant) → `LOG_INFO` (informational) → `LOG_DEBUG` (debug).
- Remote capability: with the right config, syslog can forward messages to another machine (central logging for many hosts) — that's why network daemons report to it.

---

## 3.7 Network Security Programming

### What is security?
Protecting data/resources/communication from **unauthorised access, alteration, or disruption**. Core goals (the **CIA triad** plus more): **Confidentiality**, **Integrity**, **Availability**, plus **Authenticity** and **Non-repudiation**.

### Common networking threats/challenges
- **Eavesdropping** — sniffing plaintext data on the wire.
- **Spoofing** — faking source IP or DNS (IP spoofing, DNS poisoning).
- **Man-in-the-middle (MITM)** — intercepting and altering traffic between two parties.
- **Replay attacks** — resending captured legitimate messages.
- **Protocol vulnerabilities** — attacks on the protocol itself (e.g., SYN flood).
- **Lack of authentication / authorisation** — no proof of who is connecting.
- **Buffer overflows** — malformed input crashes/compromises the daemon.

### Securing approaches (a layered "defence in depth")
1. **Securing by hostname/domain** — accept connections only from trusted names (validate via DNS / access lists). ⚠️ **DNS can be spoofed**, so this alone is not fully secure — but it's a first-tier filter.
2. **Identification by IP number** — restrict access by **source IP address** (access-control lists such as `/etc/hosts.allow` and `/etc/hosts.deny`, or firewall rules). Simple, but IPs can be forged.
3. **Wrapper program** — a **gatekeeper** process (e.g., **TCP wrappers / `in.tcpd`**) that:
   - receives the incoming connection first,
   - checks the client against a **security policy** (hosts.allow / hosts.deny, or custom rules),
   - only if allowed, **launches the real service** and relays data between it and the client; otherwise it **rejects** the connection.
   - This implements a simple **security policy** **without modifying the server** itself.

```
  Client ──▶ [ in.tcpd / wrapper ]  → checks policy (hosts.allow + hosts.deny)
                │ allowed?
                │ ──Yes──▶  authenticate/start real service → relay data
                └──No───▶   reject / log and drop the connection
```
- **Encryption (TLS/SSL)** is the deeper layer that protects data in transit (see Unit 6) — combining filtering (above) with encryption gives strong security.

---

## Unit 3 — Quick Memory Sheet
- **5 I/O models**: blocking, nonblocking, multiplexing (select/poll), signal-driven (SIGIO), async (aio). **First 4 synchronous; async is truly asynchronous.**
- `select(maxfdp1, read, write, except, timeout)`; sets are **modified** by select → rebuild each loop; `FD_ZERO/FD_SET/FD_ISSET/FD_CLR`.
- Concurrent servers: **fork-per-client**, **select** on many fds, or **threads** (thread-per-client / thread pool).
- Broadcast = **all on the LAN** (UDP only, must set `SO_BROADCAST`, not routed); Multicast = **group** (Class D 224.0.0.0–239.255.255.255, NIC filter, TTL controls scope).
- Options: **SO_REUSEADDR, SO_BROADCAST, SO_KEEPALIVE, SO_LINGER, SO_RCVBUF/SO_SNDBUF, SO_ERROR, SO_DONTROUTE, SO_DEBUG**; TCP: `TCP_NODELAY`.
- `fcntl(F_SETFL, FNDELAY/O_NONBLOCK)` = nonblocking; `FASYNC` + `F_SETOWN` = SIGIO.
- Syslog: `openlog / syslog / closelog`; priorities EMERG…DEBUG.
- Security: **hostname check, IP ACL, wrapper program** (in.tcpd) + TLS encryption.
