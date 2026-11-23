# OpenFSC Client Implementation Walkthrough
## Part 2: Pump Status & Prices

Once your client is connected and logged in (Part 1), the server asks for your products and prices right away. It asks for pump statuses later, whenever a customer needs them. This part covers how to answer these requests, how to send updates without being asked, and how `UpdateTTL` asks your client to send status updates for a limited time.

---

## Step 1: Understand the Notification Pattern

Before looking at the individual methods, note a pattern that repeats throughout the protocol: **one request is answered with several notifications**.

When the server wants all pump statuses, it doesn't ask for one pump at a time. It sends a single request (`PUMPS`), and your client answers with one `PUMP` notification per pump, followed by a final `OK` that completes the request. The same pattern applies to prices (`PRICES` → multiple `PRICE` notifications → `OK`).

```
Server:  S1 PUMPS
Client:  * PUMP 1 free
Client:  * PUMP 2 in-use
Client:  * PUMP 3 ready-to-pay
Client:  S1 OK
```

Two things to note:
- The individual `PUMP` notifications use the `*` tag. They expect no reply.
- The `OK` uses the **server's tag** (`S1`) to complete *that specific request*.

Make sure all notifications are sent before you send the `OK`. The server considers the request complete once it receives the `OK`.

---

## Step 2: Implement PUMPS

`PUMPS` is the server's way of asking for the current status of every pump at your site.

**Incoming request:**
```
S1 PUMPS
```

**Your response: one `PUMP` notification per pump, then `OK`:**
```
* PUMP 1 free
* PUMP 2 out-of-order
* PUMP 3 in-use
* PUMP 4 ready-to-pay
S1 OK
```

### Pump Statuses

Pump numbers are integers from **1 to 99**.

Your POS system must map its internal pump states to one of these values:

| Status | Meaning |
|---|---|
| `free` | Pump is idle and available for use |
| `in-use` | Customer is currently fueling |
| `ready-to-pay` | Fueling complete, awaiting payment (Post-Pay) |
| `locked` | Pump is locked, which is the default status in the Pre-Auth workflow (a customer can reserve it in the app to start fueling) |
| `out-of-order` | Pump is not operational |

Think carefully about how each state in your POS maps to these. The most critical ones to get right are `ready-to-pay` (triggers payment flow in Post-Pay) and `locked` (indicates a Pre-Auth workflow).

> The spec also lists an `in-transaction` status, but it is deprecated. Do not implement or send it.

---

## Step 3: Implement PUMPSTATUS

`PUMPSTATUS` is the single-pump equivalent of `PUMPS`. The server uses it to ask about one specific pump, often when a customer has selected it in the app.

The pump argument is a pump number from 1 to 99.

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

`PUMPSTATUS` has an important optional second argument: `UpdateTTL` (time to live). It is a number of seconds. During this time, your client must **send a notification every time the status of that pump changes**.

**Incoming request with TTL:**
```
S3 PUMPSTATUS 3 30
```

Your response works the same way: send the current status, then `OK`:
```
* PUMP 3 free
S3 OK
```

But now, for (at least) the next 30 seconds, if pump 3 changes state, you must send a notification:
```
* PUMP 3 in-use
* PUMP 3 ready-to-pay
```

These are notifications with the `*` tag that your client sends on its own. No `OK` is needed for them.

Implementation notes:
- The valid TTL range is **30 to 300 seconds**. If the value is outside this range, reply with `ERR 416 UpdateTTL is too high or too low (valid range: 30 to 300, both inclusive)`.
- A new `PUMPSTATUS` request for the same pump restarts the timer.
- **Only send updates when the status actually changes.** Don't poll and send the same status repeatedly.
- Several pumps can have an active `UpdateTTL` at the same time, so track the timer per pump.

---

## Step 4: Implement Proactive PUMP Notifications

Your client doesn't need to wait to be asked. Whenever a pump's status changes, send a `PUMP` notification, whether there is an active `UpdateTTL` or not:

```
* PUMP 3 in-use
```

This keeps the server's view of your forecourt up to date. The server uses this data to update what customers see in the app, so send these updates as soon as your POS detects the change.

---

## Step 5: Announce Products with PRODUCTS / PRODUCT

Before sending prices, you must tell the server which products you sell and what kind of fuel each one is, using the `PRODUCTS` / `PRODUCT` methods. This lets the server categorize your products correctly (e.g. as diesel, petrol, LPG), whatever your internal product IDs and names are. It also carries the **VAT rate** per product, which is needed for payment.

> Formally this is a protocol extension, but treat it as mandatory: it will become part of the standard requirements, and there is no other way to send VAT rates.

### How it Works

The server asks for `PRODUCTS` before it asks for `PRICES`, so that it knows which products the following `PRICE` notifications refer to.

`PRODUCTS` follows the same request/notification/OK pattern as `PUMPS` and `PRICES`. The server sends a `PRODUCTS` request, and your client responds with one `PRODUCT` notification per product, followed by `OK`:

```
Server: S2 PRODUCTS
Client: * PRODUCT 0100 ron98 19.0
Client: * PRODUCT 0200 ron95e10 19.0 LTR
Client: * PRODUCT 0300 ron95e5 19.0
Client: * PRODUCT 0400 diesel 19.0
Client: S2 OK
```

### PRODUCT Arguments

| Argument | Example | Required | Notes |
|---|---|---|---|
| ProductID | `0100` | ✓ | Must match the IDs used in `PRICE` and `TRANSACTION` |
| Category | `ron98` | ✓ | Standardized product type (see table below) |
| VATRate | `19.0` | ✓ | VAT rate for this product in percent |
| Unit | `LTR` | optional | Required if OptionalName is provided |
| OptionalName | `Super Müller Diesel` | optional | Human-readable name; overrides the `Description` from `PRICE` if provided |

`Unit` may be sent without `OptionalName` (e.g. `PRODUCT 0200 ron95e10 19.0 LTR`), but `OptionalName` can only be sent together with `Unit`.

### Product Categories

Use the matching category from this list:

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

Do **not** use `careDiesel` or `syntheticDiesel`. They are deprecated: use `dieselHvo` and `dieselSynthetic` instead.

---

## Step 6: Implement PRICES

`PRICES` asks your client for the current price of every fuel product available at the site.

**Incoming request:**
```
S0 PRICES
```

**Your response: one `PRICE` notification per product, then `OK`:**
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
| Unit | `LTR` | Always `LTR` (liters) for now |
| Currency | `EUR` | One of the supported currencies listed below |
| PricePerUnit | `1.339` | End-user price **including VAT**, per liter. At most 4 integer digits and 1 to 3 fractional digits; whole values need a fractional part (e.g. `589.0`, not `589`) |
| Description | `Super Plus` | Product name as shown to the customer. May contain spaces (see below) |

The supported currencies are listed in the spec's [Supported Currencies](https://github.com/pace/openfsc-spec/blob/main/1.0/openfsc-spec.md#416-supported-currencies) (4.1.6): `EUR`, `CHF`, `GBP`, `DKK`, `SEK`, `NOK`, `PLN`, `CZK`, `HUF`, `RON`, `ISK`, `RSD`, `MKD`, `MDL`, `UAH`, `TRY`, `BAM`, `ALL` (others on request).

**Use one currency per site:** send the same `Currency` in all `PRICE`, `TRANSACTION` and `UNLOCKPUMP` messages of a site. The spec does not mandate this yet, but it is a strong recommendation and may become a requirement in a later spec version.

The description is the last argument and, unlike all other arguments, may contain spaces. Everything after `PricePerUnit` is the description.

---

## Step 7: Implement Proactive PRICE Notifications

Just as with pump statuses, your client should send a `PRICE` notification whenever a fuel price changes at your site, without waiting for the server to ask:

```
* PRICE 0200 LTR EUR 1.249 Super 95
```

This way, customers who check the app before arriving at the site always see current prices.

---

## Step 8: Update Your CAPABILITY Announcement

Now that you're implementing pump, price, and product mapping methods, update the `CAPABILITY` notification you send during the handshake (Part 1, Step 3) to include the new method:

```
* CAPABILITY CLEAR HEARTBEAT PRICES PRODUCTS PUMPS PUMPSTATUS QUIT TRANSACTIONS
```

Added here: `PRODUCTS`. `PRICES`, `PUMPS` and `PUMPSTATUS` are part of the core set from Part 1, and you implement them in this part. (`CLEAR` and `TRANSACTIONS` are for Part 3, but they are shown here for completeness.)

Remember that `CAPABILITY` lists the methods your client *handles*, i.e. the server's requests. `PRICE`, `PUMP` and `PRODUCT` are notifications your client *sends*, so they don't belong in your list.

---

## Common Mistakes to Avoid

- **Sending `OK` before all notifications:** The server expects all `PUMP` or `PRICE` notifications to arrive before the `OK`. Sending `OK` early may cause the server to act on incomplete data.
- **Forgetting the UpdateTTL timer:** If you acknowledge a `PUMPSTATUS` with a TTL but then never send proactive updates, the server will have a stale view of that pump during a customer's active session.
- **Treating the TTL as the end of status updates:** The TTL only guarantees updates during its window. Proactive status changes (Step 4) are always sent, also before a `PUMPSTATUS` request arrives and after its TTL has elapsed.
- **Prices without VAT:** `PricePerUnit` in `PRICE` must always be the **end-user price including VAT**. Don't send the net price by mistake.

---

## Putting It Together: A Typical Exchange After Connecting

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

→ A customer opens the site in the app

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

With products, prices and pump statuses handled, the server has everything it needs to show your site to customers. Part 3 covers the transaction flows for Post-Pay and Pre-Auth: how your client reports open transactions, and how the server tells it that a transaction has been paid.

**Previous:** [Part 1: Connection Handling](part-1.md) · **Next:** [Part 3: Transactions](part-3.md)
