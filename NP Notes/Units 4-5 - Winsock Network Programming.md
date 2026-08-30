# Units 4 & 5 — Basics & Advanced Winsock (Windows Network Programming)

**Subject:** Network Programming (CMP 380) · **Units 4 & 5** · **Priority: HIGH**

> Winsock is *the same socket idea we saw in Unix, just tuned for Windows*. If you already understand `socket/bind/listen/accept/send/recv/close` from Unit 2, you already understand 80% of Winsock. Winsock adds two new things: **an initial setup step (WSAStartup)** and **Windows-style ways to do I/O** (messages, events, overlapped I/O, completion ports).

---

## 4.0 Background: NetBIOS / NetBEUI (short notes)

Windows had weak networking before; TCP/IP wasn't built in at first. Two old technologies matter:

- **NetBIOS** (Network Basic Input/Output System, IBM 1983) — an **API** (a set of function names) used for network communication, not a network protocol.
- **NetBEUI** (NetBIOS Extended User Interface, 1985) — the **protocol** that works with NetBIOS. **Not routable** — routers drop its packets, so it only works on one local LAN.
- **LANA numbers** — unique pairing of a network card with a transport protocol (0–9).
- **NetBIOS names** = 16 characters; **unique** (only one on the LAN) and **group** (shared / multicast).
- **nbtstat** = the utility to show NetBIOS name information.

Today this is background history; modern Windows uses Winsock over TCP/IP.

---

## 4.1 Introduction to Winsock (Windows Sockets)

### 4.1.1 What is Winsock, in one line?
**Winsock = the Windows version of the Berkeley (BSD) socket API.**

- Berkeley sockets were created for Unix. Windows wanted the **same programming model** so that:
  - Developers who knew sockets could write Windows apps easily.
  - Network software from different vendors could all work with the same standard interface.
- Winsock defines two interfaces:
  - **API** — used by **application developers** (the functions you call).
  - **SPI** (Service Provider Interface) — lets **network software vendors** plug in new protocol modules.

### 4.1.2 How is Winsock different from UNIX sockets? (VERY common exam question)
| Feature | Unix / Berkeley | Winsock |
|---|---|---|
| Socket type | `int` (a file descriptor) | `SOCKET` (a handle) |
| Close a socket | `close(fd)` | `closesocket(s)` |
| Error reporting | `errno` | `WSAGetLastError()` |
| Read/write | `read` / `write` / `send` / `recv` | `send` / `recv` (also `WSASend` / `WSARecv`) |
| Setup needed before use? | No | **Must call `WSAStartup` first, `WSACleanup` after** |
| Address structure | `struct sockaddr_in` | `SOCKADDR_IN` (same meaning) |
| Extra I/O models | select, etc. | Adds message/event/overlapped/completion-port |

**Why does Winsock need `WSAStartup`?** Windows needs to *load* the correct Winsock library (a DLL) and check the version before you use any socket function. Unix uses real system calls (`kernel`) so no setup call is needed.

**Why `WSAGetLastError()` instead of `errno`?** On Unix, `errno` is a shared variable updated by library functions. Windows' multithreaded/message model doesn't share it that way, so Winsock gives you a dedicated function to fetch the last error code.

### 4.1.3 Setting up the environment — WSAStartup / WSACleanup (HIGH priority, asked every year)

**`WSAStartup`** = "load the Winsock library and check its version."
```c
int WSAStartup(WORD wVersionRequested, LPWSADATA lpWSAData);
```
- **`wVersionRequested`** = the Winsock version you want, e.g. `MAKEWORD(2,2)` = Winsock **2.2**.
  - Winsock encodes a version as a 16-bit word: **low byte = major**, **high byte = minor**. So 1.1 = `MAKEWORD(1,1)`, 2.2 = `MAKEWORD(2,2)` (2 in the low/major byte, 2 in the high/minor byte).
- **`lpWSAData`** = pointer to a `WSADATA` structure, which on success tells you **which version was actually loaded**.

**`WSADATA` structure** (study output fields):
```c
typedef struct WSAData {
    WORD           wVersion;        /* version the library agreed to give you */
    WORD           wHighVersion;    /* highest Winsock version available     */
    char           szDescription[WSADESCRIPTION_LEN+1]; /* human description */
    char           szSystemStatus[WSASYS_STATUS_LEN+1];  /* system status     */
    unsigned short iMaxSockets;     /* (v1.1) max sockets; ignore in 2.x    */
    unsigned short iMaxUdpDg;       /* (v1.1) max UDP datagram size; 0 in 2.x*/
    char FAR      *lpVendorInfo;    /* vendor-specific string                */
} WSADATA;
```

**Version negotiation** (the "handshake" between app and library): you *request* a version with `wVersionRequested`. The DLL picks the version to give. The rule:
- If the library supports **≥ the version you asked for**, it returns `wVersion = wVersionRequested` and success (**0**).
- If the library supports **less** than you asked, it returns `WSASYSNOTREADY` / `WSAVERNOTSUPPORTED` — you must decide whether to accept the older `wVersion`.
- **Always check `WSADATA.wVersion`** after the call, and always handle a non-zero (error) return.

| `WSAStartup` return | Meaning |
|---|---|
| **0** | Success — library loaded, `wVersion` tells you what you got |
| **WSASYSNOTREADY** | Underlying network subsystem not ready |
| **WSAVERNOTSUPPORTED** | Requested version not available |
| **WSAEINPROGRESS** | A blocking WinSock operation already in progress |
| **WSAEPROCLIM** | Too many Winsock apps running |
| **WSAEFAULT** | Bad `lpWSAData` pointer |

**`WSACleanup`** = "unload the library and release its resources."
```c
int WSACleanup(void);
```
- Winsock uses **reference counting**: every successful `WSAStartup` increments the count, every `WSACleanup` decrements it. The library is fully unloaded only when the count reaches **0**.
- Every `WSAStartup` **must be matched by a `WSACleanup`** in your program; skipping it **leaks the library** and its resources.

```
First thing your program does:           Last thing before exiting:
  WSAStartup(MAKEWORD(2,2), &wsadata)       WSACleanup()
        │                                        ▲
        ▼                                        │
   ... use socket functions (socket,bind,listen,
       accept,connect,send,recv) ...
```

> The past papers also mention **"setup(), cleanup()"** — this is just another way of saying WSAStartup() (sets up) and WSACleanup() (cleans up).

### 4.1.4a Common Winsock error codes (via `WSAGetLastError()`)
| Code | What it means |
|---|---|
| **WSAEWOULDBLOCK** | Non-blocking socket: operation would block right now (try again later). |
| **WSAEADDRINUSE** | The address/port is already in use (bind failure). |
| **WSAECONNREFUSED** | Connection refused — no process listening (got RST). |
| **WSAECONNRESET** | Peer reset the connection (e.g., crashed / RST received). |
| **WSAETIMEDOUT** | Operation timed out. |
| **WSAENOTSOCK** | The descriptor is not a socket. |
| **WSAEMSGSIZE** | Datagram too large for the buffer (UDP). |
| **WSAECONNABORTED** | Connection aborted locally (software error/closed midway). |
| **WSAEHOSTUNREACH / WSAENETUNREACH** | Host / network unreachable. |
| **WSAEINTR** | Blocking call interrupted by `WSACancelBlockingCall`. |
| **WSAESHUTDOWN** | Socket already shut down (send after `shutdown(SD_SEND)`). |

### 4.1.4 Winsock DLLs — what they are and which are needed (HIGH, asked directly)

**A DLL** (Dynamic Link Library) is a library file (`.dll`) that is **loaded into memory only when a program needs it**. The executable does **not** copy the code in; it *calls into* the DLL at run time.

**Advantages of DLLs (dynamic linking):**
- Saves disk/memory (code is shared, loaded only when needed).
- Easy to **update/fix**: you replace the DLL file, no need to re-link the whole program.
- Promotes **modular** programs.

**Disadvantages of dynamic linking (vs static):**
- If a DLL is **missing, changed, or version-mismatched**, the program may fail to run ("dependency" problem).
- Slightly slower at startup (loading happens at run time).

**Static linking** = the code is copied into your `.exe` at compile time — bigger file, but no dependency on external DLLs, always runs.

**The major Winsock DLL files (memorise these):**
| DLL | What it provides |
|---|---|
| `ws2_32.dll` | **Main Winsock 2.0 32-bit API** — the one you link today |
| `wsock32.dll` | Winsock 1.1 32-bit API (older) |
| `winsock.dll` | Winsock 1.1 16-bit API (oldest) |
| `mswsock.dll` | Microsoft extensions (AcceptEx, TransmitFile, etc.) |
| `wshtcpip.dll` | Helper for TCP/IP |
| `msafd.dll` | Winsock interface to the kernel |

> To develop a modern Winsock app you **link `ws2_32.lib`** (which loads `ws2_32.dll`).

---

## 4.2 The Basic Winsock API

### The TCP/IP address structure — `SOCKADDR_IN` (HIGH)
```c
struct sockaddr_in {
    short         sin_family;   /* AF_INET                    */
    u_short       sin_port;     /* 16-bit port, network byte order */
    struct in_addr sin_addr;    /* 4-byte IPv4 address        */
    char          sin_zero[8];  /* padding                     */
};
```
- `inet_addr("1.2.3.4")` converts dotted-decimal → a 32-bit number (network order).
- `sockaddr_in` is an IPv4 address structure; **`sockaddr_in6`** is the IPv6 one (28 bytes, has `sin6_family`, `sin6_port`, `sin6_flowinfo`, `sin6_addr`, `sin6_scope_id`).

### Creating a socket
```c
SOCKET s = socket(AF_INET, SOCK_STREAM, 0);                    /* TCP */
SOCKET s = socket(AF_INET, SOCK_DGRAM, 0);                     /* UDP */
SOCKET s = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0,
                     WSA_FLAG_OVERLAPPED);                     /* overlapped */
```
- Socket types: `SOCK_STREAM`, `SOCK_DGRAM`, `SOCK_SEQPACKET`, `SOCK_RAW`, `SOCK_RDM`.

### The two call sequences (memorise — asked every year)
```
SERVER:  socket → bind → listen → accept → send/recv → closesocket
CLIENT:  socket → (resolve address) → connect → send/recv → closesocket
```
- **bind** — associate the socket with a local IP + port. Error: **WSAEADDRINUSE** (that address already in use).
- **listen** — mark the socket as listening, queue up to `backlog` pending connections. Error: **WSAEINVAL** if bind wasn't called first.
- **accept** — pull the **first pending connection** off the queue and return a **new socket** for it (the original stays listening).
- **connect** — (client) connect to a server; triggers the TCP three-way handshake.
- **send / recv** — send and receive data on a connected socket.
- **closesocket** — close the socket; further use gives **WSAENOTSOCK**.

> **KEY fact examined directly:** *"What happens if you call bind() in a TCP client?"*
> A TCP **client does not need to call bind()** — the kernel automatically assigns an **ephemeral (temporary) port** when `connect()` is called. If you call `bind()` in a client, you force a specific local IP/port, which:
> - may fail with **WSAEADDRINUSE** if that port is taken,
> - prevents the automatic ephemeral-port assignment.
> Rarely needed, except when a client must use a specific source port (e.g., some FTP modes).

### Name resolution (hostnames → addresses)
- **`gethostbyname(name)`** → `struct hostent *` with official name, aliases, address type, and `h_addr_list` (the addresses).
- **`getservbyname(name, proto)`** → `struct servent *` giving the **port** for a service (from the `services` file), e.g. `getservbyname("http","tcp")` → port 80.
- Async versions: `WSAAsyncGetHostByName`, `WSAAsyncGetServByName`.

### UDP (connectionless) in Winsock
- **Receiver**: `socket → bind` (no listen/accept) → `recvfrom`.
- **Sender**: `socket → sendto` (no connect needed).
- `recvfrom(s, buf, len, flags, from, &fromlen)` — `from` receives the **sender's** address.
- `sendto(s, buf, len, flags, to, tolen)` — `to` gives the **destination** address.

### Graceful close
- `shutdown(s, how)` with `SD_RECEIVE / SD_SEND / SD_BOTH` — tells the peer "no more data" (generates a TCP FIN on SD_SEND). Then `closesocket(s)`.

### Error handling
- `SOCKET_ERROR` = **-1** (common "it failed" return).
- `WSAGetLastError()` = the specific error code (e.g., WSAEADDRINUSE, WSAECONNREFUSED, WSAEMSGSIZE, WSAENOTSOCK, WSAEWOULDBLOCK).

---

## Unit 5 — Advanced Winsock: Asynchronous / Non-blocking I/O

### 5.1 The problem with ordinary (blocking) I/O
By default sockets are **blocking**: `recv()` will *wait (sleep)* until data arrives. In a server that must talk to **many clients**, one blocking socket would block everything.

```
BAD  (one blocking recv freezes the server):
  server: accept → recv(waits forever until client A sends)  ← can't serve B, C, D

GOOD (check readiness first / use events / overlapped):
  server watches ALL sockets; only reads/writes the ones that are ready
```

### Two socket modes in Winsock
- **Blocking mode (default)** — the call **does not return until the operation completes** (data arrives, send buffer frees up, etc.). Simple to write, but a single stalled socket stalls the whole thread.
- **Non-blocking mode** — the call returns **immediately**; if the operation can't complete it returns `SOCKET_ERROR` with **`WSAEWOULDBLOCK`**. You must then *check readiness* (select/events) or retry. This is the basis of all the async models below.

You switch modes with **`ioctlsocket`**:
```c
unsigned long mode = 1;                    /* 1 = non-blocking, 0 = blocking */
ioctlsocket(s, FIONBIO, &mode);
```
> Note: calling `WSAAsyncSelect` or `WSAEventSelect` for a socket **automatically puts it in non-blocking mode**; it does not return to blocking mode until you call `ioctlsocket(FIONBIO, 0)`.

### 5.2 The 5 Winsock I/O models (HIGH, asked every year)
| Model | Mechanism | Best for |
|---|---|---|
| **select** | check `fd_set` readability/writability | cross-platform, simple |
| **WSAAsyncSelect** | **Windows message** to a window | GUI apps, Win95/98 |
| **WSAEventSelect** | **event object** notification | Win95 (WS2)+ |
| **Overlapped I/O** | post many `WSASend/WSARecv`; get a completion notice | best performance |
| **Completion port (IOCP)** | thread pool over a completion port | hundreds/thousands of sockets (NT/2000 only) |

### The select model
Comes straight from Unix. Call `select(nfds, readfds, writefds, exceptfds, &timeout)` to ask: *"which of these sockets are ready to read/write?"* before doing real I/O. Shared by both OSs:
```c
select(0, &readfds, NULL, NULL, &tv);   /* wait until a socket in readfds is readable */
```
- On Windows `fd_set` is typically limited to **64 descriptors** and `select` **modifies** the sets (falls back to a "winsock 1.1" limit), so for larger numbers of sockets the event/overlapped models scale better.

### WSAAsyncSelect — Windows-message based (HIGH)
Meant for GUI (message-driven) Windows apps.
```c
WSAAsyncSelect(SOCKET s, HWND hWnd, u_int wMsg, long lEvent);
```
- The socket asks the **window `hWnd`** to receive a **Windows message `wMsg`** whenever one of the events in `lEvent` happens.
- Event flags: **FD_READ, FD_WRITE, FD_OOB, FD_ACCEPT, FD_CONNECT, FD_CLOSE**.
- Calling `WSAAsyncSelect` also automatically switches the socket to **non-blocking**.
- The window procedure decodes `wParam` (the socket) and `lParam` (the event) to handle it.

```
Socket event (e.g. FD_READ)  →  Winsock posts a message to the window  →  WindowProc handles it
   [socket]  ──────────────▶  [message queue]  ────────────────────────▶  your code
```

### WSAEventSelect — event-object based (HIGH)
Same idea but uses **event objects** instead of window messages — so **no window is needed** (good for console/background apps).
- `WSACreateEvent()` makes an event object.
- `WSAEventSelect(s, hEvent, lNetworkEvents)` ties the socket's events to that object.
- `WSAWaitForMultipleEvents(nEvents, lphEvents, fWaitAll, timeout, fAlertable)` waits (max **64** events per thread).
- `WSAResetEvent` / `WSACloseEvent` manage the event.

### Overlapped I/O (HIGH — asked directly)
The **fastest single-socket model**. You can issue **many I/O operations at once** and let them run in the background, then be told when each is done.
- Created with `WSASocket(..., WSA_FLAG_OVERLAPPED)`.
- Functions: `WSASend, WSASendTo, WSARecv, WSARecvFrom, WSAIoctl, AcceptEx, TransmitFile`.
- Uses a `WSAOVERLAPPED` structure.
- **Two completion methods:**
  1. **Event object** in `WSAOVERLAPPED.hEvent`, then `WSAWaitForMultipleEvents`.
  2. **Completion routine** — a callback function called when the operation finishes.
- Key error: if the call returns `SOCKET_ERROR` with `WSA_IO_PENDING`, it means the operation is **queued** and completion will come later — *not* a real error.

```
You post several reads ► kernel does them in background ► each completion
triggers a callback/event ► your thread is free meanwhile
```

### Completion port (IOCP) — the most powerful
`CreateIoCompletionPort(...)` creates a port and associates handles; a **pool of worker threads** services completed I/O. Best performance for **hundreds/thousands** of sockets. (NT/2000 only; hardest to write.)

### Distinguishing the three async models (VERY common question)
| | WSAAsyncSelect | WSAEventSelect | Overlapped |
|---|---|---|---|
| Notification | Windows **message** | **event object** | event object OR completion routine |
| Needs a window? | **Yes** | No | No |
| Best for | GUI apps | console/background, many sockets (≤64) | highest throughput |
| Multiple I/O at once | No | No | **Yes** |

### WSAPoll vs select (asked in Gandaki paper)
**`WSAPoll`** is Winsock's version of the Unix `poll()` function.
- `select` has a hard limit on how many descriptors it can watch (fd_set, e.g., 64 on Windows) and modifies the fd_sets (must reset each time).
- **`WSAPoll` uses an array of `WSAPOLLFD` structures** and returns an event bitmask — **no fd_set limit**, and it doesn't destroy the descriptors you pass. It's simpler for handling many sockets.

---

## 5.3 Winsock Extensions (how Winsock 2 goes beyond 1.1 / Berkeley)
- **WSAPoll** — poll-based multi-socket waiting (like Unix `poll`); array-based, no small fd_set limit.
- **WSACreateEvent / WSAResetEvent / WSASetEvent / WSACloseEvent / WSAWaitForMultipleEvents / WSAEventSelect** — the event-object object model.
- **WSAEnumProtocols** — enumerate installed protocols with their capabilities (`WSAPROTOCOL_INFO`).
- **WSAAccept** — accept with a **condition function** that can accept/reject/defer an incoming connection (lets you apply policy at accept time).
- **WSAConnect** — connect with caller/callee data + **QoS** (Quality of Service) parameters.
- **WSAJoinLeaf** / **WSADuplicateSocket** — join a leaf node in a multipoint/multicast session, and share sockets across processes.
- **WSARecvMsg / WSASendMsg** — richer receive/send with control info (header/ancillary data, for IPv6 and advanced protocols).
- **WSANSPIoctl / WSALookupServiceBegin / Next / End** — the Winsock **name resolution and service registration** API (replaces the old gethostbyname model).
- **mswsock extensions** (Microsoft-only, very high performance): **`AcceptEx`**, **`TransmitFile`**, **`ConnectEx`**, `GetAcceptExSockaddrs` — used heavily by IIS/web servers and overlapped servers.

## 5.4 Writing a cross-platform (Unix + Windows) app (HIGH, asked directly)
> *"Is it possible to write a common network application that runs in both UNIX and Windows?"* **Yes.**
- Wrap the Windows-only parts in `#ifdef _WIN32`.
- Use a `SOCKET`/`int` abstraction, and `closesocket()` vs `close()`.
- Replace `errno` with `WSAGetLastError()`.
- Only call `WSAStartup`/`WSACleanup` on Windows.

```c
#ifdef _WIN32
#include <winsock2.h>
#else
#include <sys/socket.h>
#include <unistd.h>
#define closesocket close
#endif

/* ... common socket code using send()/recv() works on both ... */
```

---

## Quick Revision Sheet
- Winsock = Windows implementation of BSD sockets. Init **WSAStartup**, cleanup **WSACleanup**.
- `SOCKET` (not int), `closesocket` (not close), `WSAGetLastError` (not errno).
- **Server**: socket→bind→listen→accept→send/recv→closesocket.
- **Client**: socket→connect→send/recv→closesocket.
- UDP: receiver=bind→recvfrom; sender=sendto.
- **5 I/O models**: select, WSAAsyncSelect, WSAEventSelect, overlapped, IOCP (increasing power/complexity).
- **WSAAsyncSelect** = messages to a window (GUI). **WSAEventSelect** = event objects (max 64/thread).
- **Overlapped** = many I/O at once, completion via event or callback; `WSA_IO_PENDING` = queued not error.
- **WSAPoll** = array-based, no fd_set limit (improves on select).
- **DLLs** = dynamically loaded libs (ws2_32.dll = main Winsock 2.0). Dynamic = easy update, dependency risk; static = self-contained, bigger.
- Graceful close = `shutdown` then `closesocket`.
- WinSock **history**: arose because early Windows lacked a standard TCP/IP API; agreed at Interop 1991.
