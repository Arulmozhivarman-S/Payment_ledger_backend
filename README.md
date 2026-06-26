# Payment Ledger Service


LIVE URL   =   https://payment-ledger-backend-z6t2.onrender.com/


An event-driven money-movement backend built on a double-entry ledger.
Transfers are **idempotent**, **atomic**, and safe under concurrency — the
same primitives real payment systems are built on.

## Core guarantees

| Property | How |
|---|---|
| No double payment | `Idempotency-Key` header + UNIQUE constraint on `transfer.idempotency_key` |
| No double-spend / overdraw | `SELECT … FOR UPDATE` row locks (deterministic order) + JPA `@Version` |
| Money correctness | balances/amounts stored as `long` minor units (cents) — never `double` |
| Auditability | double-entry: every transfer writes one DEBIT + one CREDIT, append-only |
| Atomicity | debit, credit, ledger rows, transfer, and outbox event commit in one TX |
| Reliable events | transactional outbox + scheduled relay (at-least-once, idempotent consumer) |

## Stack
Java 17 · Spring Boot 4 · Spring Data JPA · MySQL · Lombok

## Run
```bash
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=secret -e MYSQL_DATABASE=finance-backend mysql:8
export DB_PASSWORD=secret
./mvnw spring-boot:run
```
JPA creates the tables on startup (`ddl-auto: update`). No auth — endpoints are open.

## API
```bash
# open two accounts
curl -X POST localhost:8080/accounts -H 'Content-Type: application/json' \
  -d '{"ownerUserId":1,"currency":"USD","openingBalanceMinor":10000}'
curl -X POST localhost:8080/accounts -H 'Content-Type: application/json' \
  -d '{"ownerUserId":2,"currency":"USD","openingBalanceMinor":0}'

# transfer $25 — run this EXACT command twice; money moves only once
curl -X POST localhost:8080/transfers \
  -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: order-abc-123' \
  -d '{"sourceAccountId":1,"destAccountId":2,"amountMinor":2500,"currency":"USD"}'

curl localhost:8080/accounts/2   # balanceMinor = 2500, not 5000
```

## Proof tests
```bash
./mvnw test -Dtest=TransferServiceConcurrencyTest
```
- `concurrentWithdrawals_neverOverdraw` — 20 threads, balance for one transfer → exactly one succeeds, balance never negative.
- `sameIdempotencyKey_movesMoneyOnce` — duplicate key credits the destination once.

## Layout
```
com.arul.finance_backend
├── FinanceBackendApplication.java
├── config/SchedulingConfig.java        # enables the outbox relay
└── ledger/
    ├── model/        Account, LedgerEntry, Transfer
    ├── repository/   row-locking account lookup, idempotency lookup
    ├── service/      TransferService (the core), AccountService
    ├── controller/   TransferController, AccountController
    ├── dto/ enums/ exception/
    └── outbox/        transactional outbox + scheduled relay
```
