# OpenFSC Client Implementation Walkthrough
## Reference

Lookup tables for the walkthrough:

- [CAPABILITY Lists](#capability-lists)
- [Product Categories](#product-categories)
- [Error Codes](#error-codes)

---

## CAPABILITY Lists

Each side lists the methods it **handles**. Build your list from the rows that apply to your client:

| Your client supports | Your client adds | The server adds | Part |
|---|---|---|---|
| Core (always) | `CLEAR` `HEARTBEAT` `PRICES` `PUMPS` `PUMPSTATUS` `QUIT` `TRANSACTIONS` | `BEAT` `CHARSET` `PLAINAUTH` `PRICE` `PUMP` `TRANSACTION` `LOCKEDPUMP` `QUIT` | 1 |
| Product Mapping (treat as mandatory) | `PRODUCTS` | `PRODUCT` | 2 |
| Pre-Auth | `LOCKPUMP` `UNLOCKPUMP` | (in the core list) | 3 |
| Transaction Information | `TRANSACTIONINFO` | `TRANSACTIONINFO` | 4 |
| Connection Multiplexing | nothing | `NEWSESSION` `SESSIONS` | 4 |

For example, a client with Post-Pay, Pre-Auth and Product Mapping sends:

```
* CAPABILITY CLEAR HEARTBEAT LOCKPUMP PRICES PRODUCTS PUMPS PUMPSTATUS QUIT TRANSACTIONS UNLOCKPUMP
```

---

## Product Categories

The `Category` argument of `PRODUCT` (Part 2, Step 2) must be one of these:

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

## Error Codes

An error always concludes a request and uses the request's tag:

```
<tag> ERR <code> <message>
```

The message is free text for humans. The server and your client only act on the code.

### General Error Codes

These codes apply to every request, in both directions.

| Code | Meaning |
|---|---|
| `400` | Bad request: invalid method, arguments or tag |
| `403` | Method used in the wrong connection state, e.g. an Authenticated State method before `PLAINAUTH` |
| `405` | Method unknown |
| `406` | Wrong encoding: the bytes don't match the current encoding |
| `500` | Internal error |

Answer unknown methods with `405` and keep the connection open (Part 1, Step 9).

---

### Errors Your Client Returns

The server sends these requests. Your client answers with `OK` or one of these codes.

| Request | Code | When | Details |
|---|---|---|---|
| `HEARTBEAT` | `422` | The timestamp is invalid | Part 1, Step 7 |
| `PUMPSTATUS` | `404` | Pump unknown | Part 2, Step 6 |
| | `416` | `UpdateTTL` outside 30 to 300 seconds | Part 2, Step 6 |
| `TRANSACTIONS` | `404` | Pump unknown | Part 3, Step 2 |
| | `416` | `UpdateTTL` outside 30 to 300 seconds | Part 3, Step 2 |
| `UNLOCKPUMP` | `403` | Payment method not permitted | Part 3, Step 5 |
| | `404` | Pump or ProductID unknown | Part 3, Step 5 |
| | `412` | Pump is in use (already unlocked or used by someone else). The pump was not unlocked | Part 3, Step 5 |
| | `422` | Currency unknown | Part 3, Step 5 |
| `LOCKPUMP` | `402` | The pump can't be locked because a payment is pending | Part 3, Step 5 |
| | `404` | Pump unknown | Part 3, Step 5 |
| | `423` | Already locked | Part 3, Step 5 |
| `CLEAR` | `403` | Payment method not permitted, or (on a retried `CLEAR`) the transaction was paid in the shop. The server then cancels its payment | Part 3, Step 6 |
| | `404` | Pump and SiteTransactionID unknown. The server starts a manual back-office process | Part 3, Step 6 |
| | `410` | Transaction already cleared with the same FSCTransactionID. The server treats the payment as confirmed | Part 3, Step 6 |

`PRODUCTS`, `PRICES` and `PUMPS` have no specific error codes. Only the general codes apply.

---

### Errors Your Client Receives

Your client sends these requests. The server answers with `OK` or one of these codes.

| Request | Code | Meaning | What to do |
|---|---|---|---|
| `CHARSET` | `404` | Encoding unknown | Continue in ASCII (Part 1, Step 4) |
| `PLAINAUTH` | `401` | SiteAccessKey and/or Secret not valid | Stop retrying, alert an operator (Part 1, Step 10) |
| | `409` | Multiplexing only: the session's priority is already taken for this site | Use another priority or none (Part 4) |
| | `410` | Multiplexing only: `PLAINAUTH` on the initial connection after `NEWSESSION` | Log in on the sessions only (Part 4) |
| `LOCKEDPUMP` | `400` | Invalid reason | Use `aborted` or `timeout` (Part 3, Step 5) |
| | `403` | The transaction is in a state where it can't be canceled | Check the transaction state (Part 3, Step 5) |
| | `404` | Transaction and pump don't match | Check the FSCTransactionID and pump (Part 3, Step 5) |
| `NEWSESSION` | `409` | SessionID already taken | Use another SessionID (Part 4) |
| | `410` | `NEWSESSION` after `PLAINAUTH` on the initial connection | Reconnect without logging in on the initial connection (Part 4) |
| `SESSIONS` | `410` | Called on a session instead of the initial connection | Send it without session prefix (Part 4) |

---

**Previous:** [Part 4: Advanced Topics](part-4.md) · **Back to:** [Overview](overview.md)
