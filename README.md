<p align="center"><a href="//connectedfueling.com"><img src="assets/connected-fueling-logo.svg" width="50%" /></a></p>

---

# OpenFSC Tutorial: From Zero to Hero

<img src="assets/mascot/title.svg" width="180" align="right" alt="">

This walkthrough shows developers how to build an OpenFSC client: the software that connects a site's POS system to PACE's OpenFSC server, so that customers can pay for fuel in an app while staying at the pump (Connected Fueling).

It is written for engineers who know POS and forecourt systems but are new to OpenFSC. Each part builds on the previous one, so read them in order. By the end you'll have a client that connects, reports, gets paid and survives the occasional network hiccup.

<br clear="right">

For the full protocol specification, see the [OpenFSC Protocol Specification (Version 1.0)](https://github.com/pace/openfsc-spec/blob/master/1.0/openfsc-spec.md).

---

## Parts

### [Overview: How Connected Fueling Works](walkthrough/overview.md)
Explains the parties involved, what the customer does in Post-Pay and Pre-Auth, how OpenFSC messages are structured, and the most important terms. Start here if OpenFSC is new to you.

### [Part 1: Connection Handling](walkthrough/part-1.md)
Covers everything you need for a stable, authenticated connection to the OpenFSC server: the handshake, authentication, heartbeats and reconnecting. The foundation everything else stands on.

### [Part 2: Products, Prices & Pump Status](walkthrough/part-2.md)
Covers announcing products with `PRODUCTS` / `PRODUCT`, reporting fuel prices and pump statuses, and sending status updates for a limited time via `UpdateTTL`.

### [Part 3: Transactions](walkthrough/part-3.md)
Covers the two payment flows, Post-Pay and Pre-Auth: how to report transactions, lock and unlock pumps, and close paid transactions when the server sends `CLEAR`. Where fuelings turn into money.

### [Part 4: Advanced Topics](walkthrough/part-4.md)
Covers optional protocol extensions: Connection Multiplexing (connecting several sites over one connection) and Transaction Information (receiving card data for a payment). High Availability will be documented later. Nice to have, not need to have.

### [Part 5: Testing & Go-Live](walkthrough/part-5.md)
Covers the way to production: developing against the sandbox, the acceptance test with PACE (remote and at a pilot station), go-live and support. The part where your client meets the real world.

### [Reference](walkthrough/reference.md)
Lookup tables: the complete `CAPABILITY` lists, all product categories, and all error codes per method.

---
