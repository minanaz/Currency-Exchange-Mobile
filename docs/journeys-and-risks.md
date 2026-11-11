# Lab 01 – User Journeys and Initial Risk List

## 1. Normal user journeys

### N1 – New user registers and checks rates

1. User opens the app and registers with email and password.
2. Service creates the account and an empty wallet; the app stores the auth token.
3. User tops up the wallet with **1000 PLN (simulated)**.
4. Service records the top-up and returns the new PLN balance.
5. User opens the Rates screen. The app shows current NBP rates (EUR, USD, GBP, CHF…)
   with the date of the NBP table and the time the data was fetched.
6. User opens EUR and sees a chart of rates from the last 30 days.

**Result:** account exists, wallet shows 1000 PLN, rates are visible with a freshness label.

### N2 – User buys currency and checks history

1. User selects EUR - Buy - enters 100 EUR.
2. App shows a confirmation with the rate and total cost in PLN.
3. User confirms. The app creates a `requestId` and sends the request.
4. Service checks the balance and, in one database transaction, decreases PLN,
   increases EUR, and saves the transaction with its `requestId`.
5. Service responds with the new balances and the rate used.
6. App shows "Purchase completed" and updates the wallet and history.

**Result:** exactly one transaction, PLN and EUR balances updated consistently.

## 2. Journeys interrupted by connectivity loss

### I1 – No connection when opening the app (cached display)

1. User opens the app on the metro with no signal.
2. App detects that it is offline and shows a banner: "You're offline".
3. App shows the **cached** rates, balances and history with the label
   "Last updated: 11.11.2026 14:05".
4. Buy, Sell and Top up buttons are disabled, with a short explanation.
5. Signal returns. The app detects it, hides the banner and refreshes rates,
   balances and history from the service.

**Expected outcome:** user always sees useful data marked as cached; no data-changing
action can be started offline; data is refreshed automatically after reconnection.

### I2 – Connection lost after tapping "Buy" (uncertain outcome)

1. User confirms "Buy 100 EUR". The app sends the request with `requestId = abc-123`.
2. Connection drops. No response arrives before the timeout.
3. The app **does not** show success or failure. It shows:
   "Connection lost – checking the status of your purchase…"
4. Two possibilities the app cannot tell apart:
   - **A:** the service never received the request;
   - **B:** the service completed the purchase, but the response was lost.
5. When the connection returns, the app resends the request with the **same**
   `requestId = abc-123`.
   - Case A - the service performs the purchase now (first time).
   - Case B - the service finds `abc-123` already saved and returns the original result.
6. App shows "Purchase completed" and refreshes the wallet and history.

**Expected outcome:** exactly one transaction in both cases; success is shown only
after the service confirms it.

## 3. Initial risk list

Likelihood / Impact: L = low, M = medium, H = high.

| ID  | Risk                                                                                     | Likelihood | Impact | Planned mitigation                                                                                |
| --- | ---------------------------------------------------------------------------------------- | ---------- | ------ | ------------------------------------------------------------------------------------------------- |
| R1  | **Stale data:** cached rates or balances are shown as if current                         | H          | M      | Always show "last updated" time and an offline/cached label; refresh on reconnect                 |
| R2  | **Uncertain outcome after a lost response:** user doesn't know if the purchase happened  | M          | H      | `requestId` per operation; retry with same ID; "checking status" state instead of success/failure |
| R3  | **Duplicate operation:** double tap or automatic retry creates two purchases             | M          | H      | `requestId` stored with UNIQUE constraint; button disabled while the request is pending           |
| R4  | **Rate changes** between viewing and buying                                              | M          | M      | Service uses its current rate and returns it; confirmation screen shows the rate used             |
| R5  | **Overlapping requests** (two quick operations) drive a balance below zero               | L          | H      | Balance check and update inside one database transaction; balance can never be negative           |
| R6  | **Failure partway through** an operation (crash after PLN decrease, before EUR increase) | L          | H      | All changes of one operation in a single atomic transaction (all or nothing)                      |
| R7  | **NBP API unavailable** or no table published (weekend, holiday)                         | M          | M      | Service caches the last available table and returns its date                                      |
| R8  | **Slow connection** makes the app look frozen                                            | H          | M      | Loading indicators; request timeout; reads retried automatically                                  |
| R9  | **Auth token expires** while the user is offline                                         | M          | L      | On reconnect, an expired token leads to the login screen; cached data stays visible               |
| R10 | **Demo environment:** phone can't reach the server on the laptop (Wi-Fi, firewall)       | M          | H      | Test the setup early; document the steps in the README                                            |
| R11 | **New technologies** (Node.js, Express, SQL) slow down progress for a solo developer     | M          | M      | Build a small working server early (before M2); commit progress weekly                            |

## 4. Decision: data-changing actions while offline

**Block:** Buy, sell and top-up are disabled while the app is offline.
The user sees a short message explaining why.

**Reason:** a queued purchase would be executed later at a different rate than the
user saw, which could confuse the user and cause an unexpected result. Read-only data
(rates, balances, history) stays available offline from the local cache.
