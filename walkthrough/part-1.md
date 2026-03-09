# OpenFSC Client Implementation Walkthrough
## Part 1: Connection Handling

This walkthrough guides you through implementing an OpenFSC client that connects your POS system to an OpenFSC server. We'll start from the very first byte sent over the wire all the way to a stable, authenticated, and heartbeat-aware connection.

---

## Prerequisites

Before writing any code, make sure you have the following on hand:

- **SiteAccessKey** (a UUID) and **Secret** — provided to you during gas station on-boarding. These are your credentials for authenticating with the OpenFSC server.
- **Server address and port** — the OpenFSC server hostname and either port 443 (WSS/TLS) or the VPN-secured TCP port.
- **Connection type decision** — are you connecting over secure WebSockets (WSS) or raw TCP over a VPN? This affects your transport layer but *not* the protocol logic above it. This walkthrough focuses on the protocol layer, which is identical for both.

---

## Step 1: Open a TCP/WebSocket Connection

Establish a persistent, reliable, ordered connection to the OpenFSC server. The protocol requires a byte stream where messages are not lost or reordered.

- If using **WSS**: connect to `wss://<server>:443`. Handle TLS negotiation via your WebSocket library.
- If using **TCP over VPN**: open a raw TCP socket to the server's address and port.

Once the connection is open, you are in the **Not Authenticated State**. No authenticated methods may be used yet.

> **Important:** If the connection drops for any reason, your client must reconnect immediately and restart the handshake from scratch.

---

## Step 2: Understand the Message Format

Every single message in the protocol — sent or received — follows this structure:

```
<tag> <method> <arg0> <arg1> ... <argN>
```

Key rules:
- Fields are separated by **single spaces**.
- Every message is terminated by **`\r\n`** (CRLF). Your parser must read until it finds `\r\n` to know a message is complete. In all examples throughout this walkthrough, the line break represents this terminator.
- The **tag** is either `*` (notification, no reply expected) or an alphanumeric identifier like `S1`, `C0` (request/response, a reply with the same tag is expected).
- The **method** is an uppercase alphanumeric word like `CAPABILITY`, `PUMPS`, etc.
- The default encoding is **ASCII**. You can negotiate a different encoding (e.g. UTF-8) during the handshake using `CHARSET` — but for a first implementation, staying with ASCII is fine.

Get your message parser right before anything else. A good approach is a line reader that buffers incoming bytes and emits complete messages when it encounters `\r\n`.

---

## Step 3: Handle the CAPABILITY Exchange

Immediately after the connection is open, **both sides send a `CAPABILITY` notification simultaneously**, without waiting for the other. This is always the very first message from each party.

**What to send:**

```
* CAPABILITY BEAT CHARSET PLAINAUTH PRICE PUMP TRANSACTION LOCKEDPUMP QUIT
```

This tells the server which methods *your client* can handle. At minimum you must announce the methods you actually implement. For the connection phase, the relevant ones are:

- `BEAT` — your client can respond to heartbeats
- `CHARSET` — your client supports encoding negotiation
- `PLAINAUTH` — your client supports plain credential authentication
- `QUIT` — your client understands disconnect notifications

As you implement more functionality (pumps, prices, transactions), you will add those methods to this list.

**What to expect from the server:**

```
* CAPABILITY CLEAR HEARTBEAT LOCKPUMP PRICES PUMPS PUMPSTATUS QUIT TRANSACTIONS UNLOCKPUMP
```

Parse the server's capability list and store it. In the future, you can use this to detect which optional features the server supports. For now, note that if the server does not announce `HEARTBEAT`, it won't send heartbeat requests — but in practice all servers will.

> `CAPABILITY` is a **Notification** (tag `*`), so neither side sends an `OK` or `ERR` in response. Just send it and move on.

---

## Step 4: (Optional) Negotiate Character Encoding

If you need to send or receive non-ASCII characters (e.g. fuel product names with umlauts), send a `CHARSET` request *before* authenticating:

```
C0 CHARSET UTF-8
```

The server will reply:

```
C0 OK
```

Note that the tag `C0` here is chosen by your client — it just needs to be unique for the duration of this request. A simple incrementing counter (C0, C1, C2, …) works well.

> **Caution:** The `OK` response itself is still in the *old* encoding. All messages *after* the `OK` use the new encoding.

If the encoding is not supported, the server replies with:
```
C0 ERR 404 Unknown encoding
```

If you stick to ASCII-only content, you can skip this step entirely for now.

---

## Step 5: Authenticate with PLAINAUTH

Now send your credentials to transition into the **Authenticated State**:

```
C1 PLAINAUTH <SiteAccessKey> <Secret>
```

For example:
```
C1 PLAINAUTH 9eb56d5e-6563-430a-9d39-5ddf567e73d5 1d3b755d3bce8f09b4f8ff08dabf1796
```

The server responds:

```
C1 OK
```

You are now authenticated. You may begin using all methods defined for the **Authenticated State**.

If authentication fails:
```
C1 ERR 401 SiteAccessKey and/or secret are not valid
```

In this case, check your credentials. Retrying immediately with wrong credentials is pointless — surface the error and wait for operator intervention.

---

## Step 6: Implement a Tag Manager

Before going further, you need a robust way to manage request/response pairs. The protocol is asynchronous — the server can send you requests at any time, interleaved with your own requests.

A simple tag manager should:

1. **Generate unique tags** for outgoing requests (e.g. `C0`, `C1`, `C2`, …).
2. **Track pending outgoing requests** — store a mapping of `tag → callback/handler` so you can match an incoming `OK` or `ERR` to the request you sent.
3. **Route incoming server requests** — when the server sends a tagged request (e.g. `S4 TRANSACTIONS`), your client must reply using that same tag (`S4 OK` or `S4 ERR ...`).

Keep client-originated tags and server-originated tags separate to avoid collisions (e.g. prefix yours with `C`, the server uses `S`).

---

## Step 7: Implement HEARTBEAT / BEAT

The server will periodically send a `HEARTBEAT` request. **You must respond within 20 seconds**, or the server will consider the connection unstable and eventually force-close it with a `QUIT`.

An incoming heartbeat looks like:
```
S6 HEARTBEAT 2019-11-13T07:00:04Z
```

Your client must respond with **two messages** in sequence, both using the same tag:

1. A `BEAT` notification with your current local timestamp in RFC-3339 format:
```
S6 BEAT 2019-11-13T08:00:05+01:00
```
2. An `OK` to conclude the request:
```
S6 OK
```

The `BEAT` message is purely informational — it lets the server calculate the clock drift between your system and its own. The `OK` is what actually concludes the `HEARTBEAT` request.

Your implementation checklist for heartbeats:
- [ ] Parse incoming `HEARTBEAT` messages and extract the tag.
- [ ] Immediately reply with `BEAT` carrying your current timestamp (RFC-3339 with timezone offset).
- [ ] Follow up with `OK` using the same tag.
- [ ] Do this within 20 seconds — if your system is under load, make heartbeat handling a high priority.

---

## Step 8: Implement QUIT Handling

The server may send a `QUIT` notification at any time to signal it is about to disconnect:

```
* QUIT <reason message>
```

When you receive this, the server will drop the connection immediately after. Your client should:
1. Log the reason for diagnostics.
2. Close the connection cleanly on your side.
3. **Immediately attempt to reconnect** and restart the handshake from Step 1.

Your client may also send `QUIT` before intentionally disconnecting:
```
* QUIT bye bye
```

This is optional — you may also just close the socket — but it is good practice.

---

## Step 9: Handle Unknown and Error Messages Gracefully

During development (and in production), you may receive methods you don't yet handle, or send requests that result in errors. Build these habits in early:

- **Unknown method received:** Reply with `ERR 405 Method unknown` using the incoming tag, then continue operating normally. Do not crash.
- **Bad request from your side:** If you receive `ERR 400 Bad request`, inspect the message you sent — it likely has a malformed tag, argument, or method name.
- **Wrong state error (`403`):** You sent an authenticated-state method before completing the handshake (or vice versa). Check your state machine.
- **Unexpected disconnect:** Always reconnect immediately and restart the handshake.

---

## Connection State Summary

Here is the complete lifecycle of a connection:

```
[Socket Connected]
       │
       ▼
[Not Authenticated State]
  • Both sides send CAPABILITY simultaneously
  • Client optionally sends CHARSET
  • Client sends PLAINAUTH → receives OK
       │
       ▼
[Authenticated State]
  • Normal operation (pumps, prices, transactions)
  • Server sends HEARTBEAT periodically → client replies BEAT + OK
  • Either side may send QUIT → reconnect immediately
       │
       ▼
[Disconnected → Reconnect immediately]
```

---

## What's Next

With a stable, authenticated, heartbeat-aware connection in place, the next part of this walkthrough will cover **reporting pump statuses and prices** — the first things the server will ask for immediately after authentication.
