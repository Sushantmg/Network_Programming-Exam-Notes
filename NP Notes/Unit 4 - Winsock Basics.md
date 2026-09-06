# Unit 4 — Winsock Basics (Windows Network Programming)

**Subject:** Network Programming (CMP 380) · **Unit 4** | **Priority: HIGH**
**Number of teaching hours in syllabus:** 6

> Winsock is *the same socket idea we saw in Unix, just tuned for Windows*. If you already understand `socket/bind/listen/accept/send/recv/close` from Unit 2, you already understand 80% of Winsock. Winsock adds two new things: **an initial setup step (WSAStartup)** and **Windows-style ways to do I/O** (Unit 5). **Draw the call sequence diagrams — they are the cheapest marks.**

---

## 4.0 Background: NetBIOS / NetBEUI (short notes)

Windows had weak networking before; TCP/IP wasn't built in at first. Two old technologies matter:

- **NetBIOS** (Network Basic Input/Output System, IBM 1983) — an **API** (a set of function names) used for network communication, not a network protocol.
- **NetBEUI** (NetBIOS Extended User Interface, 1985) — the **protocol** that works with NetBIOS. **Not routable** — routers drop its packets, so it only works on one local LAN.
- **LANA numbers** — unique pairing of a network card with a transport protocol (0–9).
- **NetBIOS names** = 16 characters; **unique** (only one on the LAN) and **group** (shared / multicast).
- **nbtstat** = the utility to show NetBIOS name information.

```
 Timeline of Windows networking:
   NetBIOS (1983, API) ─▶ NetBEUI (1985, protocol, LAN-only)
        │
        └── represented by LANA numbers, NetBIOS names
        ▼
   Winsock (1991-) = standard TCP/IP socket API  ← what we actually use today
```

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

```
 The two Winsock interfaces:
   ┌───────────────────────────────┐
   │  Application                  │   ← your code
   ├───────────────────────────────┤
   │  Winsock API (ws2_32.dll)     │   ← the functions you call
   ├───────────────────────────────┤
   │  Winsock SPI (providers)      │   ← how protocol vendors plug in
   ├───────────────────────────────┤
   │  Windows kernel network stack │
   └───────────────────────────────┘
```

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

```
 The WSAStartup flow:
   app:  WSAStartup(MAKEWORD(2,2), &wsaData)
        │ request v2.2
        ▼
   ws2_32.dll: "I support up to v2.2"  → loads, sets wsaData.wVersion = 2.2, returns 0
   ws2_32.dll: "I only support v1.1"   → returns WSAVERNOTSUPPORTED (you decide)
```

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

```
 Dynamic linking (DLL) vs Static linking:
   DYNAMIC (DLL)                        STATIC
   ┌────────────┐   calls   ┌────────┐   ┌─────────────────────┐
   │  your.exe  │──────────▶│ ws2_32 │   │ your.exe (code COPY) │
   │(no socket   code)      │ .dll   │   │ (socket code embedded)│
   └────────────┘           └────────┘   └─────────────────────┘
     smaller, easy update     dependency     self-contained, bigger
```

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

```
 SOCKADDR_IN memory layout (16 bytes):
   ┌─────────┬──────────┬──────────────┬────────────┐
   │sin_family│ sin_port │  sin_addr    │ sin_zero[8]│
   │ 2 bytes  │ 2 bytes  │  4 bytes     │ 8 bytes    │
   │  AF_INET │ (htons)  │ address      │ padding=0  │
   └─────────┴──────────┴──────────────┴────────────┘
```

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

```
 FULL TCP WINSOCK FLOW (draw this)
 SERVER                                   CLIENT
 WSAStartup ──────────────────────────── WSAStartup
    │  socket()                             │  socket()
    │  bind()                               │      (no bind needed)
    │  listen()                             │
    │  accept()  ◀── connect() ──────────── │
    │      │     (3-way handshake)          │
    │  send()/recv() ◀───▶ send()/recv()    │
    │  closesocket()                        │  closesocket()
 WSACleanup ──────────────────────────── WSACleanup
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
```
 app: gethostbyname("www.example.com")
      └─ resolver query ─▶ DNS server ──answer(IP)──▶  returns struct hostent
```
- **`gethostbyname(name)`** → `struct hostent *` with official name, aliases, address type, and `h_addr_list` (the addresses).
- **`getservbyname(name, proto)`** → `struct servent *` giving the **port** for a service (from the `services` file), e.g. `getservbyname("http","tcp")` → port 80.
- Async versions: `WSAAsyncGetHostByName`, `WSAAsyncGetServByName`.

### UDP (connectionless) in Winsock
```
 UDP WINSOCK FLOW (no connect/listen/accept)
 RECEIVER (server)                  SENDER (client)
 WSAStartup                         WSAStartup
   socket()                           socket()
   bind()                             sendto(dest_addr)
   recvfrom(from_addr)  ◀───────────── │
   sendto(dest) ─────────────────────▶ recvfrom()
   closesocket                         closesocket
 WSACleanup                          WSACleanup
```
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

## Quick Revision Sheet (Unit 4)
- Winsock = Windows implementation of BSD sockets. Init **WSAStartup**, cleanup **WSACleanup**.
- `SOCKET` (not int), `closesocket` (not close), `WSAGetLastError` (not errno).
- **Server**: socket→bind→listen→accept→send/recv→closesocket.
- **Client**: socket→connect→send/recv→closesocket.
- UDP: receiver=bind→recvfrom; sender=sendto.
- **DLLs** = dynamically loaded libs (ws2_32.dll = main Winsock 2.0). Dynamic = easy update, dependency risk; static = self-contained, bigger.
- Graceful close = `shutdown` then `closesocket`.
- WinSock **history**: arose because early Windows lacked a standard TCP/IP API; agreed at Interop 1991.
