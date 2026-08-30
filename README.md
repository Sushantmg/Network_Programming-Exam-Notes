# Network Programming (CMP 380) — Study Materials

Course: **CMP 380 Network Programming** — Pokhara University (BE IT / BE Software Engineering)
Prepared: August 2026 · from your study resources in `~/Downloads`

## What's in this folder

| File/Folder | Contents |
|---|---|
| `NP Notes/` | Full syllabus notes split by unit |
| `NP Notes/Unit 1 - Network Programming Fundamentals.md` | Network defined, protocols, IPC timeline + 3 share-ways, client-server, **TCP/UDP/SCTP deep comparison, TCP 3-way handshake, ISN-not-zero, 11-state diagram, TIME_WAIT/MSL, congestion control, ports, socket pair** |
| `NP Notes/Unit 2 - Basics of Unix Network Programming.md` | Address structures (sockaddr_in/in6/un/storage), value-result args, byte ordering, all TCP/UDP calls, Unix domain sockets, signals/sigaction/EINTR/zombies, **full daemonization with code + double-fork** |
| `NP Notes/Unit 3 - Advanced Unix Network Programming.md` | **5 I/O models (all phases + sync vs async)**, select details, concurrent servers (fork/select/threads), broadcast/multicast (Class D, TTL), socket options (SO_* + TCP_NODELAY), Syslog, security |
| `NP Notes/Units 4-5 - Winsock Network Programming.md` | WinSock vs Unix, **WSAStartup version-negotiation + WSADATA + error codes**, DLLs + static/dynamic linking, TCP/UDP, **5 I/O models** (select/WSAAsyncSelect/WSAEventSelect/overlapped/IOCP), blocking vs non-blocking + ioctlsocket, WSAPoll, cross-platform |
| `NP Notes/Unit 6 - Utilities, Trends and Emerging Technologies.md` | ping/netstat/iperf/telnet, **WebSockets (handshake + frames/opcodes/close codes/ping-pong)**, gRPC (4 call models), **TLS handshake diagram** + crypto, wrapper function, SDN/OpenFlow/P4/Frenetic |
| `Exam QnA/Network Programming Exam QnA.md` | 50 exam Q&As grouped by unit with 🔴🟡🟢 priorities (**★ = asked in real NCIT 2025 / Gandaki 2025 papers**) **PLUS 22 complete 6-part 8-mark model answers** (①Definition ②Diagram ③Full concept ④Example/code ⑤Common errors/limits ⑥Conclusion) covering every HIGH 8-mark topic: TCP/UDP/SCTP, 3-way handshake + ISN, state diagram, value-result, address structs, 5 I/O models, concurrent servers, socket options, WinSock vs UNIX, WinSock TCP/UDP, overlapped I/O, HTTP vs WebSocket, SDN, TLS, gRPC, WebSocket, blocking vs non-blocking, signal-driven vs multiplexing, daemonizing, SO_LINGER/KEEPALIVE/REUSEADDR/BROADCAST, WSAAsyncSelect vs WSAEventSelect, security wrapper |

> All notes and Q&A are written at **maximum depth (textbook-level)** in simple language with ASCII diagrams — drawn from the **internet (RFC 9293/9260/6455, man pages, Microsoft Learn) plus your own resources** — so you can explain every concept and score full marks. The Q&A 8-mark model answers follow the 6-part structure you requested.

## Syllabus coverage (all 6 units)
- **Unit 1** (5h): Fundamentals — IPC, client-server & P2P, TCP/IP/UDP/SCTP, TCP state diagram, sockets.
- **Unit 2** (12h): Unix basics — address structures, byte/manipulation functions, name lookups, elementary TCP/UDP, Unix domain, signals, daemons.
- **Unit 3** (12h): Advanced Unix — I/O models, concurrent servers (fork/select/pthreads), broadcast/multicast, socket options, Syslog, security.
- **Unit 4** (6h): Winsock basics — API, WSAStartup, bind/listen/accept, send/recv, TCP/UDP clients & servers.
- **Unit 5** (5h): Advanced Winsock — async/nonblocking, select, WSAAsyncSelect, WSAEventSelect, overlapped I/O, WSAPoll, cross-platform.
- **Unit 6** (5h): Utilities & trends — ping/telnet/iperf/netstat, WebSockets, gRPC, TLS/SSL, SDN (OpenFlow, P4, Frenetic).

## Source resources used (in `~/Downloads`)
- `Akhil Mathema - network_programming.pdf` — main source (69 pages of notes).
- `Network Programming.docx` — official course description / syllabus.
- `2. Basics of Unix Network Programming.pptx.pdf`, `3. Advance Unix Network Programming.pptx.pdf`, `3.2 Advance Unix Network Programming.pptx.pdf` (Sushant Paudel) — extra Unit 2/3 detail (SCTP, TCP flags, value-result args, daemon/syslog, broadcast/multicast, security).
- `3.0-Winsock.pdf`, `5_winsock_async.pdf`, `Ch4_Winsock_Programming.pdf` — Winsock history, DLLs, async/overlapped I/O detail.

## Real past-exam papers (extracted by OCR from screenshots)
- **NCIT (Nepal College of Information Technology) Assessment, Spring 2025** — BESE VI, Network Programming (New), Full 100/Pass 45: TCP/UDP/SCTP, TCP state diagram, byte ordering, sockaddr structures, value-result args, bind() in TCP client, broadcast/multicast + daemonize, I/O models, socket options, multiple-client handling, Winsock vs UNIX + static/dynamic linking, Winsock DLLs + WSAStartup/cleanup, overlapped I/O, event-driven + WSAEventSelect, HTTP vs WebSocket + server, SDN, short notes (gRPC, telnet, ipconfig, remote login, wrapper function, iperf & netstat, TLS/SSL).
- **Gandaki College of Engineering & Science, Spring 2025** — BESE VI, same course: networking importance + protocol types, TCP state diagram, socket address structures (IPv4/IPv6, sockaddr_storage, value-result), signal()/sigaction(), multiple-client handling, non-blocking & signal-driven I/O, socket options, WSAStartup/WSACleanup, TCP/UDP Winsock client-server, WSAGetLastError + async functions, event-driven + WSAEventSelect, WSAPoll vs select, short notes (WebSockets, iperf, P4 & Frenetic, daemonizing).

These real papers were used to **prioritise** the Q&A (🔥/🟡/🟢) — the most-repeated questions are marked HIGH priority.
