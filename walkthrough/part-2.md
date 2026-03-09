# OpenFSC Client Implementation Walkthrough
## Part 2: Pump Status & Prices

With a stable, authenticated connection in place (Part 1), the server will almost immediately start asking for pump and price data. This part covers how to respond to those requests, how to proactively push updates, and the important concept of status subscriptions via `UpdateTTL`.

---

## Step 1: Understand the Notification Pattern

Before diving into specific methods, it's important to understand a pattern that repeats throughout the protocol: **bulk requests answered with individual notifications**.

When the server wants all pump statuses, it doesn't ask for one pump at a time. Instead it sends a single request (`PUMPS`), and your client responds by sending one `PUMP` notification per pump, followed by a final `OK` to conclude the request. The same pattern applies to prices (`PRICES` → multiple `PRICE` notifications → `OK`).

```
Server:  S1 PUMPS
Client:  * PUMP 1 free
Client:  * PUMP 2 in-use
Client:  * PUMP 3 ready-to-pay
Client:  S1 OK
```

Two things to note:
- The individual `PUMP` notifications use the `*` tag — they are fire-and-forget.
- The `OK` uses the **server's tag** (`S1`) to conclude *that specific request*.

Make sure all notifications are sent before you send the `OK`. The server considers the request complete once it receives the `OK`.

---

## Step 2: Implement PUMPS

`PUMPS` is the server's way of asking for the current status of every pump at your site.

**Incoming request:**
```
S1 PUMPS
```

**Your response — one `PUMP` notification per pump, then `OK`:**
```
* PUMP 1 free
* PUMP 2 out-of-order
* PUMP 3 in-use
* PUMP 4 ready-to-pay
S1 OK
```

### Pump Statuses

Your POS system must map its internal pump states to one of these values:

| Status | Meaning |
|---|---|
| `free` | Pump is idle and available for use |
| `in-use` | Customer is currently fueling |
| `ready-to-pay` | Fueling complete, awaiting payment (Post-Pay) |
| `locked` | Pump is indication the default status of the Pre-Auth workflow (can be reserved by a user to start a mobile fueling transaction) |
| `out-of-order` | Pump is not operational |

Think carefully about how each state in your POS maps to these. The most critical ones to get right are `ready-to-pay` (triggers payment flow in Post-Pay) and `locked` (indicates a Pre-Auth workflow).

> The spec also defines an `in-transaction` status, but this is not used in practice. Do not implement or send it.

---

## Step 3: Implement PUMPSTATUS

`PUMPSTATUS` is the single-pump equivalent of `PUMPS`. The server uses it to ask about one specific pump, often when a customer has selected it in the app.

**Incoming request (basic):**
```
S2 PUMPSTATUS 3
```

**Your response:**
```
* PUMP 3 free
S2 OK
```

Errors to handle:
- If the pump number is unknown, reply with `ERR 404 Pump unknown` instead of `OK`.

### The UpdateTTL Subscription

`PUMPSTATUS` has an important optional second argument: `UpdateTTL`. This is a number of seconds for which the server is asking your client to **continuously push status updates** for that pump whenever its status changes.

**Incoming request with TTL:**
```
S3 PUMPSTATUS 3 30
```

Your response works the same way — send the current status, then `OK`:
```
* PUMP 3 free
S3 OK
```

But now, for the next 30 seconds, if pump 3 changes state, you must proactively send a notification:
```
* PUMP 3 in-use
* PUMP 3 ready-to-pay
```

These are unsolicited `*`-tagged notifications — no `OK` is needed for them.

Implementation notes:
- The valid TTL range is **30 to 300 seconds**. If the value is outside this range, reply with `ERR 416 UpdateTTL is too large or too low`.
- A new `PUMPSTATUS` request for the same pump resets the subscription timer.
- **Only send updates when the status actually changes.** Don't poll and send the same status repeatedly.
- You can have subscriptions active for multiple pumps simultaneously — track them per pump.

---

## Step 4: Implement Proactive PUMP Notifications

Your client doesn't need to wait to be asked. Whenever a pump's status changes — whether there's an active subscription or not — you should send a proactive `PUMP` notification:

```
* PUMP 3 in-use
```

This keeps the server's view of your forecourt current in real time. The server uses this data to update what customers see in the app, so timely updates matter. Aim to send these as soon as your POS detects the state change.

---

## Step 5: Announce Products with PRODUCTS / PRODUCT

Before sending prices, you must announce your products and their categories to the server using the `PRODUCTS` / `PRODUCT` methods. This allows the server to correctly categorise fuel products (e.g. as diesel, petrol, LPG) independently of your internal product IDs or description strings. It also carries the **VAT rate** per product, which is required for correct payment processing.

> Although this is technically a protocol extension, treat it as mandatory — it will become part of the standard implementation requirements. If omitted, VAT rates cannot be conveyed via any other method.

### How it Works

`PRODUCTS` follows the same request/notification/OK pattern as `PUMPS` and `PRICES`. The server sends a `PRODUCTS` request, and your client responds with one `PRODUCT` notification per product, followed by `OK`:

```
Server: S2 PRODUCTS
Client: * PRODUCT 0100 ron98 19.0
Client: * PRODUCT 0200 ron95e10 19.0
Client: * PRODUCT 0300 ron95e5 19.0
Client: * PRODUCT 0400 diesel 19.0
Client: S2 OK
```

### PRODUCT Arguments

| Argument | Example | Required | Notes |
|---|---|---|---|
| ProductID | `0100` | ✓ | Must match the IDs used in `PRICE` and `TRANSACTION` |
| Category | `ron98` | ✓ | Standardised product type — see table below |
| VATRate | `19.0` | ✓ | VAT rate for this product in percent |
| Unit | `LTR` | optional | Required if OptionalName is provided |
| OptionalName | `Super Müller Diesel` | optional | Human-readable name; overrides the `Description` from `PRICE` if provided |

### Product Categories

Use the correct category from this list — do not use deprecated entries marked with `*`:

| Category | Description |
|---|---|
| `ron95e5` | RON 95 (up to 5% bio additives) |
| `ron95e10` | RON 95 (up to 10% bio additives) |
| `ron98` | RON 98 |
| `ron98e5` | RON 98 (up to 5% bio additives) |
| `ron100` | RON 100 |
| `e85` | Ethanol (85%) |
| `diesel` | Diesel |
| `dieselB0` | Diesel (0% bio additives) |
| `dieselB7` | Diesel (up to 7% bio additives) |
| `dieselPremium` | Premium Diesel |
| `dieselHvo` | HVO / CARE Diesel |
| `dieselGtl` | GTL/XTL Diesel (synthetic) |
| `dieselSynthetic` | Other synthetic Diesel (EN 15940) |
| `truckDiesel` | Diesel for commercial vehicles |
| `truckDieselPremium` | Premium Diesel for commercial vehicles |
| `lpg` | Liquefied petroleum gas |
| `truckLpg` | LPG for commercial vehicles |
| `cng` | Compressed natural gas |
| `lng` | Liquefied natural gas |
| `h2` | Hydrogen |
| `adBlue` | AdBlue (DEF / AUS 32) |
| `truckAdBlue` | AdBlue for commercial vehicles |
| `heatingOil` | Heating oil |

Do **not** use `careDiesel` or `syntheticDiesel` — these are deprecated. Use `dieselHvo` and `dieselSynthetic` respectively.

---

## Step 6: Implement PRICES

`PRICES` asks your client for the current price of every fuel product available at the site.

**Incoming request:**
```
S0 PRICES
```

**Your response — one `PRICE` notification per product, then `OK`:**
```
* PRICE 0100 LTR EUR 1.339 Super Plus
* PRICE 0200 LTR EUR 1.229 Super 95
* PRICE 0300 LTR EUR 1.499 Super 95 E5
S0 OK
```

### PRICE Arguments Explained

Each `PRICE` notification has the following fields:

| Argument | Example | Notes |
|---|---|---|
| ProductID | `0100` | Your internal product identifier |
| Unit | `LTR` | Always `LTR` (litres) for now |
| Currency | `EUR` | ISO 4217 currency code |
| PricePerUnit | `1.339` | End-user price **including VAT**, per litre |
| Description | `Super Plus` | Human-readable product name (variadic — may contain spaces) |

Note that the description is the last argument and is "variadic" — it can contain spaces, unlike all other arguments. Everything after the fourth space on a `PRICE` line is the description.

---

## Step 7: Implement Proactive PRICE Notifications

Just as with pump statuses, your client should send a `PRICE` notification proactively whenever a fuel price changes at your site — without waiting for the server to ask:

```
* PRICE 0200 LTR EUR 1.249 Super 95
```

This ensures that customers browsing the app before arriving at the forecourt always see up-to-date prices.

---

## Step 8: Update Your CAPABILITY Announcement

Now that you're implementing pump, price, and product mapping methods, update the `CAPABILITY` notification you send during the handshake (Part 1, Step 3) to include the new methods:

```
* CAPABILITY BEAT CHARSET PLAINAUTH PRICE PRODUCT PRODUCTS PUMP TRANSACTION LOCKEDPUMP QUIT
```

Added here: `PRICE`, `PRODUCT`, `PRODUCTS`, `PUMP`. (`TRANSACTION` and `LOCKEDPUMP` are for Part 3, but it's shown here for completeness.)

---

## Common Mistakes to Avoid

- **Sending `OK` before all notifications:** The server expects all `PUMP` or `PRICE` notifications to arrive before the `OK`. Sending `OK` early may cause the server to act on incomplete data.
- **Forgetting the UpdateTTL timer:** If you acknowledge a `PUMPSTATUS` with a TTL but then never send proactive updates, the server will have a stale view of that pump during a customer's active session.
- **Sending updates after TTL expires:** Once the TTL has elapsed, stop sending updates for that subscription unless a new `PUMPSTATUS` request renews it, or the status changes proactively.
- **Prices without VAT:** `PricePerUnit` in `PRICE` must always be the **end-user price including VAT**. Don't send the net price by mistake.

---

## Putting It Together — A Typical Post-Connection Exchange

Here is what the first few seconds after authentication typically look like:

```
→ Client authenticates (Part 1)

Server: S0 PRODUCTS
Client: * PRODUCT 0100 ron98 19.0
Client: * PRODUCT 0200 ron95e10 19.0
Client: S0 OK

Server: S1 PRICES
Client: * PRICE 0100 LTR EUR 1.339 Super Plus
Client: * PRICE 0200 LTR EUR 1.229 Super 95
Client: S1 OK

Server: S2 PUMPS
Client: * PUMP 1 in-use
Client: * PUMP 2 out-of-order
Client: * PUMP 3 free
Client: * PUMP 4 ready-to-pay
Client: S2 OK

→ Customer selects pump 3 in the app

Server: S3 PUMPSTATUS 3
Client: * PUMP 3 free
Client: S3 OK

Server: S4 PUMPSTATUS 3 30
Client: * PUMP 3 free
Client: S4 OK

→ Customer begins fueling

Client: * PUMP 3 in-use

→ Customer finishes fueling

Client: * PUMP 3 ready-to-pay

→ Continue to Part 3: Transactions
```

---

## What's Next

With pump statuses and prices handled, the server now has everything it needs to present the forecourt to customers. Part 3 will cover the transaction flows — how open transactions are reported, and how the server clears them after payment — for both the Post-Pay and Pre-Auth processes.
