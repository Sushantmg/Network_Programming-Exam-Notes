# Unit 5 — Advanced Winsock: Asynchronous / Non-blocking I/O

**Subject:** Network Programming (CMP 380) · **Unit 5** | **Priority: HIGH**
**Number of teaching hours in syllabus:** 5

> This unit answers one question: *"How do I make a server that handles many clients at once on Windows?"* The answer is the **five Winsock I/O models**. Understand the *problem* (blocking I/O freezes the server), then each model is a different *solution*. Draw the notification diagram for each model.

---

## 5.1 The problem with ordinary (blocking) I/O

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

```
 BLOCKING vs NON-BLOCKING
   BLOCKING:  recv() ──▶ sleeps until data ──▶ returns data   (thread blocked)
   NON-BLOCK: recv() ──▶ returns WSAEWOULDBLOCK if no data   (thread free)
                     ──▶ returns data if ready
```

You switch modes with **`ioctlsocket`**:
```c
unsigned long mode = 1;                    /* 1 = non-blocking, 0 = blocking */
ioctlsocket(s, FIONBIO, &mode);
```
> Note: calling `WSAAsyncSelect` or `WSAEventSelect` for a socket **automatically puts it in non-blocking mode**; it does not return to blocking mode until you call `ioctlsocket(FIONBIO, 0)`.

---

## 5.2 The 5 Winsock I/O models (HIGH, asked every year)

| Model | Mechanism | Best for |
|---|---|---|
| **select** | check `fd_set` readability/writability | cross-platform, simple |
| **WSAAsyncSelect** | **Windows message** to a window | GUI apps, Win95/98 |
| **WSAEventSelect** | **event object** notification | Win95 (WS2)+ |
| **Overlapped I/O** | post many `WSASend/WSARecv`; get a completion notice | best performance |
| **Completion port (IOCP)** | thread pool over a completion port | hundreds/thousands of sockets (NT/2000 only) |

```
 THE FIVE MODELS AT A GLANCE
  1. select        : you ask "which are ready?" ── blocking, then read
  2. WSAAsyncSelect: socket→ window message (needs a window/GUI)
  3. WSAEventSelect: socket→ event object (no window, ≤64/thread)
  4. Overlapped    : post MANY I/O ops, kernel runs them, notify on completion
  5. IOCP          : completions go to a thread pool over a port (most scalable)
```

### The select model
Comes straight from Unix. Call `select(nfds, readfds, writefds, exceptfds, &timeout)` to ask: *"which of these sockets are ready to read/write?"* before doing real I/O. Shared by both OSs:
```c
select(0, &readfds, NULL, NULL, &tv);   /* wait until a socket in readfds is readable */
```
- On Windows `fd_set` is typically limited to **64 descriptors** and `select` **modifies** the sets (falls back to a "winsock 1.1" limit), so for larger numbers of sockets the event/overlapped models scale better.

```
 select model:
   app: build readfds with the sockets I care about
        select(0, &readfds, ...)  ──▶ blocks until one is ready
        app: check FD_ISSET → which sockets are ready → recv on them
```

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
 WSAAsyncSelect — message-based notification (needs a window):
   Socket event (e.g. FD_READ)  →  Winsock posts a message to the window  →  WindowProc handles it
      [socket]  ──────────────▶  [message queue]  ────────────────────────▶  your code (WndProc)
```
- **Requires a window (`HWND`) and a message loop** — so it only fits GUI applications.

### WSAEventSelect — event-object based (HIGH)
Same idea but uses **event objects** instead of window messages — so **no window is needed** (good for console/background apps).
- `WSACreateEvent()` makes an event object.
- `WSAEventSelect(s, hEvent, lNetworkEvents)` ties the socket's events to that object.
- `WSAWaitForMultipleEvents(nEvents, lphEvents, fWaitAll, timeout, fAlertable)` waits (max **64** events per thread).
- `WSAResetEvent` / `WSACloseEvent` manage the event.

```
 WSAEventSelect — event-object based (no window):
   app: hEvent = WSACreateEvent();              → create event object
        WSAEventSelect(s, hEvent, FD_READ|FD_WRITE|FD_CLOSE)
        WSAWaitForMultipleEvents(...)  ──▶ blocks until an event is signaled
        WSAEnumNetworkEvents(s, hEvent, &events)  → what happened? → handle
```

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
 OVERLAPPED I/O — many ops at once:
   app: WSASend(op1) ──▶ returned (queued)
        WSARecv(op2) ──▶ returned (queued)
        WSARecv(op3) ──▶ returned (queued)      ← thread NOT blocked
        ... keep doing other work ...
        completion arrives → event/callback fires → handle op result
   └─ the KERNEL does all the I/O in the background
```

### Completion port (IOCP) — the most powerful
`CreateIoCompletionPort(...)` creates a port and associates handles; a **pool of worker threads** services completed I/O. Best performance for **hundreds/thousands** of sockets. (NT/2000 only; hardest to write.)

```
 IOCP — thread pool over a completion port:
   [many sockets] ──▶ Completion Port ──▶ worker thread 1
                                          worker thread 2   ← pool handles completions
                                          worker thread 3
   └─ the most scalable: thousands of connections, few threads
```

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

```
 select vs WSAPoll
   select:  fd_set (fixed, ~64)  ── overwritten each call, must rebuild
   WSAPoll: WSAPOLLFD[] (no limit) ── preserved, just read revents
```

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

```
 Cross-platform design:
   ┌─────────────────────────── common code ───────────────────────────┐
   │  socket(); bind(); listen(); accept(); send(); recv()             │
   └───────────────────────────────────────────────────────────────────┘
        ▲ winsock.h +            ▲ sys/socket.h
        │ WSAStartup/Cleanup     │ no init
        │ closesocket            │ close
        │ WSAGetLastError        │ errno
     ┌──┴─── (Windows)        (Unix) ───┴──┐  ← all hidden behind #ifdef _WIN32
```

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

## Quick Revision Sheet (Unit 5)
- **5 I/O models**: select, WSAAsyncSelect, WSAEventSelect, overlapped, IOCP (increasing power/complexity).
- **WSAAsyncSelect** = messages to a window (GUI). **WSAEventSelect** = event objects (max 64/thread).
- **Overlapped** = many I/O at once, completion via event or callback; `WSA_IO_PENDING` = queued not error.
- **WSAPoll** = array-based, no fd_set limit (improves on select).
- **Blocking vs non-blocking**: blocking sleeps; non-blocking returns `WSAEWOULDBLOCK` (switch with `ioctlsocket FIONBIO`).
- **Cross-platform**: wrap Windows parts in `#ifdef _WIN32`.
