# Unit 6 — Network Utilities, Current Trends & Emerging Technologies

**Subject:** Network Programming (CMP 380) · **Unit 6** · **Priority: HIGH (short notes & descriptive)**

> This unit is a mix of **practical tools** (ping, iperf, netstat, telnet) and **modern technologies** (WebSockets, gRPC, TLS/SSL, SDN). In the exams these mostly come as **short notes (5 marks)** and a couple of **8-mark** descriptive questions (SDN, WebSockets). Everything below is written to be *explained* in your own words.

---

## 6.1 Network Utilities & Applications (short notes)

| Utility | What it does | Port/Protocol |
|---|---|---|
| **ping** | Test reachability + round-trip time (RTT) to a host | **ICMP** |
| **telnet** | Unencrypted remote terminal login; can also test a TCP port | TCP 23 (`telnet host 80`) |
| **ip / ifconfig** | Show/configure network interfaces (IP, MAC, netmask) | local |
| **iperf** | Measure **network throughput / bandwidth** between host & client | TCP/UDP |
| **netstat** | Show connections, routing table, listening ports, interfaces | local |
| **remote login (rlogin / ssh)** | Log into a remote machine (ssh = secure + encrypted) | TCP 22 |

**iperf** — a command-line tool where one machine runs `iperf -s` (server) and another runs `iperf -c <server-ip>` (client) to measure how much data can flow per second (bandwidth) and how reliably. Useful for testing whether a link is as fast as promised.

**netstat** — dumps the network stack status: all open TCP/UDP connections, which ports are listening (`-l`), the routing table (`-r`), and interface stats. Great for debugging "why can't I connect?".

---

## 6.2 Real-Time Communication Protocols

### 6.2.1 WebSockets (HIGH — asked directly: "HTTP vs WebSocket" + simple server)

**The problem with HTTP:** HTTP is **request–response**. The client asks, the server answers — the server **cannot push** data on its own, and every message repeats headers. For real-time apps (chat, live scores, games) this is slow and awkward.

**WebSocket** = a protocol that gives **full-duplex (bidirectional), persistent** communication over **one TCP connection**. After a one-time handshake, **either side can push messages any time**.

- URIs: `ws://` (plain) and `wss://` (over TLS, i.e. encrypted).
- Uses ports like HTTP (80/443 for wss).

### Handshake (HTTP → WebSocket upgrade)
```
CLIENT                                           SERVER
  │ GET /chat HTTP/1.1                              │
  │ Host: example.com                               │
  │ Upgrade: websocket                              │
  │ Connection: Upgrade                            │
  │ Sec-WebSocket-Key: <base64>                    │
  │                                ──────────▶      │
  │                              HTTP/1.1 101       │
  │                               Switching Protocols (Upgrade: websocket)
  │ ◀────────── 101 + Sec-WebSocket-Accept ──────   │
  │                                                │
  │  === now both sides can send frames freely ===  │
```

| HTTP vs WebSocket | HTTP | WebSocket |
|---|---|---|
| Direction | request–response (client asks) | **full-duplex** (both can push) |
| Connection | closed after response (unless keep-alive) | **persistent** single connection |
| Overhead | headers repeated per message | small frames |
| Latency | higher (new handshake per request) | low |
| Use case | normal web pages, APIs | chat, gaming, live dashboards, stock tickers |

**Simple WebSocket server idea (pseudo):**
```
1. Create a TCP listening socket (like any server).
2. Wait for a client's HTTP "Upgrade" request.
3. Reply "101 Switching Protocols".
4. Loop: read a frame, optionally broadcast it to all connected clients.
```

```
 WEBSOCKET SERVER FLOW (completeness)
   socket() → bind() → listen()          ← ordinary TCP server
        │
        ▼  connect() arrives
   conn = accept()
        │
        ▼  client sends HTTP Upgrade request
   read request; verify "Upgrade: websocket"
        │
        ▼
   send "HTTP/1.1 101 Switching Protocols" + Sec-WebSocket-Accept
        │  ── now the connection is a WebSocket (full duplex) ──
        ▼
   loop { read frame from conn ──▶ / parse opcode+payload / handle or broadcast }
        and, if needed: server can PUSH frames to any client at any time
```

### WebSocket frames & protocol internals (RFC 6455)
After the handshake, data travels in small pieces called **frames**. A frame header starts with a byte:
```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)    |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
```
- **FIN** (1 bit): set if this is the final frame of a message.
- **RSV1-3** (3 bits): reserved; only meaningful if an extension is negotiated.
- **opcode** (4 bits): tells what kind of frame:
  - `0x0` continuation, `0x1` **text**, `0x2` **binary**, `0x8` **close**, `0x9` **ping**, `0xA` **pong**.
- **MASK** (1 bit): **must be 1 for client→server** frames (masking protects against cache-poisoning attacks). The masking key (4 bytes) follows the length.
- **Payload length** (7 bits): if 0–125 it's the length; if **126**, the next 2 bytes are the length; if **127**, the next 8 bytes are the length.

**Ping / Pong (keepalive):** either side may send a `ping`; the peer must reply with a `pong` containing the same payload. This is how a WebSocket connection stays alive and detects dead peers.

**Close codes:** a close frame carries a code, e.g. **1000** (normal), **1001** (going away), **1002** (protocol error), **1008** (policy violation), **1011** (server error).

---

### 6.2.2 gRPC (short note)

**gRPC** = a **high-performance, open-source RPC (Remote Procedure Call) framework** by Google. It lets a program **call a function on another machine** as if it were local.

- Built on **HTTP/2** (gives: multiplexing, bidirectional streaming, header compression).
- **Default serialization = Protocol Buffers (protobuf)** — a compact **binary** format. You define messages & services in a `.proto` file, then the compiler generates client/server code in **many languages** (C++, Java, Python, Go, Node.js).
- **Features:** bi-directional streaming, efficient binary encoding, language-agnostic, strongly typed.
- **Use cases:** microservices, mobile↔backend, real-time streaming.

```proto
service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}
message HelloRequest { string name = 1; }
message HelloReply   { string message = 1; }
```

**gRPC vs REST:** gRPC = binary protobuf + HTTP/2 + streaming; REST = JSON over HTTP/1.1, human-readable, simpler.

**gRPC call models:** **Unary** (one request, one response), **server streaming** (one request, many responses), **client streaming** (many requests, one response), and **bidirectional streaming** (many requests, many responses − full duplex).

```
 THE FOUR gRPC CALL MODELS (draw these)
 UNARY                SERVER STREAMING     CLIENT STREAMING     BIDIRECTIONAL
 C          S         C          S         C          S         C          S
 │ req ───▶│          │ req ───▶│          │ req1 ──▶│          │ req1 ──▶│
 │ ◀──resp │          │ ◀──resp1│          │ req2 ──▶│          │ ◀──resp1│
 │         │          │ ◀──resp2│          │ req3 ──▶│          │ req2 ──▶│
 │         │          │ ◀──resp3│          │ ◀──resp │          │ ◀──resp2│
 │ one-to-one         │ 1 → many           │ many → 1            │ many ↔ many
```

**How the pieces fit (proto → code → client/server):**
```
 .proto file (Greeter/SayHello)  ──protoc compile──▶  stubs for C++/Java/Python/Go...
        │
        ├─▶ server: implements the service, runs on a port
        └─▶ client: calls the generated stub → transparently an RPC over HTTP/2
```

---

## 6.3 Security in Network Programming — TLS/SSL (HIGH short note)

### 6.3.1 What is TLS/SSL?
- **SSL** (Secure Sockets Layer) evolved into **TLS** (Transport Layer Security).
- It sits **between the application and TCP** — it *wraps* plain socket data so that, before it hits the network, it is encrypted and authenticated.

```
Application (HTTP/WebSocket)   ← data in the clear
        ▼
      TLS/SSL layer  (encrypt + authenticate + integrity check)
        ▼
        TCP  →  network    ← only ciphertext travels
```

### 6.3.2 The cryptography concepts used (know all three)
1. **Encryption** — plaintext → ciphertext.
   - **Symmetric** (same key to encrypt & decrypt), e.g. AES — fast.
   - **Asymmetric / public-key** (public + private key pair), e.g. RSA — used to safely exchange a session key.
2. **Hashing** — a **one-way** function producing a fixed-size digest (e.g., SHA-256). Used to **verify integrity** (detect tampering); you cannot reverse it.
3. **Certificates** — bind a **public key to an identity**. They are **signed by a Certificate Authority (CA)**. The client checks the certificate to *authenticate* the server and to safely obtain its public key.

### 6.3.3 What the TLS handshake does
1. Client sends **ClientHello** — protocol version + list of supported **cipher suites** + a random number.
2. Server replies **ServerHello** — chosen cipher suite + its random; then sends its **certificate**.
3. Client **verifies the server's certificate** via a trusted **CA** (authenticating the server).
4. Both derive the **session (symmetric) keys** — either via the certificate's public key (RSA) or an ephemeral **Diffie-Hellman** key exchange (giving **forward secrecy**).
5. From then on, all data is sent encrypted with the shared symmetric key (e.g. **AES**) + integrity-checked (e.g. **SHA**). (`https://` = HTTP over TLS.)

```
CLIENT                                  SERVER
  │ ① ClientHello (versions, ciphers, random)  │
  │ ────────────────────────────────────────▶  │
  │ ② ServerHello (chosen cipher, random)      │
  │ ③ Certificate  ◀────────────────────────── │
  │ ④ verify cert via CA                       │
  │ ⑤ key exchange (RSA / Diffie-Hellman)      │
  │ ◀─────────────────────────────────────────│
  │ ⑥ [now: encrypted with the shared key]     │
  └────────── all http data is now TLS ────────┘
```

### 6.3.4 Why it matters
Plain TCP/UDP send data **in the clear** → vulnerable to **eavesdropping, tampering, impersonation**. TLS fixes all three. As a network programmer you secure sockets using a library like **OpenSSL** (`SSL_CTX`, `SSL_new`, `SSL_connect`, `SSL_accept`, `SSL_read/SSL_write`).

---

## 6.4 Network Security Programming: Wrapper function (HIGH short note)

A defensive programming technique: **don't let untrusted clients reach your real service directly — run every connection through a small "wrapper" (gatekeeper) that checks a policy first.**

- We can identify/allow clients by:
  1. **Hostname / domain name** — look up and allowlist trusted names (note: DNS can be spoofed).
  2. **IP number** — restrict by source IP address.
  3. **A wrapper program** — a short front-end that, before starting the real server, checks "is this client allowed?" If yes, pass control to the actual service; if no, drop/deny.

```
CLIENT ──► WRAPPER (checks IP/hostname against policy) ── allow ──► REAL SERVICE
                              │
                              └──── deny ──► close / log
```

---

## 6.5 Software-Defined Networking (SDN) (HIGH, 8-mark descriptive)

### 6.5.1 The core idea
Traditional switches/routers have *both* the **brain and the muscle** built in: each device decides where packets go (**control plane**) and moves them (**data/forwarding plane**) with its own closed firmware.

**SDN separates these two planes:**
- **Control plane** (the "brain" — where packets should go) moves into a **central SDN controller** (a software program).
- **Data plane** (the "muscle" — actually moving packets) stays in simple, cheap switches that just follow instructions.

```
            ┌─────────────────────────────┐
            │       SDN CONTROLLER        │  ← the brain (centralized, programmable)
            │   (computes routes/policy)  │
            └──────────────┬──────────────┘
                           │  OpenFlow (control messages: install flow rules)
        ┌─────────┬────────┴────────┬─────────┐
       switch1   switch2          switch3    ...   ← simple forwarding devices (data plane)
```

### 6.5.2 Benefits (asked: "Discuss its key advantages")
- **Centralized control** — one place to see/control the whole network.
- **Programmable** — network behaviour is changed in software, not on each device.
- **Agility & automation** — quickly provision/move services.
- **Better utilization** — controller can balance load network-wide.
- **Vendor-independent** — switches are generic, programmable hardware.

### 6.5.3 OpenFlow
The **standard protocol** between the **SDN controller** and the **switches**. The controller uses it to install rules in the switches' **flow tables** — i.e., "if a packet matches X, forward it to Y". This is the foundation of most SDN systems.

### 6.5.4 P4 and Frenetic (short note, asked directly)
- **P4** = *Programming Protocol-independent Packet Processors* — a **high-level language to program the data plane itself** (what a switch should *do* with packets), independent of the hardware chip/protocol.
- **Frenetic** = a **domain-specific language (DSL) to program the SDN controller** — you write high-level network policies and it compiles them into OpenFlow rules.

> In short: **P4 programs the switches (data plane); Frenetic programs the controller (control plane).**

---

## Quick Revision Sheet
- Utilities: ping (ICMP), telnet (TCP 23), ip/ifconfig, **iperf** (bandwidth), **netstat** (connections/routing), ssh (secure login).
- **WebSockets** = full-duplex, persistent, single TCP connection; upgrade handshake → 101; `ws://`/`wss://`; beats HTTP for real-time push.
- **gRPC** = RPC on HTTP/2 + Protocol Buffers (binary), multi-language, streaming.
- **TLS/SSL** = encryption + authentication + integrity; → encryption (AES/RSA), hashing (SHA), certificates (signed by CA).
- **Wrapper function** = gatekeeper that checks client policy before letting it reach a service.
- **SDN** = separate control plane (central controller) from data plane (switches); **OpenFlow** = controller↔switch protocol; **P4** = program data plane; **Frenetic** = program the controller.
