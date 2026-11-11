# 3. System Requirements

Priority: **High** = required for the project to work; **Medium** = expected, but the system
works without it; **Low** = nice to have.

## 3.1. Functional Requirements

### Core functions

| ID | Function Name | Description | Priority |
|---|---|---|---|
| F1 | User registration | The user can create an account with an email address and a password (minimum 8 characters). Each email can be registered only once. | High |
| F2 | User login | The user can log in with email and password and receives an access token used for all further requests. | High |
| F3 | Logout | The user can log out; the token and cached personal data are removed from the device. | Medium |
| F4 | Simulated wallet top-up | The user can add virtual PLN to the wallet (1–10,000 PLN per operation). The screen clearly states that the funds are simulated. | High |
| F5 | Current exchange rates | The user can view current average rates of selected currencies (EUR, USD, GBP, CHF, CZK, NOK, SEK, JPY) from NBP table A. | High |
| F6 | Historical exchange rates | The user can view the rate history of a selected currency for the last 7, 30 or 90 days as a chart. | Medium |
| F7 | Buy currency | The user can buy a selected currency with PLN. The PLN balance decreases and the currency balance increases. | High |
| F8 | Sell currency | The user can sell a currency they own for PLN. The currency balance decreases and the PLN balance increases. | High |
| F9 | Operation confirmation | Before a buy or sell, the app shows the amount, the rate and the total in PLN, and the user must confirm. | High |
| F10 | Wallet balances | The user can view the balance of every currency in their wallet. | High |
| F11 | Transaction history | The user can view a list of their operations (top-up, buy, sell) with date, currencies, amounts, rate and status. | High |
| F12 | Rate freshness information | Every displayed rate shows the NBP table date and the time the app obtained it; cached data is labelled "cached". | High |

### Mobile-network functions

| ID | Function Name | Description | Priority |
|---|---|---|---|
| F13 | Offline state | The app detects loss of connection and shows a visible "You're offline" banner; the banner disappears when the connection returns. | High |
| F14 | Offline read access | While offline, the user can view cached rates, wallet balances and transaction history, each with its "last updated" time. | High |
| F15 | Blocking changes offline | While offline, top-up, buy and sell are disabled, with a short message explaining why. Nothing is queued. | High |
| F16 | Uncertain operation status | If a top-up, buy or sell gets no response, the app shows "Checking status…" instead of success or failure, and resolves the status with the server when possible. | High |
| F17 | Synchronisation after reconnection | When the connection returns, the app first resolves any uncertain operation, then refreshes balances, history and rates from the server. | High |
| F18 | Manual refresh | The user can refresh data with a pull-to-refresh gesture. | Medium |

## 3.2. Non-Functional Requirements

| ID | Name | Description | Category |
|---|---|---|---|
| N1 | Response time | On the normal test connection (phone and server on the same Wi-Fi), 95% of API requests complete within 2 s, measured over 20 requests per endpoint type. | Performance |
| N2 | Request timeout | Every request from the app has a timeout of 10 s. After the timeout the request is treated as failed and the interface shows a clear message (reads) or the "Checking status…" state (data-changing actions). The service uses a 5 s timeout for calls to the NBP API. | Network resilience |
| N3 | Retry policy – reads | Read requests (rates, balances, history) are retried automatically at most 2 times (3 attempts in total), after 1 s and then 3 s, only after a timeout, a network error or a 5xx response. 4xx responses are not retried. Cached data stays visible during retries. | Network resilience |
| N4 | Retry policy – data-changing requests | A top-up, buy or sell is never sent as a new operation automatically. After a timeout or network error the app resends the **same** request (same request ID) up to 3 times while online, after 2 s, 5 s and 10 s. If the status is still unknown, the operation stays "Checking status…" and is resolved at the next reconnection (F17). | Network resilience |
| N5 | Duplicate-request protection | Every data-changing request carries a unique request ID (UUID v4) created on the phone before the first attempt. The service stores it with a UNIQUE constraint. A repeated request with the same ID returns the original result and creates no second operation. | Consistency |
| N6 | Atomic operations | All database changes of one operation (both wallet balances and the transaction record) are made in one database transaction: either all are saved or none. | Consistency |
| N7 | Balance integrity | A wallet balance can never become negative. The balance check and update happen inside the same transaction, so overlapping requests cannot spend the same money twice. | Consistency |
| N8 | Rate consistency | The app sends the rate it showed to the user. If the service's current rate differs, the service rejects the operation with a "rate changed" response and the app asks the user to confirm the new rate. The rate used is stored with each transaction. | Consistency |
| N9 | Monetary precision | Amounts are stored as whole numbers of the smallest unit (e.g. grosze, cents) and rates with 4 decimal places, to avoid floating-point rounding errors. | Data accuracy |
| N10 | Local cache | The app stores the last received rates, balances and history on the device together with the time they were obtained. Cached data is used for instant display and offline access. | Network resilience |
| N11 | Cache freshness | While online, the app refreshes data when it is opened, when it returns to the foreground, and when cached data is older than 15 minutes. | Network resilience |
| N12 | Synchronisation rule | The server is the authoritative source. After reconnection (within 5 s of the connection being detected) server data replaces the local cache; local data never overwrites server data. | Network resilience |
| N13 | Offline detection time | The offline banner appears within 5 s of the device losing its connection. | Usability |
| N14 | External API unavailability | If the NBP API is unavailable or no table is published that day (weekends, holidays), the service returns the last stored table together with its date. | Reliability |
| N15 | Authentication and authorisation | Passwords are stored only as bcrypt hashes. All wallet and transaction endpoints require a valid token (valid for 24 h), and a user can access only their own data. | Security |
| N16 | Input validation | The service validates every data-changing request: supported currency, positive amount, at most 2 decimal places, top-up limits, sufficient balance. Invalid requests are rejected with a clear error message. | Security |
| N17 | Secret handling | No passwords, keys or token secrets are stored in the Git repository; configuration is provided in an `.env` file. | Security |
| N18 | Clear status labelling | The interface uses consistent labels for data and operation states: "offline", "cached · last updated …", "checking status…", "completed", "failed". | Usability |
| N19 | Platform support | The app runs on Android and iOS through Expo Go. | Portability |
| N20 | Reproducibility | The README contains all steps needed to install and run the service and the app, and to reproduce each network-failure test. | Maintainability |

### Justification of the chosen values

- **2 s response time (N1):** research on response times (Nielsen) shows that up to about
  1 s the user keeps their flow of thought, and up to about 10 s attention is held. The service
  runs on the same local network and NBP rates are cached on the server, so 2 s for 95% of
  requests is realistic and leaves a margin for a slower phone.
- **10 s timeout (N2):** about 10 s is the limit of the user's attention; waiting longer without
  a result feels broken. It is also 5× the normal response time (N1), so a slow but working
  connection is not treated as a failure too early.
- **2 retries with 1 s / 3 s waits for reads (N3):** reads do not change data, so retrying them
  is safe. Short, increasing waits handle brief signal loss without flooding a weak network.
  Cached data stays on screen, so the user is not blocked.
- **Same request ID for data-changing retries (N4, N5):** after a timeout the app cannot know
  whether the server never received the request or completed it but the response was lost.
  Reusing the request ID makes a repeat safe in both cases.
- **15 min cache age (N11):** NBP publishes table A once per working day, so rates change
  rarely. Refreshing every 15 minutes keeps data current with very few requests, which also
  saves battery and mobile data.
- **Blocking changes offline (F15):** a purchase queued offline would be executed later at a
  rate the user did not see. Blocking is simpler and safer (decision recorded in Lab 01).
