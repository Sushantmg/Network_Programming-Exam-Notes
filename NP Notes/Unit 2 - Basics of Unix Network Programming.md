# Unit 2 — Basics of Unix Network Programming

**Subject:** Network Programming (CMP 380) · **Unit 2** | **Priority: HIGH** (biggest unit, most weight)

> This is the biggest and most heavily-examined unit. It covers the actual BSD/Berkeley socket API — the foundation that Winsock copies. Read each section until you can explain it, and memorise the **call orders** and **address-structure layouts** exactly.

---

## 2.1 Unix Files & File Descriptors

### Everything is a file
In UNIX, **everything is a file** — regular files, directories, devices, sockets all use the same read/write model. All I/O is done by **reading and writing files** using the kernel.

### File descriptor (fd)
When a process opens a file, the kernel returns a small **non-negative integer** called a **file descriptor**. It is an index into the process's per-process open-file table.

### The 3 standard descriptors (always open when a process starts)
```
fd 0 = STDIN  (standard input, typically the keyboard)
fd 1 = STDOUT (standard output, typically the screen)
fd 2 = STDERR (standard error, typically the screen)
```

### Core I/O functions
```c
ssize_t read(int fd, void *buf, size_t nbytes);   /* returns bytes read, 0 = EOF, -1 = error */
ssize_t write(int fd, const void *buf, size_t nbytes); /* returns bytes written, -1 = error */
```
- The **return value** is important: a short read/write can happen; always check it.
- **-1** with the global variable **`errno`** set means an error. Use **`strerror(errno)`** to print the message.

### Common include files for socket programs (know what each gives you)
```c
#include <sys/types.h>      /* primitive system data types (pid_t, size_t, ...) */
#include <sys/socket.h>     /* socket(), bind(), listen(), accept(), ... addr structs */
#include <netinet/in.h>     /* struct sockaddr_in, INADDR_ANY, inet_* functions */
#include <arpa/inet.h>      /* inet_aton, inet_ntoa, inet_pton, inet_ntop */
#include <netdb.h>          /* gethostbyname, getservbyname, hostent, servent */
#include <unistd.h>         /* read, write, close, fork, exec, ... */
#include <signal.h>         /* signal(), sigaction(), signal names */
#include <fcntl.h>          /* file control: open, fcntl, non-blocking flags */
#include <errno.h>          /* errno variable and error values */
#include <sys/time.h>       /* struct timeval, select() */
#include <sys/wait.h>       /* wait(), waitpid(), macros to check child status */
```

### Data-transfer calls — which to use when
| Call | Used with | Notes |
|---|---|---|
| `read` / `write` | any connected socket | simplest, most efficient for streams |
| `recv` / `send` | connected stream sockets | extra **flags** argument |
| `recvfrom` / `sendto` | **datagram** (UDP) sockets | you specify/learn the peer address each time |
| `recvmsg` / `sendmsg` | specialized | **scatter/gather** + can pass **descriptors** (Unix domain) |
| `readv` / `writev` | file or socket | scatter/gather across multiple buffers |

Key **flags** for `recv`/`send`:
- **`MSG_OOB`** — send/receive **out-of-band** (urgent) data.
- **`MSG_PEEK`** — look at data without removing it from the receive queue.
- **`MSG_WAITALL`** — wait until the requested number of bytes arrive.

---

## 2.2 The Socket Layer Model (how the pieces fit)

```
   APPLICATION PROCESS
         │  ① BSD Socket layer  = generic socket functions (portable across apps)
         ▼
      ┌──────────────┐
      │  BSD Socket  │   e.g. socket(), bind()
      └──────────────┘
         │  ② INET Socket layer = endpoints for IP family (TCP/UDP)
         ▼
      ┌──────────────┐
      │  INET Socket │   (struct sock)
      └──────────────┘
         ┌────┴───┐        ③ transport
         ▼        ▼
      ┌─────┐  ┌─────┐
      │ TCP │  │ UDP │
      └─────┘  └─────┘
         └──┬────┘       ④ network layer
            ▼
         ┌──────┐
         │  IP  │        (ARP resolves IP→hardware)
         └──────┘
            ▼           ⑤ link/device layer
   (Network devices: Ethernet, SLIP, PLIP, ...)
```
- Communication is **two-way**: packets received go **up** through the same layers in reverse (Ethernet → IP → TCP/UDP → INET socket → BSD socket → application).
- This layered design is why a program written against **BSD sockets** is portable: the transport (TCP or UDP) is chosen at `socket()` time, and the same generic calls work.

---

## 2.3 Socket Address Structures (memorise — very frequent exam topic)

Every socket function takes a **pointer to a socket address structure**. Each protocol family has its own struct whose **name is `sockaddr_` + family suffix**.

### a) IPv4 — `struct sockaddr_in` (in `<netinet/in.h>`)
```c
struct sockaddr_in {
    uint8_t        sin_len;      /* length field = 16 (not in older/some Linux) */
    sa_family_t    sin_family;   /* AF_INET */
    in_port_t      sin_port;     /* 16-bit TCP/UDP port, NETWORK byte order */
    struct in_addr sin_addr;     /* 32-bit IPv4 address, NETWORK byte order */
    char           sin_zero[8];  /* unused, always zero to pad to 16 bytes */
};
struct in_addr { in_addr_t s_addr; };   /* the 32-bit IPv4 address */
```
- **Size = 16 bytes.** POSIX guarantees only three fields exist: `sin_family`, `sin_addr`, `sin_port`.
- **`sin_len` is optional** — it exists in 4.4BSD, but **Linux does not have it**. Portable code should not rely on it.
- **Always set `sin_port` and `sin_addr.s_addr` in network byte order** (see 2.5).

### b) Generic — `struct sockaddr` (in `<sys/socket.h>`)
```c
struct sockaddr {
    uint8_t     sa_len;      /* length */
    sa_family_t sa_family;   /* address family, e.g. AF_INET */
    char        sa_data[14]; /* protocol-specific address data */
};
```
- This is the **generic** structure that socket functions accept. You **cast** your specific structure to it: `(struct sockaddr *) &servaddr`.
- It exists so a generic call like `bind()` can accept any family's structure.

### c) IPv6 — `struct sockaddr_in6`
```c
struct sockaddr_in6 {
    uint8_t         sin6_len;      /* length = 28 */
    sa_family_t     sin6_family;   /* AF_INET6 */
    in_port_t       sin6_port;     /* port, network order */
    uint32_t        sin6_flowinfo; /* flow information */
    struct in6_addr sin6_addr;     /* 128-bit IPv6 address */
    uint32_t        sin6_scope_id; /* scope id for link-local */
};
```
- **Size = 28 bytes.** Unlike IPv4, `sin6_flowinfo` and `sin6_scope_id` are meaningful fields.

### d) Unix domain — `struct sockaddr_un` (in `<sys/un.h>`, family `AF_LOCAL`)
```c
struct sockaddr_un {
    sa_family_t sun_family;   /* AF_LOCAL */
    char        sun_path[104];/* null-terminated pathname */
};
```
- The pathname must be **null-terminated**; use the **`SUN_LEN`** macro to compute the correct length.

### e) Storage — `struct sockaddr_storage`
- A generic structure **large enough (and aligned) to hold any** address structure (IPv4 or IPv6). Use it when you don't know the family in advance (e.g., a server that serves both v4 and v6).

### How the address values are set — the "bind table"
| IP address | Port | Result |
|---|---|---|
| Wildcard (*) | 0 | kernel chooses IP **and** port |
| Wildcard (*) | non-zero | kernel chooses IP, process sets port |
| Local IP | 0 | process sets IP, kernel chooses port |
| Local IP | non-zero | process sets both |

- The IPv4 **wildcard address** is **`INADDR_ANY`** (value 0). Set it in network order:
  ```c
  sin_addr.s_addr = htonl(INADDR_ANY);
  ```

---

## 2.4 Value-Result Arguments (important concept — asked directly)

A socket address structure is passed **by reference** (a pointer) along with its **length**. The crucial subtlety: **the length is passed differently depending on the direction of data flow**.

### Case 1 — Process → Kernel (`bind`, `connect`, `sendto`, `sendmsg`)
The process knows the structure's size, so it passes a **pointer + an integer size**.
```c
struct sockaddr_in serv;
connect(sockfd, (SA *) &serv, sizeof(serv));   /* integer size, by value */
```

### Case 2 — Kernel → Process (`accept`, `recvfrom`, `getpeername`, `getsockname`)
The kernel **writes** the address into the caller's structure. The size is a **value** on input (how big my buffer is) but a **result** on output (how much the kernel actually wrote) ⇒ a **value-result argument**.
```c
struct sockaddr_in cli;
socklen_t len;
len = sizeof(cli);             /* len = value: "I have this much space" */
getpeername(fd, (SA *) &cli, &len);  /* on return, len = result: "this much was written" */
```
- For **fixed-size** structures (16-byte IPv4, 28-byte IPv6) the returned length is fixed.
- For **variable-size** ones (`sockaddr_un`, whose pathname varies) the length can be **less** than the max — it tells you exactly how much was used.
- On the **kernel→process** path the kernel **truncates** the address if your buffer is too small; on some systems the returned length is the untruncated value so you can detect truncation.

---

## 2.5 Byte Ordering & Manipulation Functions (frequent)

### Endianness
- **Little-endian**: the **low-order** byte is stored at the **starting** (lowest) address.
- **Big-endian**: the **high-order** byte is stored at the **starting** address.
- A machine's native order is its **host byte order**.
- **Network byte order = big-endian**, because the Internet protocols specify it. All multi-byte fields in protocol headers (ports, addresses, lengths) travel in network order.

```
16-bit value example, value = 0x1234:
LITTLE-ENDIAN:  address A = 0x34 (low-order),  address A+1 = 0x12 (high-order)
BIG-ENDIAN:     address A = 0x12 (high-order), address A+1 = 0x34 (low-order)
```

### Byte-order conversion functions (`<netinet/in.h>`)
```c
uint16_t htons(uint16_t v);   /* host → network SHORT (16-bit)  → socket port/shorts */
uint32_t htonl(uint32_t v);   /* host → network LONG  (32-bit)  → addresses */
uint16_t ntohs(uint16_t v);   /* network → host SHORT */
uint32_t ntohl(uint32_t v);   /* network → host LONG */
```
- `h`=host, `n`=network, `s`=short(16-bit), `l`=long(32-bit).
- **On a big-endian machine these are null macros (no-ops).** Using them keeps code portable across little- and big-endian hosts.
- **Always** use them when filling `sin_port` and `sin_addr.s_addr` (e.g., `htons(8080)`, `htonl(INADDR_ANY)`).

### Byte-manipulation functions (for multibyte fields, NOT C strings)
- **Berkeley/DERIVED** names (still common in socket books):
  - **`bzero(ptr, n)`** — write n zero bytes (useful to init address struct).
  - **`bcopy(src, dst, n)`** — copy n bytes.
  - **`bcmp(p1, p2, n)`** — compare n bytes.
- **ANSI C** equivalents (preferred today):
  - **`memset(ptr, 0, n)`**, **`memcpy(dst, src, n)`**, **`memcmp(p1, p2, n)`**.
- Important: use a **byte** function, **not `strcpy`/`strcmp`**, because address structures contain **binary** data (including zero bytes), not C strings. `memset(&addr, 0, sizeof(addr))` is the standard way to initialise an address structure.

### Address conversion functions
| Function | Converts | Notes |
|---|---|---|
| `inet_aton(str, &addr)` | dotted-decimal (e.g. "127.0.0.1") → 32-bit binary | **preferred** (robust) |
| `inet_addr(str)` | same, returns value | **deprecated** — 255.255.255.255 returns -1 (looks like error) |
| `inet_ntoa(addr)` | 32-bit binary → dotted-decimal string | |
| `inet_pton` / `inet_ntop` | both IPv4 & IPv6 | **modern**; p = presentation (text), n = numeric (binary); use with `AF_INET`/`AF_INET6` |

---

## 2.6 Hostname & Network Name Lookups

- **DNS** translates human-friendly names (e.g., `www.example.com`) to IP addresses (and back).
- **`gethostbyname(name)`** returns a pointer to **`struct hostent`**:
  ```c
  struct hostent {
      char  *h_name;       /* official (canonical) hostname */
      char **h_aliases;    /* array of alias names */
      int    h_addrtype;   /* address family (AF_INET) */
      int    h_length;     /* address length (4) */
      char **h_addr_list;  /* array of IP addresses (network order) */
  };
  #define h_addr h_addr_list[0]   /* first address, for backward compatibility */
  ```
- **`getservbyname(name, proto)`** returns service/port info (e.g., `"http"`, `"tcp"` → port 80) from the `services` database.
- Modern code prefers **`getaddrinfo()`** (works for both IPv4/IPv6 and returns a linked list) over `gethostbyname()`, but the latter is still the classic exam answer.

---

## 2.7 Elementary TCP & UDP Sockets (the heart of the course — MUST know call order)

### TCP Server call order
```
socket() → bind() → listen() → accept() ─▶ read()/write() → close()
                                          (process request)
```
### TCP Client call order
```
socket() → connect() → read()/write() → close()
   (bind() optional — kernel picks the ephemeral port)
```
### UDP (connectionless — no connect/listen/accept)
```
Server: socket() → bind() → recvfrom() → sendto() → close()
Client: socket() → sendto() → recvfrom() → close()
```

### `socket()` — create an endpoint
```c
#include <sys/socket.h>
int socket(int family, int type, int protocol);
```
- `family`: `AF_INET` (IPv4), `AF_INET6` (IPv6), `AF_LOCAL`/`AF_UNIX` (Unix domain), `AF_ROUTE`.
- `type`: `SOCK_STREAM` (TCP), `SOCK_DGRAM` (UDP), `SOCK_RAW` (raw IP), `SOCK_SEQPACKET` (SCTP).
- `protocol`: usually **0** (lets kernel pick from family+type), else `IPPROTO_TCP`, `IPPROTO_UDP`.
- Returns a **socket descriptor** (small non-negative int) or **-1** on error.
```c
s = socket(AF_INET, SOCK_STREAM, 0);   /* TCP socket */
s = socket(AF_INET, SOCK_DGRAM, 0);    /* UDP socket  */
```

### `connect()` — client establishes a connection (TCP)
```c
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
```
- Initiates the **three-way handshake**; returns when connected (or on error).
- Client **need not `bind()` first** — the kernel picks an ephemeral port automatically.
- **Common errors:**
  - **`ETIMEDOUT`** — SYN sent but no response (e.g., server host unreachable/dropped the SYN). TCP retries then times out.
  - **`ECONNREFUSED`** — an **RST** was returned, meaning **no process is listening** on that port.
  - **`EHOSTUNREACH` / `ENETUNREACH`** — host or network unreachable.
- **When is RST sent?** (i) a SYN arrives for a port with no listener, (ii) TCP aborts an existing connection, (iii) TCP receives a segment destined for a connection that doesn't exist.

### `bind()` — assign a local protocol address
```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```
- Maps the socket to a local **IP + TCP/UDP port**. Servers bind their **well-known port**.
- If you don't bind (or pass port 0), the **kernel chooses an ephemeral port** (for clients).
- Passing a **wildcard IP** (`INADDR_ANY`) means "accept on any local interface".

### `listen()` — server converts an active socket into a passive one
```c
int listen(int sockfd, int backlog);
```
- Does **two things**: (1) changes the socket type from active (client-oriented) to **passive** (listening); (2) sets the **maximum** number of queued connections (`backlog`).
- The kernel maintains **two queues** per listening socket:
  - **Incomplete queue** — connections where **SYN** was received but the handshake isn't finished.
  - **Completed queue** — connections whose **three-way handshake finished**, waiting for `accept()`.
- New TCP **SYN flood** protection was later added to limit the incomplete queue automatically.

```
Client                          Server
connect()  ───SYN J──▶   [incomplete queue]
           ◀──SYN+ACK──   (handshake in progress)
           ───ACK──▶   →  [completed queue] →  accept() returns a new fd
```

### `accept()` — server returns the next completed connection
```c
int accept(int sockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
```
- Returns the next connection from the **front of the completed queue**; **sleeps (blocks) if the queue is empty**.
- **Listening socket** (first argument) vs **connected socket** (return value): the returned descriptor is a *new* socket connected to that one client.
- Returns **3 values**: the connected descriptor (or -1), the client's protocol address, and its size.
- `cliaddr` and `addrlen` may be **NULL** if you don't need the client's address.

### `fork()` — create a new process (used for concurrent servers)
```c
#include <unistd.h>
pid_t fork(void);
```
- Called **once, returns twice**: in the **parent** it returns the child's **PID**; in the **child** it returns **0**; **-1** on failure.
- The two copies share initial memory then diverge; the child gets a copy of the parent's file descriptors (so the child inherits the accepted socket).
- **Two typical uses:** (1) a process copies itself so one copy handles one task (a concurrent server), (2) to run a different program — copy itself, then call `exec`.

### `exec()` — replace the process image (6 variants)
`execl, execv, execle, execve, execlp, execvp`. The **PID does not change** — only the memory image (program) is replaced. Only **`execve`** is a true system call; the other five are library wrappers that call it.
```
convert filename→path        build argv        system call
execlp(file,arg,...)  ────▶ execl(...)  ───────────────────┐
execvp(file,argv)     ────▶ execv(...)  ───────────────────┤
execl(path,arg,...)   ────────────────▶ execl(path,...) ───┼─▶ execve(path, argv, envp)
execv(path,argv)      ────────────────▶ execv(path,...) ───┘
execle(path,arg,...)  ─── add envp ──▶ execve(...)
```
- Differences among them: **filename vs full pathname** (the `p` variants search `PATH`); **list vs argv array** (`l` vs `v`); **explicit vs inherited environment** (`e` adds a custom `envp`).

### `close()` — close a socket / end a connection
```c
int close(int sockfd);
```
- Marks the socket closed. For a TCP socket, **any queued outgoing data is still sent**, then normal termination (FIN) proceeds.

### `getsockname()` / `getpeername()`
- Return the **local** / **foreign** protocol address (`addrlen` is a **value-result**).
- Uses: learn the ephemeral port after `connect`, after `bind(0)`, get the address family, and in an **`xinetd` child after `exec`** (peer identity would otherwise be lost, since `exec` keeps descriptors but discards variables).

---

## 2.8 Unix Domain Sockets (IPC on the same host)

- Not a real protocol suite — a way to do client/server **on a single host** using the same API as TCP/UDP, but **much faster** (no IP/TCP protocol processing, no routing). Classic use: the **X Window System**.
- Two types: **stream** (like TCP) and **datagram** (like UDP).
- Used to **pass file descriptors** between processes and so a server can learn the **client's UID/GID** (local security).

### `socketpair()`
```c
int socketpair(int family, int type, int protocol, int sockfd[2]);
```
- Creates **two connected sockets** on the same host. Family must be `AF_LOCAL`; type `SOCK_STREAM` or `SOCK_DGRAM`; protocol 0.
- For `SOCK_STREAM` this produces a **stream pipe** (full-duplex) — a Unix-domain equivalent of a full-duplex pipe.

### Passing descriptors (sendmsg / recvmsg)
1. Create a Unix domain socket (via `socketpair`, or `bind`+`connect`).
2. One process opens a descriptor (`open`, `accept`, ...) to pass.
3. The sender builds an **`msghdr`** whose ancillary data (`msg_control` / `CMSG_DATA`) carries the descriptor, and calls **`sendmsg`**.
4. The receiver calls **`recvmsg`** and extracts the descriptor (which may be **numbered differently** in the receiver) from ancillary data use. `MSG_CMSG_CLOEXEC`/`SCM_RIGHTS` handle passing.

---

## 2.9 Signal Handling in Unix (deep)

### What is a signal?
A **signal** = an **asynchronous notification** sent to a process telling it that an event occurred. Any process can install a **handler** for any signal (except a few like SIGKILL/SIGSTOP that cannot be caught).

### Common signals in socket programming
| Signal | Meaning / typical use |
|---|---|
| **SIGINT** | Interrupt from keyboard (Ctrl+C). |
| **SIGCHLD** | A child process stopped or terminated — used by servers to `wait()` on dead children. |
| **SIGALRM** | Alarm clock (timer) has gone off (used for timeouts). |
| **SIGPIPE** | Wrote to a socket whose peer has closed — **default action kills the process** (servers ignore it). |
| **SIGIO / SIGPOLL** | Socket/file is ready for I/O (used with O_ASYNC / poll-based I/O). |
| **SIGURG** | Out-of-band (urgent) data. |
| **SIGTERM** | Termination request (default graceful exit). |

### Installing a handler
```c
signal(SIGCHLD, sig_chld);            /* classic, less flexible */
sigaction(SIGCHLD, &act, NULL);       /* recommended: richer control */
```
**`sigaction` is preferred** over `signal` because it gives you:
- control over which signals are **blocked during the handler** (`sa_mask`),
- whether the call is auto-**restarted** (`SA_RESTART`) after the handler returns,
- whether **`SA_SIGINFO`** is set so you can get detailed info in `sa_sigaction`.

### EINTR — interrupted system call
If a slow system call (e.g., `accept`, `read`) is **interrupted by a signal**, it returns **-1 with `errno = EINTR`**. Without `SA_RESTART` you must manually handle `EINTR` (e.g., loop `accept()` again); with `SA_RESTART` the kernel retries the call for you.

### The "child zombie" problem for concurrent servers (important)
- When a child process dies, it becomes a **zombie** until its parent calls **`wait()`**.
- A server that `fork()`s many children MUST reap them, or it leaks zombies.
- Standard solution: handle **`SIGCHLD`**:
  ```c
  void sig_chld(int signo) { pid_t pid;
      while ((pid = waitpid(-1, NULL, WNOHANG)) > 0) ;   /* reap without blocking */
  }
  ```
  Using `waitpid(-1, ..., WNOHANG)` in a loop handles **multiple** children dying at once and never blocks the signal handler.

---

## 2.10 Daemon Processes (deep — with full code)

A **daemon** = a **long-running background process** with **no controlling terminal** and no user interaction. Examples: `syslogd`, `inetd`, `sshd`, `crond`.

### Why not just `&` (run in background)? Because that still
1. keeps the **controlling terminal** (sending SIGHUP on logout kills it),
2. keeps **stdin/stdout/stderr** connected to the terminal,
3. inherits the shell's **working directory** and **umask**.

### The standard daemonization steps (8-step "how to daemonize")
```c
// Step 1: fork so we're not the session leader
if ( (pid = fork()) < 0 ) err_sys("fork error");
else if (pid != 0) exit(0);            // parent (the shell) exits

// Step 2: new session, detach from controlling terminal
setsid();

// Step 3: (re-fork, optional but recommended) — see "double fork" below
// if ( (pid = fork()) < 0 ) err_sys("fork error");
// else if (pid != 0) exit(0);

// Step 4: change working directory so we don't hold a mount point busy
chdir("/");

// Step 5: clear the file-mode creation mask
umask(0);

// Step 6: close all inherited open file descriptors
//   (or at least close/reopen fd 0,1,2 to /dev/null)
...

// Step 7: redirect stdin, stdout, stderr to /dev/null
fd = open("/dev/null", O_RDWR);
dup2(fd, 0); dup2(fd, 1); dup2(fd, 2);
if (fd > 2) close(fd);

// Step 8: write our PID to a pidfile (e.g., /var/run/daemon.pid)
```

```
fork ─▶ parent exits ─▶ child is an orphan
   ▼
setsid()  ─▶ child becomes session leader, no controlling terminal
   ▼
(second fork) ─▶ guarantees we can never reacquire a controlling terminal
   ▼
chdir("/") , umask(0)
   ▼
close/reopen std fds to /dev/null
   ▼
(optional) write pidfile
   ▼
daemon running in background
```

### Why the "double fork"? (good depth to know)
- A session leader (which a single-`setsid` daemon is) can still **re-acquire a controlling terminal** if it later `open()`s a terminal device without `O_NOCTTY`.
- The **second `fork()`** makes the daemon **no longer a session leader** (only the *first* child is the leader; the *grandchild* is not), so it can never accidentally grab a controlling TTY again. This is what `daemon(3)` (glibc) and systemd-style daemons do.

### SysV vs BSD daemonising (older books mention it)
- **SysV** style: `fork → setsid → fork → chdir("/") → umask(0) → close` — the full sequence above.
- **BSD** style (older): `fork → chdir("/") → setsid` — fewer steps; on older BSD that was enough.

### Modern systems (systemd)
- Today daemons are often **supervised by systemd**, which: sets `O_NOCTTY`, sets `UMask` (default 022), sets `WorkingDirectory` (default `/`), and **ignores SIGPIPE/SIGHUP** — so the program doesn't need to daemonize itself; it just runs as a foreground child of systemd.

> **Typical exam prompt:** "Explain how to daemonize a process" or "Write the daemon function." Give the 8 steps above with the rationale for each (terminal, cwd, umask, std fds, pidfile).

---

## Unit 2 — Quick Memory Sheet
- fd: 0=stdin, 1=stdout, 2=stderr. `read`/`write` return bytes or -1 (`errno`).
- Address structs: `sockaddr` (generic cast), `sockaddr_in` (IPv4, 16B), `sockaddr_in6` (IPv6, 28B), `sockaddr_un` (Unix), `sockaddr_storage` (holds either). Set port/addr in **network byte order**.
- **Value-result argument**: kernel→process calls (`accept`, `recvfrom`, `getpeername`, `getsockname`) pass pointer+`&len`.
- Network byte order = **big-endian**. `htons/htonl/ntohs/ntohl`; use `memset`/`bzero`, not string functions, on binary structs.
- TCP server `socket→bind→listen→accept`; client `socket→connect`; UDP no connect/listen/accept.
- `listen` maintains **two queues** (incomplete + completed). `accept` returns new connected fd.
- `fork` returns twice; `exec` replaces image (only `execve` is a syscall; 6 variants).
- Unix domain = fast local IPC + descriptor passing (`socketpair`, `sendmsg`/`recvmsg` + `SCM_RIGHTS`).
- Signals: `sigaction` preferred; handle `EINTR`; **reap children with `waitpid` on `SIGCHLD`** (avoid zombies); ignore `SIGPIPE`.
- Daemon = `fork`→parent exit→`setsid`→(second `fork`)→`chdir("/")`→`umask(0)`→redirect std fds to `/dev/null`→(pidfile).
