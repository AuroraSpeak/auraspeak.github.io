+++ 
draft = false
date = 2026-01-10T23:30:29+01:00
title = "Packages, Routing and Tracing"
description = "A progress update on building a UDP protocol in Go: server-side tracing, debug clients, and the first structured packet types."
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++
## TL;DR – run it

I hit a point where “printing stuff” wasn’t enough anymore — I wanted to **see the packet flow**, end-to-end, and be sure that the communication I’m testing is *actually* going over the network.

So I added a **server-only tracer** (debug build only), gave my debug clients a **stable ID handshake**, and then finally did the thing I kept postponing:

**real packets + packet types.**

---

## Concept (what I’m building)

UDP is still UDP: no sessions, no connections, just datagrams from random `remoteAddr`.

That becomes annoying the moment you want to debug more than “it works / it doesn’t”.

So the concept is:

1. **Every debug client has an ID**
2. On startup it sends a **DebugHello** packet containing that ID of the Client.
3. The server stores a mapping `remoteAddr -> clientID`
4. Every packet the server receives/sends gets traced as an event:

   * direction (`in` / `out`)
   * local + remote address
   * payload length (+ a payload preview)
   * and — most importantly — the **clientID** if known

That way I can look at a trace stream and say:

> “ah, client 2 sent this, the server forwarded that, client 1 received it…”

No guessing, no “which terminal was that again?”.

(Yes, I have a small WebUI to display and debug this — but the important part is in `pkg/`. The UI is just a viewer.)

---

## Why this approach?

A few reasons:

* **I needed proof** that my test setup is “real networking” and not accidentally cheating with local shortcuts.
* UDP gives me only `remoteAddr`, but I need an identity that survives restarts and stays readable in logs.
* Tracing *everything* is expensive — so I wanted it to be:

  * available while debugging
  * basically free in release builds

That last point is where I discovered a feature I somehow missed in Go until now: **build tags**.

---

## Build tags in Go (the thing I learned)

Go can include/exclude files at build time using **build constraints** (aka build tags). You declare them at the top of a file using `//go:build ...` and then build with `-tags`.

Example:

* `trace_debug.go` starts with: `//go:build debug`

* `trace_release.go` starts with: `//go:build !debug`

So when I build with:

```
go build -tags=debug ./...
```

…the debug tracer code exists. Without that tag, the tracer becomes a no-op.

Also: the old `// +build ...` syntax still exists for compatibility, but the modern syntax is `//go:build ...`.
This is *exactly* what I needed: **same API, different implementation**, zero runtime switches.

---

## Step-by-step implementation

### 1) The tracer (server-only)

In a debug build, the server allocates a buffered channel:

* `TraceCh` is used to emit trace events
* payload is trimmed to avoid logging megabytes by accident
* and we attach the client ID if we know it

Conceptually it looks like this:

```go
type TraceEvent struct {
    TS       time.Time
    Dir      TraceDirection // "in" / "out"
    Local    string
    Remote   string
    Len      int
    Payload  []byte
    ClientID int
}
```

Then in the server read loop:

```go
packet, err := protocol.Decode(buf[:n])
s.trace(TraceIn, remoteAddr, packet.Payload)
```

…and when broadcasting:

```go
s.conn.WriteToUDP(packet.Encode(), remote)
s.trace(TraceOut, remote, packet.Payload)
```

In release builds, the tracer functions exist but do nothing — so the server code doesn’t have to care.

### 2) Debug client identity handshake

A client ID is only useful if the server learns it early.

So I added a dedicated debug packet type:

* **DebugHello** → payload is the client’s numeric ID (as bytes)

On client start (debug build):

```go
packet := &protocol.Packet{
    PacketHeader: protocol.Header{PacketType: protocol.PacketTypeDebugHello},
    Payload:      []byte(strconv.Itoa(c.ClientState.ID)),
}
c.Send(packet.Encode())
```

On the server (debug build), that ID gets stored:

* `dbgAddrToClientMap` is basically: `remoteAddr.String() -> int`

Now the tracer can do:

* “remote address seen before?”
* attach the stored ID to the trace event

That’s the whole trick: I can now correlate “UDP address” with “human identity”.

### 3) Finally: packets and packet types

At this point, I realized I’m basically building a protocol anyway — so it was time to stop pretending I’m just sending “some bytes”.

#### Packet layout

I went with the smallest possible header:

* **1 byte** for packet type (`uint8`)
* rest is payload

Of course, this header will grow over time as the protocol evolves — but my goal is to keep it as small and intentional as possible.

Why `uint8`?

Because every extra header byte steals space from the payload. And especially later with voice data, I want to keep overhead low. Also: 1 byte gives me **256 packet types**, which is plenty for now.

So the packet is:

```go
const HeaderSize = 1

type Packet struct {
    PacketHeader Header
    Payload      []byte
}
```

Encoding is basically “type byte + payload”:

```go
buf.WriteByte(byte(packetType))
buf.Write(payload)
```

Decoding is “read first byte, rest is payload”.

#### Separate routers (client vs server)

I split the router into two versions:

* **Client router:** handlers just receive the packet
* **Server router:** handlers receive `(packet, clientAddr)` because the server always needs to know *who* sent it

This keeps the handler signatures clean and avoids passing fake addresses around on the client.

### 4) A “debug any” packet type

For debugging I added a “just send whatever” packet:

* **DebugAny** → payload is arbitrary bytes
* server can just broadcast it back out

This is super useful while the protocol is still forming — it’s basically my “wire this through the system” test.

---

## Tiny “gotchas” (future me will thank me)

A few things I already noticed (and will adjust):

* **Build tags are build-time, not runtime.**
  `-tags=debug` is a flag for `go build` / `go run`, not something the compiled binary understands. ([pkg.go.dev][1])
* The tracing call in the server loop should happen **after** I know decoding succeeded — otherwise I risk tracing `packet.Payload` when `packet` is nil.
* I still have `"STOP"` as a raw control message — it works, but at some point I’ll want protocol-level control packets too.

---

## Where this goes next

Now that packet types exist and I can trace the real flow across the network, the next step is to make the transport **secure**.

**Next TODO:** write my own **DTLS implementation** — essentially bringing TLS-like security guarantees (privacy + integrity) to UDP without turning the whole thing into a stream protocol.

This isn’t necessarily the most logical next step — but it’s something I’ve wanted to build for a while, and this feels like the right moment to finally do it.

After that, I can start building on top of a safer foundation:

* protocol-level control packets (instead of ad-hoc string commands)
* proper message framing for voice/data payloads
* replay protection / anti-abuse primitives
* and eventually: ordering, loss handling, and the rest of the “UDP reality” toolbox