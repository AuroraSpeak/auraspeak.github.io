+++ 
draft = false
date = 2026-01-08T14:35:59+01:00
title = "The UDP Basics"
description = "Setup a basic UDP Server and client"
slug = ""
authors = ["22peacemaker"]
tags = []
categories = []
externalLink = ""
series = []
+++
## TL;DR – run it

I didn’t feel like writing a pure theory post, so I built something that _actually runs_:  
a tiny UDP server + a tiny UDP client.

**What happens:**  
Start the server, start two clients, type `hello` in one client → the server broadcasts it → both clients print it.

This is one of the building blocks for my bigger “decentralized voice chat” journey (that’s what this blog is about).

---

## Concept (what we’re building)

UDP is “connectionless”. That means:

- the server doesn’t _accept connections_ like TCP
- it just receives datagrams from random addresses
- a “client” is basically just an **IP:port** we’ve seen before (`remoteAddr`)

So the concept is:

1. Server listens on a UDP port
2. Every time it receives a packet, it stores the sender address (peer list)
3. When it wants to broadcast, it iterates over all known peers and `WriteToUDP`’s the message to them
4. On shutdown, it sends a `"STOP"` packet to everyone (primitive, but it’s a clear signal)    

No fancy protocol yet. Just “bytes in, bytes out”.

_(Also: UDP can drop, duplicate or reorder packets — that’s kind of the deal. For this toy setup it’s fine.)_

---

## Why this approach?

A few reasons:

- I want something **simple and debuggable**.
- I need a networking base for later stuff (voice packets, NAT fun, encryption… all that pain).
- The server uses a `sync.Map` to track peers because this thing will be concurrent anyway. 
- I also like the idea of routing packets via callbacks — even if right now I’m basically routing everything through `""` as packet type (yep, super advanced).

---

## Step-by-step implementation

### 1) The server: listen, remember peers, broadcast

The server does three jobs:

- open a UDP socket and read packets forever
- store `remoteAddr` when a new peer appears
- hand the packet to a router callback (and in our case: broadcast it)

Here’s the “shape” of the read loop (simplified):

```go
buf := make([]byte, 1024)
n, remoteAddr, err := s.conn.ReadFromUDP(buf)

if _, ok := s.remoteConns.Load(remoteAddr.String()); !ok {
    s.remoteConns.Store(remoteAddr.String(), remoteAddr)
}

_ = s.packetRouter.HandlePacket("", buf[:n])
```

#### Broadcasting

Broadcast is just: iterate over known peers and send the packet back out. If a write fails, we throw the peer away.

```go
func (s *Server) Broadcast(message []byte) {
    s.wg.Go(func() {
        s.remoteConns.Range(func(key, value any) bool {
            if _, err := s.conn.WriteToUDP(message, value.(*net.UDPAddr)); err != nil {
                s.remoteConns.Delete(key)
            }
            return true
        })
    })
}
```

Yes, a few too many brackets. Welcome to Go.

As you might notice, we are using `wg.Go` so have at least go 1.25 installed.
#### Stopping

Stop is intentionally dumb: server sends `"STOP"` to everyone and closes the UDP socket.

```go
func (s *Server) Stop() {
    s.setShouldStop()

    if s.conn != nil {
        s.remoteConns.Range(func(key, value any) bool {
            s.conn.WriteToUDP([]byte("STOP"), value.(*net.UDPAddr))
            return true
        })
        s.conn.Close()
    }

    s.remoteConns.Range(func(key, value any) bool {
        s.remoteConns.Delete(key)
        return true
    })
}
```

It’s not “graceful shutdown enterprise edition”, but it’s fine for now.

### 2) The client: send loop + recv loop

The client is also pretty simple:

- `DialUDP` to the server address
- a send loop that writes whatever you put into `sendCh`
- a recv loop that prints whatever the server sends back
    
Connection setup:
```go
conncetionString := fmt.Sprintf("%s:%d", c.Host, c.Port)
s, _ := net.ResolveUDPAddr("udp4", conncetionString)
c.conn, _ = net.DialUDP("udp", nil, s)
```
Then we start the loops via `WaitGroup.Go`.

Send loop:

```go
func (c *Client) sendLoop() {
    for {
        select {
        case <-c.ctx.Done():
            return
        case msg := <-c.sendCh:
            _, _ = c.conn.Write(msg)
        }
    }
}
```

Recv loop:

```go
func (c *Client) recvLoop() {
	buffer := make([]byte, 1024)
    for {
        n, _, err := c.conn.ReadFromUDP(buffer)
        if err != nil || n == 0 {
            continue
        }

        dst := make([]byte, n)
        copy(dst, buffer[:n])

        if string(dst) == "STOP" {
            log.Info("Received STOP message from server")
            return
        }

        _ = c.packetRouter.HandlePacket("", dst)
    }
}
```

The `copy()` is there so we don’t accidentally reuse the same underlying buffer while we’re still working on the packet.

### Wiring it up (minimal example)

Server:

```go
ctx := context.Background()
srv := server.NewServer(8080, ctx)

srv.OnPacket("", func(packet []byte) error {
    srv.Broadcast(packet)
    return nil
})

_ = srv.Run()
```

## Tiny “gotchas” (aka: future me will thank me)

A few things are intentionally rough right now:

- Packet types are not really used yet (everything is routed as `""`). That’s fine for now, but later I’ll parse types from the payload.
- The `"STOP"` signal is primitive. It’s basically a placeholder for “real protocol states”.
- This is UDP, so you don’t get delivery guarantees for free (later I’ll need sequence numbers / timeouts for real-time voice packets).

The full code lives in the [Github Repo](https://github.com/AuroraSpeak/networking). I’ll keep iterating on it as the blog grows — so expect more packet types and probably some breaking changes along the way.