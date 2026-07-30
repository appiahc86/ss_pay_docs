# Smart Serve Pay API — Normal User Integration Guide

This document covers the API surface for **normal users** (the `user` account type — i.e. wallet
holders in the mobile app). It does not cover admin, merchant, or field-agent endpoints.

> Status: this describes the API as currently implemented in the codebase, including a few rough
> edges and inconsistencies called out inline as **⚠️ Note**. Where behavior looks unfinished or
> risky, that's flagged so the mobile team doesn't build against assumptions the backend doesn't
> actually guarantee yet.
>

## Base URL

https://server.smartservepay.com

| Mount path       | Router file                                              |
|------------------|-----------------------------------------------------------|
| `/auth`          | `src/routes/auth/authRouter.js`                          |
| `/profile`       | `src/routes/profile/profileRouter.js`                    |
| `/transactions`  | `src/routes/transactions/transactionsRouter.js`          |
| `/wallets`       | `src/routes/transactions/wallet/walletRouter.js`         |
| `/wallet-loads`  | `src/routes/walletRouter.js` (only the `/users/*` and `/paystack/webhook` sub-paths apply to normal users) |

## Authentication

All endpoints except `/auth/*` and the Paystack webhook require a JWT bearer token:

```
Authorization: Bearer <token>
```

The token is issued by `/auth/register` or `/auth/login` and expires after **24 hours**. It encodes
`userId`, `userType` ("user"), `phone`, and `cardNumber`. There is no refresh-token endpoint —
when the token expires, the app must send the user back through `/auth/login`.

If the token is missing/invalid/expired, or the account's `status` is not `active` (e.g.
`suspended`), requests get:

```json
{ "success": false, "message": "Access token is required" }        // 401, no token
{ "success": false, "message": "Invalid token" }                     // 401
{ "success": false, "message": "Token has expired" }                 // 401
{ "success": false, "message": "Account is not active" }             // 403
```

## Response envelope

**Most** endpoints return:

```json
{ "success": true|false, "message": "...", "data": { ... } }
```

but a few return a doubly-nested shape instead — **check the shape documented per-endpoint below**,
don't assume it's always the same:

```json
{ "data": { "success": true, "data": { ... } } }
```

(This affects `GET /profile`, `PUT /profile`, and everything under `/wallet-loads/users/*`.)

## Validation errors

Endpoints validator return HTTP 400 with:

```json
{ "success": false, "message": "Validation failed", "errors": ["Phone number is required", "..."] }
```

## Rate limiting

- Global: 100 requests / 15 min per IP (all routes).
- `/auth/register`, `/auth/login`, `/auth/reset-pin`: 5 attempts / 15 min per IP.
- `/auth/request-otp`: 3 requests / 5 min per IP.

Exceeding a limit returns HTTP 429 with `{ "success": false, "message": "..." }`.

## Business constants relevant to users

| Constant | Value | Notes |
|---|---|---|
| PIN length | 4 digits, numeric | Hashed with bcrypt, never returned by the API |
| Card number | 10 digits | Auto-generated at registration (8 random digits + 2-digit year suffix) |
| Min transaction amount | GHS 1.00 | Enforced on transfer and wallet load |
| Max transaction amount | GHS 10,000.00 | Enforced on transfer and wallet load |
| Transfer fee | 1% of amount | Added on top of amount; sender pays `amount + fee` |
| OTP length | 6 digits | |
| OTP expiry | 30 minutes | |
| OTP resend delay | 50 seconds | Server returns this as `canResendAfter` (seconds) |
| Max failed login attempts | 5 | Account **locks for 15 minutes** after this |
| Max failed transaction-PIN attempts | 5 | Account is **suspended** (not just locked) after this — see §5.2 |

Login lockouts and transaction-PIN suspensions use **separate counters** (`failed_login_attempts`
vs. `pin_failed_attempts`) — a wrong login PIN no longer counts against the transfer PIN limit or
vice versa.

---

## 1. Auth (`/auth`)

None of these require a token.

### 1.1 Request OTP

`POST /auth/request-otp`

Rate limited: 3 / 5 min.

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone` | string | yes | Ghana number, any common format (`0244...`, `+233244...`, `233244...`) |
| `purpose` | string | yes | `registration` \| `login` \| `password_reset` |
| `user_type` | string | no | Send `"user"` for login/password_reset purposes so the server can check the account exists first |

For `purpose: "registration"`: fails with 409 if the phone is already registered.
For `purpose: "login"` / `"password_reset"` (with `user_type` set): fails with 404 if no account
exists for that phone, or **403** with a message like
`"Account is locked. Try again in 12 minutes."` if too many failed login attempts recently occurred.

Success (200):
```json
{ "success": true, "message": "OTP sent to +233***456", "expiresIn": 1800, "canResendAfter": 50 }
```

### 1.2 Verify OTP

`POST /auth/verify-otp`

| Field | Type | Required |
|---|---|---|
| `phone` | string | yes |
| `otp` | string, 6 digits | yes |
| `purpose` | string | yes — must match the purpose used in 1.1 |

Success (200):
```json
{ "success": true, "message": "OTP verified successfully", "otpId": 123 }
```

Failure (400) — `verificationResult.error` is one of `OTP_NOT_FOUND`, `OTP_EXPIRED`,
`MAX_ATTEMPTS_EXCEEDED`, `INVALID_OTP`:
```json
{ "success": false, "message": "Invalid OTP code. 2 attempts remaining.", "error": "INVALID_OTP" }
```

Keep the returned `otpId` — it's required by `/auth/register` and `/auth/reset-pin`.

### 1.3 Register

`POST /auth/register`

Rate limited: 5 / 15 min. Requires a **verified** OTP for `purpose: "registration"` on the same phone.

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | 2–100 chars, letters/spaces/hyphens/apostrophes only |
| `phone` | string | yes | Same number the OTP was verified against |
| `pin` | string, 4 digits | yes | Becomes the transaction PIN |
| `otpId` | number | yes | From step 1.2 |

Success (201):
```json
{
  "success": true,
  "message": "Registration successful",
  "data": {
    "user": {
      "id": 1, "name": "Ama Owusu", "phone": "+233244123456",
      "cardNumber": "1234567826", "balance": "0.00", "status": "active",
      "created_at": "2026-07-23T...", "phone_verified": true, "userType": "user"
    },
    "token": "<jwt>"
  }
}
```

### 1.4 Login

`POST /auth/login`

Rate limited: 5 / 15 min.

| Field | Type | Required | Notes |
|---|---|---|---|
| `phone` | string | yes | |
| `user_type` | string | yes | Send `"user"` |
| `credential` | string | yes | The 4-digit PIN |

Success (200):
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "user": {
      "id": 1, "name": "Ama Owusu", "phone": "+233244123456", "status": "active",
      "cardNumber": "1234567826", "balance": "120.50",
      "created_at": "...", "last_login_at": "...", "phone_verified": true
    },
    "token": "<jwt>"
  }
}
```

Errors: 404 account not found, 403 account not active, 400 invalid PIN/credentials, or
**403** with `"Account is locked. Try again in N minutes."` if 5 failed attempts were made recently.

### 1.5 Reset PIN

`POST /auth/reset-pin`

Rate limited: 5 / 15 min. Flow: call 1.1 with `purpose: "password_reset"` and `user_type: "user"`,
then 1.2 with the same purpose to get a verified `otpId`, then call this.

| Field | Type | Required |
|---|---|---|
| `phone` | string | yes |
| `newPin` | string, 4 digits | yes |
| `otpId` | number | yes — must reference a verified OTP |

Success (200): `{ "success": true, "message": "PIN reset successfully" }`

> `POST /auth/reset-password` also exists on this router but is only for `merchant` / `field_agent` /
> `admin` account types — normal users don't have a separate password, only the PIN above.

---

## 2. Profile (`/profile`, auth required, `userType: user`)

### 2.1 Get profile

`GET /profile`

⚠️ **Note the doubly-nested envelope** (`data.data`, not `data`):
```json
{
  "data": {
    "success": true,
    "data": {
      "user": {
        "name": "Ama Owusu", "phone": "+233244123456", "status": "active",
        "createdAt": "...", "lastLoginAt": "...", "failedLoginAttempts": 0,
        "balance": 120.5, "accountType": "User", "cardNumber": "1234567826"
      }
    }
  }
}
```

### 2.2 Update profile

`PUT /profile`

| Field | Type | Required |
|---|---|---|
| `name` | string | yes (only field currently supported) |

`400` if `name` is missing/blank. Success (200) returns the updated profile, same shape as 2.1:

```json
{
  "data": {
    "success": true,
    "data": {
      "user": {
        "name": "Ama Owusu Mensah", "phone": "+233244123456", "status": "active",
        "createdAt": "...", "lastLoginAt": "...", "balance": 120.5,
        "accountType": "User", "cardNumber": "1234567826"
      }
    }
  }
}
```

⚠️ **Note:** only `name` can be updated today; sending other fields (e.g. photo, gender) is
silently ignored server-side.

---

## 3. Wallet (`/wallets`, auth required, `userType: user`)

### 3.1 Get balance

`GET /wallets/balance`

```json
{ "success": true, "data": { "balance": 120.5, "cardNumber": "1234567826", "name": "Ama Owusu" } }
```


### 3.3 Wallet load history

`GET /wallets/loads?page=1&limit=10`

```json
{
  "success": true,
  "data": {
    "loads": [ { "id": 5, "user_id": 1, "payment_reference": "WL...", "amount": "50.00", "method": "payment_api", "status": "completed", "created_at": "..." } ],
    "pagination": { "currentPage": 1, "totalPages": 2, "totalLoads": 12, "limit": 10, "hasNextPage": true, "hasPrevPage": false }
  }
}
```

⚠️ **Note:** `method` is a DB column with a default of `payment_api` (enum: `payment_api` \|
`merchant_cash_in` \| `mobile_money`). The real top-up flow (§4.1, Paystack mobile money) never
sets it explicitly on insert, so every row created that way comes back as `method: "payment_api"`
here — not `"mobile_money"` — even though the underlying payment was a mobile money charge. Don't
rely on this field to distinguish payment channel today.

### 3.4 Payment methods

`GET /wallets/payment-methods`

```json
{
  "success": true,
  "data": {
    "paymentMethods": [
      {
        "id": "mobile_money", "name": "Mobile Money", "description": "Pay with MTN, Telecel, or AirtelTigo",
        "icon": "pi pi-mobile",
        "enabled": true,
        "providers": [
          { "code": "mtn", "name": "MTN Mobile Money", "icon": "mtn", "image": "/images/momo/mtn.png" },
          { "code": "vod", "name": "Telecel Cash", "icon": "telecel", "image": "/images/momo/telecel.png" },
          { "code": "tgo", "name": "AirtelTigo Money", "icon": "airteltigo", "image": "/images/momo/airteltigo.webp" }
        ]
      }
    ],
    "balance": 170.5
  }
}
```


### 3.5 Verify payment

`GET /wallets/verify-payment/:reference`

`:reference` is the `payment_reference` of a wallet load previously created by `POST /wallets/load`
(§3.2) — looks it up for the authenticated user and reports its real, current status (no longer a
stub with hardcoded values).

```json
{
  "success": true,
  "message": "Payment verified successfully",
  "data": { "verified": true, "status": "completed", "amount": 50, "reference": "WL..." }
}
```

`message`/`verified` reflect `status` (`pending` \| `completed` \| `failed`). `404` if the
reference doesn't exist for this user.

This checks the same `wallet_loads` table as §4.2 — for Paystack-initiated mobile money top-ups
use `GET /wallet-loads/users/payment-status/:reference` (§4.2) instead, since that's the reference
returned by the Paystack initialize call.

---

## 4. Mobile Money Wallet Load via Paystack (`/wallet-loads/users`, auth required, `userType: user`)

This is the real top-up flow: server initiates a Paystack mobile money charge, the user approves
the charge prompt on their phone (MTN/Telecel/AirtelTigo), and Paystack calls a server-side webhook
when it resolves.

### 4.1 Initialize a mobile money charge

`POST /wallet-loads/users/initialize`

| Field | Type | Required | Notes |
|---|---|---|---|
| `amount` | number | yes | 1.00–10,000.00 |
| `phone` | string | yes | Exactly 10 digits, local format (e.g. `0244123456`) — the mobile money number to charge |
| `provider` | string | yes | `mtn` \| `vod` \| `tgo` |

A `transactions` row is created up front with `status: "pending"`, and a `wallet_loads` row tracks
the Paystack reference. The wallet balance is **not** updated yet.

Success (200), note the doubly-nested envelope:
```json
{ "data": { "success": true, "data": { "reference": "IOC-..." } } }
```

The mobile app should prompt the user to approve the mobile money charge on their phone (Paystack
sends the USSD/approval prompt directly), then poll 4.2 until `status` is `completed` or `failed`.

⚠️ **Dev-only quirk:** outside of `production mode`, the phone number actually sent to Paystack
is hardcoded to a test MSISDN — the `phone` you pass is ignored in non-prod environments.

### 4.2 Check payment status

`GET /wallet-loads/users/payment-status/:reference`

`reference` is the value returned from 4.1.

```json
{ "data": { "success": true, "data": { "status": "pending", "balance": 220.5 } } }
```

`status` is one of `pending` \| `completed` \| `failed` (mirrors the underlying `transactions.status`).
`balance` is the user's actual current wallet balance at the time of the call (a DB trigger credits
the wallet automatically the moment the underlying transaction flips to `completed`, so there's no
need to add the payment amount on top — poll this endpoint and read `balance` directly once
`status` is `completed`).

⚠️ **Note:** if `reference` doesn't match any transaction, this comes back as **HTTP 400** (not 404)
with `{ "success": false, "message": "Payment Reference not found" }`.


---

## 5. Transactions (`/transactions`, auth required, `userType: user`)

### 5.1 Look up a recipient by card number

`GET /transactions/lookup-recipient/:cardNumber`

`cardNumber` must be 10 digits and not the caller's own card number. Use this to confirm who the
user is about to send money to before calling 5.2.

```json
{
  "success": true,
  "message": "Recipient found",
  "data": { "user": { "name": "Kojo Mensah", "phone": "+233...", "card_number": "9876543221", "status": "active", "photo_url": null, "institution": "Some University" } }
}
```

404-style errors come back as **HTTP 400** with `success: false` (not 404) if the card doesn't
exist, isn't active, or matches the caller's own card.

### 5.2 Transfer money

`POST /transactions/transfer`

| Field | Type | Required | Notes |
|---|---|---|---|
| `recipientCardNumber` | string, 10 digits | yes | |
| `amount` | number | yes | 1.00–10,000.00 |
| `description` | string | no | max 100 chars |
| `pin` | string, 4 digits | yes | Sender's transaction PIN |

A 1% fee is added: `total_amount = amount + fee`, deducted from the sender; the recipient receives
`amount` (fee-free).

Success (200):
```json
{
  "success": true,
  "message": "Transfer completed successfully",
  "data": {
    "transaction": {
      "id": 42, "transaction_id": "TXN...", "amount": 100, "fee": 1, "total_amount": 101,
      "recipient": { "name": "Kojo Mensah", "card_number": "9876543221" },
      "timestamp": "..."
    },
    "newBalance": 219.5
  }
}
```

Failure cases:
- `400 { message: "You cannot send money to yourself" }`
- `400 { message: "Insufficient balance (including fees)", data: { required, available, shortfall } }`
- `400 { message: "Invalid PIN" }`
- `400 { message: "Recipient not found or account not active" }`
- `401 { message: "Sender account not found or inactive" }`
- `401 { message: "Account suspended due to too many incorrect PIN attempts. Please contact support." }`

⚠️ **Note:** the transaction PIN has its own failed-attempt counter, separate from login (see
"Business constants" above). After **5** wrong transaction PINs in a row the account is suspended
(not just locked) — there's no self-service unlock for a suspended account today, so surface the
403/401 message above clearly and point the user to support. A correct PIN entry resets this
counter back to 0.

### 5.3 Transaction history

`GET /transactions/history?page=1&limit=10&type=user_transfer&status=completed`

All query params optional. `type` ∈ `user_transfer` \| `wallet_load` \| `wallet_to_card`.
`status` ∈ `pending` \| `completed` \| `failed` \| `cancelled`. `limit` max 50.

```json
{
  "success": true,
  "data": {
    "transactions": [ { "id": 42, "transaction_id": "TXN...", "type": "user_transfer", "amount": "100.00", "fee": "1.00", "status": "completed", "sender_name": "...", "sender_card_number": "...", "recipient_name": "...", "recipient_card_number": "...", "created_at": "..." } ],
    "walletBalance": "219.50",
    "pagination": { "currentPage": 1, "totalPages": 3, "totalTransactions": 25, "limit": 10, "hasNextPage": true, "hasPrevPage": false }
  }
}
```

Note: this list includes transactions where the user is either sender or recipient, so check
`sender_card_number` / `recipient_card_number` against the user's own card number client-side to
determine debit vs. credit direction and which "name" field is the counterparty.

### 5.4 Recent recipients

`GET /transactions/recent-recipients?limit=5`

Only includes people the user has sent money to (not received from), most recent first.

```json
{
  "success": true,
  "data": {
    "recipients": [ { "name": "Kojo Mensah", "cardNumber": "9876543221", "last_transfer": "...", "initials": "KM" } ],
    "currentBalance": 219.5
  }
}
```

### 5.5 Single transaction

`GET /transactions/:id`

`:id` is the numeric `transactions.id` (not `transaction_id`). Only returns the transaction if the
caller was the sender or recipient; otherwise `404`.

```json
{ "success": true, "data": { "transaction": { "id": 42, "transaction_id": "TXN...", "sender_name": "...", "sender_phone": "...", "recipient_name": "...", "recipient_phone": "...", "merchant_name": null, ... } } }
```

### 5.6 Transaction receipt

`GET /transactions/:id/receipt`

```json
{
  "success": true,
  "data": {
    "receipt": {
      "receiptNumber": "RCPTXN...", "transactionId": "TXN...", "timestamp": "...",
      "type": "user_transfer", "status": "completed", "amount": "100.00", "fee": "1.00", "totalAmount": "101.00",
      "description": "...", "sender": { "name": "...", "cardNumber": "..." }, "recipient": { "name": "...", "cardNumber": "..." }
    }
  }
}
```

### 5.7 Transaction stats summary

`GET /transactions/stats/summary?period=30`

`period` is a number of days (default 30).

```json
{
  "success": true,
  "data": {
    "period": "30 days",
    "statistics": { "totalTransactions": 12, "totalSent": 500, "totalReceived": 300, "totalFees": 5 }
  }
}
```

---

## Quick reference — recommended mobile flows

**Onboarding:** `POST /auth/request-otp` (purpose=registration) → `POST /auth/verify-otp` → `POST /auth/register` → store token.

**Login:** `POST /auth/request-otp`? *(not required for login unless you want an extra factor —
login itself only needs phone + PIN)* → `POST /auth/login` → store token.

**Forgot PIN:** `POST /auth/request-otp` (purpose=password_reset, user_type=user) →
`POST /auth/verify-otp` (purpose=password_reset) → `POST /auth/reset-pin`.

**Top up wallet (real money):** `GET /wallets/payment-methods` → `POST /wallet-loads/users/initialize` →
poll `GET /wallet-loads/users/payment-status/:reference` → on `completed`, refresh via
`GET /wallets/balance`.

**Send money:** `GET /transactions/lookup-recipient/:cardNumber` (confirm recipient) →
`POST /transactions/transfer` → refresh balance/history.

---
