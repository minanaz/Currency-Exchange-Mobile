# Mobile Currency Exchange System

Semester project for **Network Issues in Mobile Systems**.
Author: Amina ([@minanaz](https://github.com/minanaz))

A mobile app for **virtual** currency exchange. Users register, top up a simulated PLN wallet,
view current and historical exchange rates from the National Bank of Poland (NBP) API,
buy and sell currencies, and view wallet balances and transaction history.
The focus of the project is correct behaviour on an **unreliable mobile connection**:
offline state, timeouts, safe retries, local cache, synchronisation, and protection
against duplicate requests.

> ! All money in this system is simulated. No real payments are made.

---

## Project decision

| Item                 | Decision                                                                |
| -------------------- | ----------------------------------------------------------------------- |
| Topic                | **Project A – Mobile Currency Exchange System**                         |
| Team                 | Solo project (Amina)                                                    |
| External data source | NBP Web API – `https://api.nbp.pl` (exchange rate tables A, no API key) |

## Technology stack

| Component       | Technology                       | Reason                                                                          |
| --------------- | -------------------------------- | ------------------------------------------------------------------------------- |
| Mobile client   | React Native + Expo (JavaScript) | Prior experience; runs on a real phone via Expo Go                              |
| Web service     | Node.js + Express                | Same language as the client; simple REST API                                    |
| Database        | SQLite (via `better-sqlite3`)    | Single file, no server to install; supports transactions and UNIQUE constraints |
| Communication   | REST API, JSON over HTTP(S)      | Phone ⇄ Wi-Fi / mobile network ⇄ service                                        |
| Authentication  | Token-based (planned: JWT)       | Stateless; works with retries                                                   |
| Version control | Git + GitHub                     | Required by the course                                                          |

Design choices (timeouts, retry count, cache age) are documented in `docs/` and justified in the M1 report.

## Repository structure

```
currency-exchange-mobile/
├── mobile/      React Native + Expo app
├── server/      Node.js + Express web service
├── database/    SQL schema and migrations
└── docs/        Reports, lab notes, test evidence
    └── diagrams/  Editable UML, ERD and architecture diagrams
```

## Setup and running

> Filled in at milestone M2, when the components exist.

### Requirements

- Node.js 24 LTS and npm
- Git
- Expo Go app on an Android or iOS phone (same Wi-Fi network as the computer)

### Configuration

- `server/.env` – copied from `server/.env.example` (no secrets are committed)

### Run the web service

_TBD (M2)_

### Run the mobile app

_TBD (M2)_

## Network-failure demonstration

> Filled in at milestone M3.

How to reproduce: no connection, timeout, slow response, connection lost after submitting
a purchase, repeated request with the same identity, offline cache, restored connection
and synchronisation.

## Assumptions

- Exchange rates come from NBP table A (average rates), published on working days only.
  On weekends and holidays the latest available table is used.
- Buy and sell use the NBP average rate (no spread).
- Account funding is virtual and clearly marked as simulated in the app.
