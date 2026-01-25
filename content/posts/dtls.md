+++ 
draft = false
date = 2026-01-25T19:54:23+01:00
title = "How I Added DTLS to My UDP Transport"
description = "Added DTLS to Aura-Speak’s UDP transport using Pion DTLS. This post walks through the config, server/client wiring, certificate modes, and the small gotchas (MTU, handshake order, verification) I hit along the way—plus my takeaway: sometimes the best move is to rely on battle-tested libraries."
slug = ""
authors = []
tags = ["aura-speak", "go", "dtls", "pion", "udp", "security", "encryption", "networking", "tls"]
categories = []
externalLink = ""
series = []
+++
## TL;DR – what changed

Last time, I mentioned I wanted to implement DTLS. I didn’t roll my own implementation—instead, I integrated **Pion DTLS**, a fantastic DTLS implementation in Go.

I added a persisted DTLS config, switched the server from raw UDP to `dtls.Listen/Accept + HandshakeContext`, and updated the client to `dtls.Dial`.

---

## Concept (what I’m building)

I already had UDP connectivity and could trace packets end-to-end. But in the real world, I don’t want anyone on the network to read or tamper with what we send. So I added DTLS: it encrypts the traffic and (with certificate verification) authenticates the server to protect against MITM attacks when certificate verification is enabled.

So we need secure communication—starting at the transport layer—to protect against MITM and tampering.

1. DTLS adds a security layer to existing UDP connections just like TLS in the browser (the ‘s’ in https).
2. Both server and client need to speak DTLS; otherwise the handshake can’t complete.
3. Right now I’m doing **server authentication** (server presents a cert). mTLS is possible later, but it requires a CA/enrollment story for client certs.
4. In dev environments, we use self-signed certificates.

---

## Why this approach?

If this project is about building a community-hosted, privacy-respecting voice chat, then transport security can’t be optional. DTLS is the minimum layer that makes ‘I only share what I want’ true on the wire.

- DTLS encrypts application data, so nobody on the local network can read what we send.
- Because managing trusted client certificates is hard, I’m starting with server authentication only.
- We have two certificate modes, self-signed for dev environments and a file-based mode for production

---

## Don’t do everything yourself (what I learned)

I’ve gone pretty deep down the DTLS rabbit hole. But the deeper I got, the more I realized it’s kind of overkill — one of my typical ADHD moments. So I decided to integrate and use Pion DTLS instead. 
Pion DTLS is a common and and Wildly used Implementation for DTLS. Right now it targets DTLS 1.2, and there’s active work toward DTLS 1.3 in the Pion DTLS repo. It gets updates and contains many mechanisms against threats, like replay protection and DoS mitigations.

---

## Step-by-step implementation

### 1) Config-Layout (YAML)

`ServerConfig.Server.DTLS` in three blocks:

- **`certs`:** `mode` (`self_signed` | `files`), `path`, `cert`, `key`, `ca`.  
    if `mode` is empty: `env==dev` → `self_signed`, else `files`.  
    `ca` only if needed, if `client_auth` ClientCAs required.
    
- **`security`:** `client_auth` (e.g. `no_client_cert`, `require_and_verify_client_cert`), `cipher_suites` (optional, Pion-Default if empty), `extended_master_secret` (`request`|`require`|`disable`).
    
- **`tuning`:** `mtu` (Default 1200), `replay_protection_window` (64), `flight_interval` (e.g `"1s"`), `insecure_skip_verify_hello` (only special cases, DoS-Risk).
    

Defaults in `WriteDefaultServerConfig`: `mode=self_signed`, `client_auth=no_client_cert`, `extended_master_secret=request`, mtu/replay/tuning.

---

### 2) DTLS-Config-Builder (`pkg/server/dtls.go`)

`DTLSConfigFromServerConfig(cfg)` → `*dtls.Config`:

- **`self_signed`:**  
    `selfsign.GenerateSelfSigned()` (Pion) → a `tls.Certificate`. No files.  
    Constraint: `client_auth` has to be `no_client_cert`  (for client Certs you need CA files → `files`).
    
- **`files`:**  
    `tls.LoadX509KeyPair(path/cert, path/key)`.  
    if `client_auth` ClientCAs needs: CA from `path/ca` in `x509.CertPool`, at `dtls.Config.ClientCAs`.
    
- **Mappings:**  
    `cipherSuiteMap`, `clientAuthMap`, `extendedMasterSecretMap` — YAML-Strings → Pion-IDs.  
    Cipher Suites: ECDHE_ECDSA, ECDHE_RSA, PSK-Varianten etc.  
    Client-Auth: `no_client_cert` till `require_and_verify_client_cert`.
    
- **Tuning:**  
    MTU, `ReplayProtectionWindow`, `FlightInterval` (Duration), `InsecureSkipVerifyHello` directly in `dtls.Config`.
    

---

### 3) Server: Listen, Accept, Handshake, then Router

**Before (Plain-UDP):** One UDP-Socket, per packet `remoteAddr`.

**Now:**

```text
dtls.Listen("udp", addr, s.dtlsConfig)  →  ln (net.Listener)
ln.Accept()                             →  conn (net.Conn)
conn.(*dtls.Conn).HandshakeContext(ctx) →  explicitly, 30s Timeout
s.nm.RegisterConn(conn)                 →  after that: connReadLoop, Broadcast, etc.
```

Important: **`Accept` does not yet deliver completed handshake connections.**  
Pion: `HandshakeContext` call on `*dtls.Conn`, then  `Read`/`Write` is safe.  
`node_manager` saves `net.Conn`; `conn.RemoteAddr()` replaces earlier „per-Paket-Addr“.  
`connReadLoop` does `conn.Read` → `protocol.Decode` → Router stays the same.  
`Broadcast` iterates over `conn.Write(packet.Encode())`.

`NewServer(port, ctx, cfg)`: `cfg` is allowed to be `nil` → Minimal-Config (env=dev, mode=self_signed).  

**Certificate-Generation:** `util.GenerateCertificates(cfg)` for `mode=files`: checks with `os.Stat` for Cert/Key/CA; if at least one is missing it generates  Self-Signed Cert/Key/CA. Will automatically  `NewServer` with `mode=files` called; Can also be used for tooling/preliminary setup.

---

### 4) Client: DTLS-Dial, one line, The rest is coming right up.

In `Client.Run()`:

**Before:** (e.g.) `net.DialUDP` or something like that., `conn` raw UDP.

**Now:**

```go
c.conn, err = dtls.Dial("udp", raddr, &dtls.Config{InsecureSkipVerify: c.InsecureSkipVerify})
```

`conn` is `net.Conn`; `dtls.Dial` blocks until the handshake completes.
`recvLoop`/`sendLoop` stays: `conn.Read(buffer)`, `conn.Write(msg)`.  
Router, `OnPacket`, `Send` also stays the same.

**`InsecureSkipVerify`:**  
At Self-Signed in Dev: Client would reject self signed certificates.  
`InsecureSkipVerify: true` → Skip test. Only for Dev; Prod: real PKI or own CA.

---

### 5) Certificate for `mode=files` (optional)

`util.GenerateCertificates(cfg)` writes Self-Signed Cert/Key/CA to disk (`path/cert`, `path/key`, `path/ca`), if they don't exist.
Will **not** be called by `NewServer` or Run  – for setup scripts or manual preparation of  `certs/`.  
`self_signed` doesn’t need that: everything is in-memory.

---

## Tiny “gotchas” (future me will thank me)

- `Accept()` ≠ “ready to use”: call `HandshakeContext(ctx)` before `Read/Write`.
- MTU matters: I defaulted to 1200 to avoid fragmentation issues.
- Self-signed in dev: client will fail verification unless you trust the CA or (dev-only) set `InsecureSkipVerify`.
- `InsecureSkipVerifyHello` is a DoS footgun — keep it off unless you really know why.
- If you want mTLS, you need CA files (`mode=files`) so the server can verify client certs.

---

## Where this goes next

My Big Next TO-DO: the repo has grown pretty big, and the client won't need the server’s debug web UI. The server doesn’t need to know about the client codebase and so on so I decided to split this big repo up into smaller ones.

This keeps the "small" parts more maintainable.

In later implementation steps one Aura-Speak instance can host multiple Nodes, or in other words: "One Aura-Speak server instance can contain multiple Virtual Servers". 
