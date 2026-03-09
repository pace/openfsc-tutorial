<p align="center"><a href="//connectedfueling.com"><img src="assets/connected-fueling-logo.svg" width="50%" /></a></p>

---

# OpenFSC Client Implementation Walkthrough

This walkthrough guides developers through implementing an OpenFSC client that connects a POS system to an OpenFSC server, enabling the Connected Fueling / pay-at-the-pump experience for customers.

It is written for engineers who are new to the OpenFSC protocol. Each part builds on the previous one, so it is recommended to read them in order.

For the full protocol specification, see the [OpenFSC Protocol Specification (Version 1.0)](../openfsc-spec.md).

---

## Parts

### [Part 1: Connection Handling](walkthrough/part-1.md)
Covers everything needed to establish a stable, authenticated connection to the OpenFSC server — including the handshake, authentication, heartbeat handling, and reconnection behaviour. Start here.

### [Part 2: Pump Status & Prices](walkthrough/part-2.md)
Covers reporting pump statuses and fuel prices, handling status subscriptions via `UpdateTTL`, and announcing products with the `PRODUCTS` / `PRODUCT` extension.

### [Part 3: Transactions](walkthrough/part-3.md)
Covers the two payment flows — Post-Pay and Pre-Auth — including how to report transactions, handle pump locking and unlocking, and process payment clearance via `CLEAR`.

### [Part 4: Advanced Topics](walkthrough/part-4.md)
Covers optional protocol extensions: Connection Multiplexing (for connecting multiple sites over a single channel) and Transaction Information (for card payment data delivery). High Availability documentation is pending.

---
