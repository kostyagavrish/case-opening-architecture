# Case opening + Provably Fair: how I would build it

**Goal:** charge exactly once, and nobody can influence the result, including us.

---

## 1. Where it runs

```mermaid
flowchart LR
    U(["User"]) -- "open case" --> API["API (stateless)"]
    API -- "one transaction" --> PG[("Postgres<br/>source of truth")]
    API -. "after commit, best effort" .-> RD[("Redis<br/>live feed, counters")]
    RD -. "pub/sub" .-> WS["WebSocket"] -.-> U
```

- **Postgres is the source of truth.** Redis can lose an event without losing money.
- **Money is a ledger** in integer cents: every movement is a posting with a unique key.

---

## 2. The request: one transaction

```mermaid
sequenceDiagram
    autonumber
    participant C as Client
    participant API as API
    participant DB as Postgres

    C->>API: POST /cases/42/open + Idempotency-Key
    API->>DB: BEGIN
    API->>DB: insert idempotency key (exists = replay)
    API->>DB: lock user's seed pair FOR UPDATE
    API->>DB: load case version (price, ranges)
    API->>DB: ledger debit, key case_open/{id}
    API->>API: ticket = roll(serverSeed, clientSeed, nonce)
    API->>DB: insert opening + user item
    API->>DB: nonce = nonce + 1
    API->>DB: COMMIT
    API-->>C: item (animation is only for show)
```

---

## 3. Provably Fair

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant S as Server

    S->>S: serverSeed = 32 random bytes (secret)
    S->>U: sha256(serverSeed) = commitment
    U->>S: clientSeed (user's choice)
    loop each opening
        S->>U: result for nonce 0, 1, 2 ...
    end
    U->>S: rotate seed
    S->>U: reveal old serverSeed
    U->>U: verify hash, recompute every roll
```

```mermaid
flowchart LR
    A["HMAC_SHA256(serverSeed,<br/>clientSeed:nonce)"] --> B["first 52 bits"]
    B --> C["ticket 1..100000"]
    C --> D{"range"}
    D -- "1 - 60000" --> E["Blue $0.50"]
    D -- "60001 - 95000" --> F["Purple $3"]
    D -- "95001 - 99900" --> G["Pink $25"]
    D -- "99901 - 100000" --> H["Knife $400"]
```

---

## 4. Data model (minimal)

| Table | Key columns | Constraints / notes |
|---|---|---|
| `seed_pairs` | `id`, `user_id`, `server_seed_enc`, `server_seed_hash`, `client_seed`, `nonce`, `status (active\|revealed)`, `revealed_at` | **Partial unique** `(user_id) WHERE status = 'active'`. The server seed is encrypted at rest. |
| `cases` / `case_versions` | `case_versions`: `id`, `case_id`, `price_cents`, `total_tickets`, `house_edge_bps`, `published_at` | A version is **immutable**. Changing odds or price = a new version. |
| `case_version_items` | `case_version_id`, `skin_id`, `ticket_from`, `ticket_to`, `value_cents_at_publish` | Ranges cover `1..total_tickets` with no gaps and no overlap (checked on publish). |
| `case_openings` | `id`, `user_id`, `case_version_id`, `seed_pair_id`, `nonce`, `client_seed`, `ticket`, `skin_id`, `price_cents`, `created_at` | **Unique** `(seed_pair_id, nonce)`: a nonce can never be used twice. Partitioned by month. |
| `user_items` | `id`, `user_id`, `skin_id`, `status`, `source_type`, `source_id`, `value_cents_at_acquire` | A site-inventory item is a **claim** on a skin, not a real Steam item yet. |
| ledger: `ledger_postings`, `ledger_entries`, `account_balances` | posting `key`, e.g. `case_open/{openingId}` | Posting key **unique**. Integer cents. User balance `CHECK (>= 0)`. |
| `idempotency_keys` | `user_id`, `key`, `response_status`, `response_body` | **Unique** `(user_id, key)`. |

### Key constraints

| Constraint | Why |
|---|---|
| One `active` seed pair per user | One nonce stream per user |
| `unique (seed_pair_id, nonce)` | A nonce can never be used twice |
| Immutable case versions | Every opening references the odds it used |
| `unique (key)` on postings | One opening = at most one debit |
| `CHECK (balance >= 0)` | The DB itself refuses overspending |

---

## 5. What can go wrong

| Risk | Example | Protection |
|---|---|---|
| **Charged twice** | Double click, mobile retry after a timeout | Idempotency key replays the first response, and the posting key is unique |
| **Parallel overspend** | 5 tabs open cases at once with a $10 balance | Seed-pair `FOR UPDATE` = per-user mutex, plus `CHECK balance >= 0` |
| **Re-roll** | Retry until a good result | Deterministic roll, and **the nonce advances only on commit**: a retry gives the same result |
| **Predictable rolls** | Weak randomness, a leaked seed | `crypto.randomBytes`, seed encrypted at rest, never logged, rotate on suspicion |
| **Odds changed silently** | Shrink the knife's range | Immutable public case versions. **PF proves the roll, not the odds.** |
| **House edge drift** | A skin in the case doubles in price | EV monitor + alert, fix = publish a new version |

