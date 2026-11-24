# OpenFSC Client Implementation Walkthrough
## Part 3: Transactions

This part is about turning fuelings into payments. It covers how your client reports transactions, how the server tells it that a transaction has been paid, and the two flows your client can support: **Post-Pay** and **Pre-Auth**.

---

## Step 1: Understand Transaction Lifecycle

A transaction is always `open` from the moment fuel is dispensed until either:
- The server sends a `CLEAR` (payment succeeded via OpenFSC), or
- The customer pays in the shop. In this case the transaction disappears from your POS and is no longer reported.

> The spec also defines a `deferred` transaction status, meant for cases where an operator manually releases a pump with an unpaid transaction. It is not used in practice, so do not implement it.

> **Important:** Only send a `TRANSACTION` notification if fuel was actually dispensed (non-zero volume). A fueling with zero volume is not a transaction but a cancellation, which the Pre-Auth flow handles differently.

---

## Step 2: Implement TRANSACTIONS

`TRANSACTIONS` works exactly like `PUMPS` and `PRICES` from Part 2: one request, answered with one notification per item. The server uses it to fetch all open transactions, for example right after your client logs in, to catch up on anything still unpaid.

**Incoming request (all pumps):**
```
S4 TRANSACTIONS
```

**Your response: one `TRANSACTION` notification per open transaction, then `OK`:**
```
* TRANSACTION 3 c71b9838ad3dfc15 open 0100 EUR 86.83 72.97 19.0 13.86 LTR 64.85 1.339
S4 OK
```

If there are no open transactions, just send `OK` with no notifications:
```
S4 OK
```

### TRANSACTIONS for a Single Pump

The server may also ask for transactions on a specific pump only:

```
S4 TRANSACTIONS 2
```

Respond with only the transactions for that pump (or just `OK` if none), then `OK`. Reply with `ERR 404 Pump unknown` if the pump number is invalid.

### The UpdateTTL Subscription

Just like `PUMPSTATUS`, `TRANSACTIONS` supports an `UpdateTTL` argument, but only together with a pump number:

```
S4 TRANSACTIONS 3 60
```

After answering with the current transactions and `OK`, your client must send a `TRANSACTION` notification for that pump whenever a transaction changes during the TTL. The same TTL range applies: **30 to 300 seconds**. Respond with `ERR 416` if out of range.

---

## Step 3: Understand the TRANSACTION Notification Arguments

The `TRANSACTION` notification carries a lot of fields. Here is a breakdown:

| Position | Name | Example | Notes |
|---|---|---|---|
| arg0 | Pump | `3` | Pump number (1 to 99) |
| arg1 | SiteTransactionID | `c71b9838ad3dfc15` | Your POS-defined transaction ID (Post-Pay), or the FSCTransactionID provided by UNLOCKPUMP (Pre-Auth) |
| arg2 | Status | `open` | Always `open` |
| arg3 | ProductID | `0100` | Fuel product identifier |
| arg4 | Currency | `EUR` | One of the supported currencies (see Part 2, Step 3) |
| arg5 | PriceWithVAT | `86.83` | Total price including VAT |
| arg6 | PriceWithoutVAT | `72.97` | Total price excluding VAT |
| arg7 | VATRate | `19.0` | VAT percentage |
| arg8 | VATAmount | `13.86` | VAT amount (PriceWithVAT − PriceWithoutVAT) |
| arg9 | Unit | `LTR` | Always `LTR` |
| arg10 | Volume | `64.85` | Liters dispensed |
| arg11 | PricePerUnit | `1.339` | Price per liter including VAT. At most 4 integer digits and 1 to 3 fractional digits; whole values need a fractional part (e.g. `589.0`) |

Double-check your math before sending: `PriceWithoutVAT × (1 + VATRate/100)` should equal `PriceWithVAT`, `VATAmount` should equal `PriceWithVAT − PriceWithoutVAT`, and `Volume × PricePerUnit` should equal `PriceWithVAT`. Results must agree within one cent. Inconsistent values cause problems with the payment.

The **SiteTransactionID** differs between the flows. The Post-Pay and Pre-Auth steps below explain how.

---

## Step 4: The Post-Pay Flow

Post-Pay is the most common flow in Germany. The customer fuels first and pays afterward.

### Flow Overview

```
Pump status: free
  ↓ customer starts fueling
Pump status: in-use
  ↓ customer finishes fueling
Pump status: ready-to-pay
  ↓ client sends TRANSACTION (open)
  ↓ server sends CLEAR
Pump status: free
```

### Your Responsibilities

**1. When the pump reaches `ready-to-pay`:**

Proactively send a `PUMP` status notification (Part 2), and a `TRANSACTION` notification with status `open`. Use your own POS-generated transaction ID as `SiteTransactionID`:

```
* PUMP 3 ready-to-pay
* TRANSACTION 3 c71b9838ad3dfc15 open 0100 EUR 86.83 72.97 19.0 13.86 LTR 64.85 1.339
```

**2. Keep reporting the transaction:**

If the server sends `TRANSACTIONS` (e.g. after a reconnect), include all currently open transactions in your response, including this one.

**3. When the server sends `CLEAR`:**

See Step 6 below.

---

## Step 5: The Pre-Auth Flow

Pre-Auth is common outside Germany and at unmanned sites. The customer authorizes a maximum amount before fueling, and the server then tells your client to unlock the pump.

### Flow Overview

```
Pump status: locked  ← idle state of a Pre-Auth pump
  ↓ server sends UNLOCKPUMP
Pump status: free  ← pump is now open for this specific customer to fuel
  ↓ customer starts fueling
Pump status: in-use
  ↓ customer finishes fueling
Pump status: locked  ← session still active, awaiting CLEAR (not ready-to-pay)
  ↓ client sends TRANSACTION (open, using FSCTransactionID)
  ↓ server sends CLEAR
Pump status: locked  ← back to pre-auth idle, available for the next mobile reservation / next Pre-Auth session
```

### Handling UNLOCKPUMP

The server sends `UNLOCKPUMP` to authorize fueling on a specific pump with a credit limit:

```
S5 UNLOCKPUMP 3 EUR 100.00 e2f74ef5-f427-4ae6-bdd3-70a96709992f pace
```

Arguments you need to handle:

| Argument | Example | Notes |
|---|---|---|
| Pump | `3` | Pump to unlock |
| Currency | `EUR` | Currency of the credit, one of the supported currencies (see Part 2, Step 3) |
| Credit | `100.00` | Maximum authorized amount |
| FSCTransactionID | `e2f74ef5-...` | Transaction ID assigned by the server. Store it, you'll need it later |
| PaymentMethod | `pace` | Payment method in use |
| ProductID1–8 | *(optional)* | If present, only unlock these specific nozzles |

**Your response:**

If the pump was successfully unlocked, reply:
```
S5 OK
```

Then update the pump status:
```
* PUMP 3 free
```

**Errors to return:**

| Code | Meaning |
|---|---|
| `403` | Payment method not permitted |
| `404` | Pump or ProductID unknown |
| `412` | Pump is in use: it was already unlocked or is currently used by someone else (e.g. a customer paying at the POS or with another payment method). The pump was not unlocked. |
| `422` | Currency unknown |

### Leaving Headroom for Pump Overrun

The `Credit` in `UNLOCKPUMP` is the maximum amount the customer has authorized. Many pumps do not stop the instant the preset limit is reached: valve closing time and hose pressure typically lead to a small overrun. If you preset the pump to exactly the `Credit`, the customer can end up with more fuel than was authorized.

We therefore recommend that you **preset the pump to a slightly lower limit than the `Credit`**. The `Credit` itself remains the authorized maximum; only the limit you program into the pump is reduced.

Two common approaches:

| Approach | Example (`Credit` 100.00) | Notes |
|---|---|---|
| Percentage | 2–3 % → pump limit 97.00–98.00 | Scales with the authorized amount. Recommended as the default. |
| Absolute amount | 1–2 EUR → pump limit 98.00–99.00 | Simple, but proportionally large for small credits. |

Guidelines:

- Choose the margin per site or pump type, based on how far your pumps typically overrun. Make it configurable.
- Make sure the reduced limit stays meaningful for small credits (for example, never reduce it to zero or below the pump's minimum preset). With an absolute margin, consider applying it only above a minimum credit.
- Only reduce the limit you send to the pump. Do not change anything in your `OK` reply to `UNLOCKPUMP`.
- Always report the amount actually dispensed in the `TRANSACTION` notification, even if the pump still overran and the total ended up above the margin you reserved.

### Using FSCTransactionID

In the Pre-Auth flow, you do **not** generate your own SiteTransactionID. Instead, use the `FSCTransactionID` provided by `UNLOCKPUMP` as the `SiteTransactionID` in your `TRANSACTION` notification:

```
* TRANSACTION 3 e2f74ef5-f427-4ae6-bdd3-70a96709992f open 0100 EUR 86.83 72.97 19.0 13.86 LTR 64.85 1.339
```

This is how the server links the authorized amount to the actual fueling.

### Handling LOCKPUMP (Server-Side Cancellation)

The server may cancel a Pre-Auth session before fueling begins by sending `LOCKPUMP`:

```
S6 LOCKPUMP 3
```

Cancel the pending authorization on your side and respond:
```
S6 OK
* PUMP 3 locked
```

Errors to return:

| Code | Meaning |
|---|---|
| `402` | Payment required: the pump can't be locked because a payment is pending |
| `404` | Pump unknown |
| `423` | Already locked |

### Handling LOCKEDPUMP (Client-Side Cancellation)

If no fuel was dispensed and *your* side needs to cancel the Pre-Auth session (for example after a timeout or a hardware problem), your client sends `LOCKEDPUMP`:

```
C7 LOCKEDPUMP 3 e2f74ef5-f427-4ae6-bdd3-70a96709992f aborted
```

The `Reason` argument must be one of:
- `aborted`: no fuel dispensed, e.g. the customer left
- `timeout`: the system canceled the session after a timeout

The server responds `OK` if it accepts the cancellation. Then send a pump status update to return the pump to its Pre-Auth idle state:

```
Server: C7 OK
Client: * PUMP 3 locked
```

Or the server returns one of these errors:

| Code | Meaning |
|---|---|
| `404` | Invalid transaction and/or pump combination |
| `403` | The transaction is in a state where it can't be canceled |
| `400` | Invalid reason |

> Remember: `LOCKEDPUMP` is only for the case where **no fuel was dispensed**. If fuel was dispensed, always send a `TRANSACTION` notification instead, even if the amount was very small.

---

## Step 6: Implement CLEAR

With `CLEAR`, the server tells your client that a transaction has been paid, so your client marks it as paid and releases the pump. `CLEAR` is used in both flows. Afterward, Post-Pay pumps return to `free` and Pre-Auth pumps return to `locked` (their idle state).

**Incoming request:**
```
S5 CLEAR 3 c71b9838ad3dfc15 e2f74ef5-f427-4ae6-bdd3-70a96709992f pace
```

Arguments:

| Argument | Example | Notes |
|---|---|---|
| Pump | `3` | Pump number |
| SiteTransactionID | `c71b9838ad3dfc15` | The transaction ID as reported by your client |
| FSCTransactionID | `e2f74ef5-...` | The server-side transaction ID |
| PaymentMethod | `pace` | Payment method used |

**On success:**
1. Mark the transaction as cleared in your POS.
2. Reply `OK`.
3. Send a pump status update. The status to report depends on the flow:
   - **Post-Pay:** return the pump to `free`. It is now available for any customer.
   - **Pre-Auth:** return the pump to `locked`. This is the Pre-Auth idle state and shows that the pump is available for the next customer using the app.

```
S5 OK
* PUMP 3 free   ← Post-Pay
```
```
S5 OK
* PUMP 3 locked   ← Pre-Auth
```

**Important:** Mark cleared transactions in your reconciliation lists with "Clearance source: Connected Fueling" and the PaymentMethod used. The fuel brand (MOC) needs this to reconcile the site's transactions.

### CLEAR Error Handling

| Code | When to return | Meaning to the server |
|---|---|---|
| `404` | Pump and SiteTransactionID are unknown | The transaction is too old and your client no longer knows it → the server starts a manual back-office process |
| `403` | Payment method not permitted; or, on a retried `CLEAR`, the transaction was paid by other means (usually in the shop) | In the retry case the server cancels its payment transaction |
| `410` | Transaction not open, FSCTransactionID matches | The initial `CLEAR` was already processed successfully → transaction confirmed |

### Handling CLEAR After Reconnection

This is one of the harder edge cases. If the connection drops while your client is handling a `CLEAR`, the server doesn't know whether the `CLEAR` arrived. It therefore sends the `CLEAR` again after your client reconnects. Respond like this:

- **Transaction is still `open`:** The original `CLEAR` was never received. Return `OK` and clear it now as normal.
- **Transaction is already cleared:** Return `ERR 410`. The server then knows the payment went through.
- **Transaction is unknown (cleared too long ago):** Return `ERR 404`. This starts a manual back-office process.
- **Transaction was paid in the shop:** Return `ERR 403`. The server then cancels its payment.

Build this logic into your `CLEAR` handler from the start.

---

## Step 7: Update Your CAPABILITY Announcement

Make sure all newly implemented methods are reflected in your `CAPABILITY` notification. For Post-Pay, the list from Part 2 is already complete:

```
* CAPABILITY CLEAR HEARTBEAT PRICES PRODUCTS PUMPS PUMPSTATUS QUIT TRANSACTIONS
```

If you also implement Pre-Auth, add `LOCKPUMP` and `UNLOCKPUMP`:

```
* CAPABILITY CLEAR HEARTBEAT LOCKPUMP PRICES PRODUCTS PUMPS PUMPSTATUS QUIT TRANSACTIONS UNLOCKPUMP
```

`TRANSACTION` and `LOCKEDPUMP` are notifications/requests your client *sends*, so they are not part of your list.

---

## Complete Flow Examples

### Post-Pay (Full Example)

```
→ Authentication complete (Part 1)

Server: S0 PRODUCTS
Client: * PRODUCT 0100 ron98 19.0
Client: S0 OK

Server: S1 PRICES
Client: * PRICE 0100 LTR EUR 1.339 Super Plus
Client: S1 OK

Server: S2 PUMPS
Client: * PUMP 1 free
Client: * PUMP 2 free
Client: * PUMP 3 in-use
Client: * PUMP 4 ready-to-pay
Client: S2 OK

Server: S3 PUMPSTATUS 3 30
Client: * PUMP 3 in-use
Client: S3 OK

→ Customer finishes fueling on pump 3

Client: * PUMP 3 ready-to-pay
Client: * TRANSACTION 3 c71b9838ad3dfc15 open 0100 EUR 86.83 72.97 19.0 13.86 LTR 64.85 1.339

Server: S4 TRANSACTIONS
Client: * TRANSACTION 3 c71b9838ad3dfc15 open 0100 EUR 86.83 72.97 19.0 13.86 LTR 64.85 1.339
Client: S4 OK

→ Customer pays via app

Server: S5 CLEAR 3 c71b9838ad3dfc15 e2f74ef5-f427-4ae6-bdd3-70a96709992f pace
Client: S5 OK
Client: * PUMP 3 free
```

### Pre-Auth (Full Example)

```
→ Authentication complete, pumps and prices exchanged

→ Customer pre-authorizes payment in app

Server: S1 UNLOCKPUMP 3 EUR 100.00 e2f74ef5-f427-4ae6-bdd3-70a96709992f pace
Client: S1 OK
Client: * PUMP 3 free

→ Customer starts fueling

Client: * PUMP 3 in-use

→ Customer finishes fueling

Client: * PUMP 3 locked
Client: * TRANSACTION 3 e2f74ef5-f427-4ae6-bdd3-70a96709992f open 0100 EUR 86.83 72.97 19.0 13.86 LTR 64.85 1.339

→ Server captures pre-authorized amount

Server: S2 CLEAR 3 e2f74ef5-f427-4ae6-bdd3-70a96709992f e2f74ef5-f427-4ae6-bdd3-70a96709992f pace
Client: S2 OK
Client: * PUMP 3 locked

---

→ Alternative: Customer walks away without fueling

Server: S1 UNLOCKPUMP 3 EUR 100.00 e2f74ef5-f427-4ae6-bdd3-70a96709992f pace
Client: S1 OK
Client: * PUMP 3 free

Client: C2 LOCKEDPUMP 3 e2f74ef5-f427-4ae6-bdd3-70a96709992f aborted
Server: C2 OK
Client: * PUMP 3 locked
```

---

## Common Mistakes to Avoid

- **Using your own SiteTransactionID in Pre-Auth:** In Pre-Auth, the `SiteTransactionID` in your `TRANSACTION` notification must be the `FSCTransactionID` from `UNLOCKPUMP`. With a different ID, the server can't link the authorized amount to the transaction.
- **Sending TRANSACTION for zero-volume fuelings:** If no fuel was dispensed, use `LOCKEDPUMP` instead.
- **Not handling the CLEAR retry after reconnect:** Implement all four `CLEAR` response cases (normal, 404, 403, 410) from day one.
- **Forgetting reconciliation marking:** Transactions cleared via OpenFSC must be flagged as such in your reconciliation lists. This is a requirement, not an option.
- **Sending LOCKEDPUMP when fuel was dispensed:** Even a tiny amount of dispensed fuel means a `TRANSACTION` must be reported. `LOCKEDPUMP` is strictly for zero-dispense cancellations.

---

## What's Next

You now have everything you need to build a complete OpenFSC client for Post-Pay and Pre-Auth. Part 4 covers optional extensions: Connection Multiplexing and Transaction Information. High Availability will be documented later.

**Previous:** [Part 2: Products, Prices & Pump Status](part-2.md) · **Next:** [Part 4: Advanced Topics](part-4.md)
