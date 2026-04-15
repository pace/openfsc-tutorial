# OpenFSC Client Implementation Walkthrough
## Part 3: Transactions

This is the heart of the OpenFSC protocol — where fueling events become payments. Part 3 covers how transactions are reported, how the server clears them after payment, and the two distinct flows your client needs to support: **Post-Pay** and **Pre-Auth**.

---

## Step 1: Understand Transaction Lifecycle

A transaction is always `open` from the moment fuel is dispensed until either:
- The server sends a `CLEAR` (payment succeeded via OpenFSC), or
- The customer pays in the shop — in which case the transaction simply disappears from your POS and is no longer reported.

> The spec also defines a `deferred` transaction status, intended for cases where an operator manually releases a pump with an unsettled transaction. This has not been adopted in practice — do not implement it.

> **Important:** Only send a `TRANSACTION` notification if fuel was actually dispensed (non-zero volume). Zero-quantity events are not transactions — they are cancellations, handled differently in the Pre-Auth flow.

---

## Step 2: Implement TRANSACTIONS

`TRANSACTIONS` works exactly like `PUMPS` and `PRICES` from Part 2 — a bulk request answered with individual notifications. The server uses it to fetch all currently open transactions, typically right after authentication to catch up on anything unsettled.

**Incoming request (all pumps):**
```
S4 TRANSACTIONS
```

**Your response — one `TRANSACTION` notification per open transaction, then `OK`:**
```
* TRANSACTION 3 c71b9838ad3dfc15 open 0100 EUR 86.83 72.978 19.0 13.65 LTR 54.40 1.339
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

Just like `PUMPSTATUS`, `TRANSACTIONS` supports an `UpdateTTL` argument — but only when a specific pump is also specified:

```
S4 TRANSACTIONS 3 60
```

After responding with current transactions and `OK`, your client must proactively push `TRANSACTION` notifications for that pump whenever a transaction changes during the TTL window. The same TTL range applies: **30 to 300 seconds**. Respond with `ERR 416` if out of range.

---

## Step 3: Understand the TRANSACTION Notification Arguments

The `TRANSACTION` notification carries a lot of fields. Here is a breakdown:

| Position | Name | Example | Notes |
|---|---|---|---|
| arg0 | Pump | `3` | Pump number |
| arg1 | SiteTransactionID | `c71b9838ad3dfc15` | Your POS-defined transaction ID (Post-Pay), or the FSCTransactionID provided by UNLOCKPUMP (Pre-Auth) |
| arg2 | Status | `open` | Always `open` |
| arg3 | ProductID | `0100` | Fuel product identifier |
| arg4 | Currency | `EUR` | ISO 4217 |
| arg5 | PriceWithVAT | `86.83` | Total price including VAT |
| arg6 | PriceWithoutVAT | `72.978` | Total price excluding VAT |
| arg7 | VATRate | `19.0` | VAT percentage |
| arg8 | VATAmount | `13.65` | VAT amount (PriceWithVAT − PriceWithoutVAT) |
| arg9 | Unit | `LTR` | Always `LTR` |
| arg10 | Volume | `54.40` | Litres dispensed |
| arg11 | PricePerUnit | `1.339` | Price per litre including VAT |

Double-check your maths before sending: `PriceWithoutVAT × (1 + VATRate/100)` should equal `PriceWithVAT`, and `VATAmount` should equal `PriceWithVAT − PriceWithoutVAT`. Inconsistent values will cause issues on the payment side.

The **SiteTransactionID** differs between flows — this is covered in detail in the Post-Pay and Pre-Auth steps below.

---

## Step 4: The Post-Pay Flow

Post-Pay is the most common flow in Germany. The customer fuels first, and payment is collected afterward.

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
* TRANSACTION 3 c71b9838ad3dfc15 open 0100 EUR 86.83 72.978 19.0 13.65 LTR 54.40 1.339
```

**2. Keep reporting the transaction:**

If the server sends `TRANSACTIONS` (e.g. after a reconnect), include all currently open transactions in your response, including this one.

**3. When the server sends `CLEAR`:**

See Step 6 below.

---

## Step 5: The Pre-Auth Flow

Pre-Auth is used outside Germany and for unmanned stations. The payment is pre-authorized before fueling begins, and the pump is unlocked by the server.

### Flow Overview

```
Pump status: locked  ← pre-auth idle state (default for Pre-Auth pumps)
  ↓ server sends UNLOCKPUMP
Pump status: free  ← pump is now open for this specific customer to fuel
  ↓ customer starts fueling
Pump status: in-use
  ↓ customer finishes fueling
Pump status: locked  ← session still active, awaiting CLEAR (not ready-to-pay)
  ↓ client sends TRANSACTION (open, using FSCTransactionID)
  ↓ server sends CLEAR
Pump status: locked  ← back to pre-auth idle, available for next customer
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
| Currency | `EUR` | Currency of the credit |
| Credit | `100.00` | Maximum authorized amount |
| FSCTransactionID | `e2f74ef5-...` | Server-assigned transaction ID — store this, you'll need it |
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
| `412` | Pump is already locked |
| `422` | Currency unknown |

### Using FSCTransactionID

In the Pre-Auth flow, you do **not** generate your own SiteTransactionID. Instead, use the `FSCTransactionID` provided by `UNLOCKPUMP` as the `SiteTransactionID` in your `TRANSACTION` notification:

```
* TRANSACTION 3 e2f74ef5-f427-4ae6-bdd3-70a96709992f open 0100 EUR 86.83 72.978 19.0 13.65 LTR 54.40 1.339
```

This is how the server correlates the pre-authorization with the actual fueling transaction.

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
| `402` | Payment required — pump can't be locked due to a pending payment |
| `404` | Pump unknown |
| `423` | Already locked |

### Handling LOCKEDPUMP (Client-Side Cancellation)

If fueling was never started (zero fuel dispensed) and the Pre-Auth session needs to be cancelled from *your* side — for example due to a timeout or hardware issue — your client sends `LOCKEDPUMP`:

```
C7 LOCKEDPUMP 3 e2f74ef5-f427-4ae6-bdd3-70a96709992f aborted
```

The `Reason` argument must be one of:
- `aborted` — zero fueling / customer walked away
- `timeout` — the system aborted due to timeout

The server responds `OK` if accepted. Once you receive that acknowledgement, send a pump status update to return the pump to its pre-auth idle state:

```
Server: C7 OK
Client: * PUMP 3 locked
```

Or the server returns one of these errors:

| Code | Meaning |
|---|---|
| `404` | Invalid transaction and/or pump combination |
| `403` | Transaction is in invalid state and can't be cancelled |
| `400` | Invalid reason |

> Remember: `LOCKEDPUMP` is only for the case where **no fuel was dispensed**. If fuel was dispensed, always send a `TRANSACTION` notification instead, even if the amount was very small.

---

## Step 6: Implement CLEAR

`CLEAR` is the server's instruction to mark a transaction as paid and free the pump. It is used in both the Post-Pay and Pre-Auth flows.

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
2. Free the pump.
3. Reply `OK`.
4. Send a proactive pump status update:

```
S5 OK
* PUMP 3 free
```

**Important:** Mark cleared transactions in your reconciliation lists with "Clearance source: Connected Fueling" and the PaymentMethod used. This is required for back-office reconciliation with the payment operator.

### CLEAR Error Handling

| Code | When to return | Meaning to the server |
|---|---|---|
| `404` | Pump and SiteTransactionID are unknown | Clearing was too long ago, client no longer knows the transaction → triggers back-office process |
| `403` | Payment method not permitted | Customer paid in the shop instead → server cancels the payment transaction |
| `410` | Transaction not open, FSCTransactionID matches | The initial `CLEAR` was already processed successfully → transaction confirmed |

### Handling CLEAR After Reconnection

This is one of the trickier edge cases. If your client disconnects while processing a `CLEAR`, the transaction may be in an ambiguous state when you reconnect. The server will retry the `CLEAR` after reconnection. Here is how to respond:

- **Transaction is still `open`:** The original `CLEAR` was never received. Return `OK` and clear it now as normal.
- **Transaction is already `cleared`:** Return `ERR 410` — the server will understand the payment went through.
- **Transaction is unknown (cleared too long ago):** Return `ERR 404` — this triggers a back-office process.
- **Transaction was paid in the shop:** Return `ERR 403` — the server will cancel its payment transaction.

Build this logic into your `CLEAR` handler from the start, not as an afterthought.

---

## Step 7: Update Your CAPABILITY Announcement

Make sure all newly implemented methods are reflected in your `CAPABILITY` notification:

```
* CAPABILITY BEAT CHARSET PLAINAUTH PRICE PRODUCT PRODUCTS PUMP TRANSACTION LOCKEDPUMP QUIT
```

---

## Complete Flow Examples

### Post-Pay (Full Example)

```
→ Authentication complete (Part 1)

Server: S0 PRICES
Client: * PRICE 0100 LTR EUR 1.339 Super Plus
Client: S0 OK

Server: S1 PUMPS
Client: * PUMP 1 free
Client: * PUMP 2 free
Client: * PUMP 3 in-use
Client: * PUMP 4 ready-to-pay
Client: S1 OK

Server: S2 PUMPSTATUS 3 30
Client: * PUMP 3 in-use
Client: S2 OK

→ Customer finishes fueling on pump 3

Client: * PUMP 3 ready-to-pay
Client: * TRANSACTION 3 c71b9838ad3dfc15 open 0100 EUR 86.83 72.978 19.0 13.65 LTR 54.40 1.339

Server: S3 TRANSACTIONS
Client: * TRANSACTION 3 c71b9838ad3dfc15 open 0100 EUR 86.83 72.978 19.0 13.65 LTR 54.40 1.339
Client: S3 OK

→ Customer pays via app

Server: S4 CLEAR 3 c71b9838ad3dfc15 e2f74ef5-f427-4ae6-bdd3-70a96709992f pace
Client: S4 OK
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
Client: * TRANSACTION 3 e2f74ef5-f427-4ae6-bdd3-70a96709992f open 0100 EUR 86.83 72.978 19.0 13.65 LTR 54.40 1.339

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

- **Using your own SiteTransactionID in Pre-Auth:** In Pre-Auth, the `SiteTransactionID` in your `TRANSACTION` notification must be the `FSCTransactionID` from `UNLOCKPUMP`. Using a different ID will break the server's ability to correlate the authorization with the transaction.
- **Sending TRANSACTION for zero-volume fuelings:** If no fuel was dispensed, use `LOCKEDPUMP` instead.
- **Not handling the CLEAR retry after reconnect:** Implement all four `CLEAR` response cases (normal, 404, 403, 410) from day one.
- **Forgetting reconciliation marking:** Transactions cleared via OpenFSC must be flagged as such in your reconciliation lists. This is an operational requirement, not just a nice-to-have.
- **Sending LOCKEDPUMP when fuel was dispensed:** Even a tiny amount of dispensed fuel means a `TRANSACTION` must be reported. `LOCKEDPUMP` is strictly for zero-dispense cancellations.

---

## What's Next

You now have everything needed to implement a fully functional OpenFSC client for both Post-Pay and Pre-Auth flows. Part 4 covers advanced topics — Connection Multiplexing, High Availability, and Transaction Information.
