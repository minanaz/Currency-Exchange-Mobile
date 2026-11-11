# Architecture Sketch (Lab 01)

Initial end-to-end architecture of the Mobile Currency Exchange System.
This is a first sketch. It will be refined into the full architecture and data-flow
diagram for the M1 report.

## 1. Components

```mermaid
flowchart LR
    subgraph PHONE["📱 Mobile app (React Native + Expo)"]
        UI["Screens<br/>login, rates, wallet,<br/>buy/sell, history"]
        NET["Network layer<br/>timeout, retry,<br/>request ID"]
        CACHE[("Local cache<br/>AsyncStorage<br/>rates, balances, history<br/>+ last update time")]
        CONN["Connectivity detector<br/>online / offline"]
        UI --> NET
        UI <--> CACHE
        CONN --> UI
    end

    WIRELESS{{"📶 Wireless segment<br/>Wi-Fi / LTE<br/>UNRELIABLE"}}

    subgraph SERVER["🖥️ Web service (Node.js + Express)"]
        AUTH["Auth<br/>register, login, token"]
        RATES["Rates module<br/>calls NBP, caches rates"]
        EXCHANGE["Exchange module<br/>validation, buy/sell,<br/>duplicate check"]
    end

    DB[("🗄️ SQLite database<br/>users, wallets,<br/>transactions, rates cache")]
    NBP["🌐 NBP API<br/>api.nbp.pl"]

    NET <-->|"HTTP + JSON"| WIRELESS
    WIRELESS <-->|"HTTP + JSON"| SERVER
    AUTH <--> DB
    EXCHANGE <--> DB
    RATES <--> DB
    RATES <-->|"HTTPS, wired"| NBP
```

| Component | Role |
|---|---|
| Mobile app | User interface. Shows data, sends requests, keeps a read-only local cache. Never decides the result of a buy/sell. |
| Wireless segment | Phone ⇄ access point / base station. The unreliable part: loss, delay, timeouts, disconnections. |
| Web service | Business logic: authorisation, validation, exchange operations, NBP integration, duplicate-request protection. |
| Database | Permanent storage. The authoritative record of users, wallet balances and transactions. |
| NBP API | External source of current and historical exchange rates. |

## 2. Request path: "Buy 100 EUR"

```mermaid
sequenceDiagram
    autonumber
    participant App as 📱 Mobile app
    participant Air as 📶 Wireless segment
    participant API as 🖥️ Web service
    participant DB as 🗄️ Database

    App->>App: Create unique requestId
    App->>Air: POST /exchange/buy {EUR, 100, requestId}
    Note over Air: ⚠️ Failure point A:<br/>request lost, server never receives it
    Air->>API: request arrives
    API->>DB: requestId already used?
    alt New request
        API->>DB: BEGIN transaction<br/>check PLN balance<br/>PLN −, EUR +<br/>save transaction + requestId<br/>COMMIT
    else Duplicate request
        DB-->>API: return the saved first result
    end
    API->>Air: 200 OK + new balances
    Note over Air: ⚠️ Failure point B:<br/>server committed, but response lost
    Air->>App: response arrives
    App->>App: Show success, update cache
```

**Why it matters:** after a timeout the app can't tell failure point A from failure point B.
The app therefore retries with the **same requestId**. In case A the server performs
the operation for the first time. In case B the server recognises the ID and returns the
earlier result, so the money is not exchanged twice.

## 3. Data owners

| Data | Authoritative owner | Copies | Notes |
|---|---|---|---|
| User accounts, password hashes | Web service DB | – | Passwords stored only as hashes |
| Wallet balances | Web service DB | Phone cache (read-only, with timestamp) | Phone never edits balances itself |
| Transactions | Web service DB | Phone cache (history, read-only) | Each one stores its requestId (UNIQUE) |
| Exchange rates | NBP | Server rates cache → phone cache | Shown with "rate from" date and time |
| Auth token | Web service (issues it) | Phone secure storage | Sent with every request |
| requestId of a pending operation | Phone (creates it) | Server stores it with the transaction | Lets the server detect repeats |
