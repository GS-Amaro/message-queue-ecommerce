# message-queue-ecommerce — PRD

Build spec for the team. Prose in English; **all identifiers in Portuguese**, so
the code matches the graded documents.

The scenario, architecture and design rationale are **not repeated here** — they
live in the delivered documents, which are the single source of truth:

| Document | Owns |
|---|---|
| [`etapa1.md`](etapa1.md) | Scenario, why messaging |
| [`etapa2.md`](etapa2.md) | Producers/consumers, message flow, **topology**, scalability, reliability, fault tolerance, design decisions |
| [`diagramas/`](diagramas) | 5 diagrams (`.mmd` + `.png`) |

This PRD owns what those do not: naming, use cases, implementation rules, code
layout, user stories, and the plan.

| Delivery | Content | Due | Points | Status |
|---|---|---|---|---|
| Stages 1–2 | Scenario + Architecture | 2026-09-17 23:59 | 2.5 | **done** |
| Stages 3–5 | Config + Use cases + Technical | 2026-09-24 | 3.5 | to do |
| Presentation | 10 min, every member presents | 2026-09-24 | 4.0 | to do |

---

## 1. Names

One table, so nothing drifts. Topology arguments are in [`etapa2.md` §2.2](etapa2.md).

| Kind | Identifiers |
|---|---|
| Exchanges | `pedidos` · `retry` · `dlx` |
| Queues | `pagamento` · `estoque` · `notificacao` · `retry.q` · `dlq` |
| Routing keys | `pedido.criado` · `pagamento.aprovado` · `pagamento.recusado` · `estoque.reservado` · `estoque.insuficiente` |
| Order status | `CRIADO` → `PAGO` → `CONCLUIDO`, or `RECUSADO` / `SEM_ESTOQUE` |
| Tables | `pedidos` · `produtos` · `processados` |
| HTTP | `POST /pedidos` · `GET /pedidos/:id` |
| Binary modes | `/app api` · `/app worker` |
| Env | `AMQP_URL` · `DB_URL` · `FAIL_RATE` (0–1, forces handler errors) |

**Envelope** — every message:

```json
{
  "id": "uuid",
  "tipo": "pagamento.aprovado",
  "pedido_id": "uuid",
  "data": { }
}
```

`id` is the idempotency key. Timestamp and content-type come from the AMQP
properties — do not duplicate them in the body.

---

## 2. Security (Stage 3)

- Users `api` and `worker` via `definitions.json`; `guest` removed.
- Least privilege: `api` writes only to the `pedidos` exchange; `worker` reads the
  queues and writes to the exchanges.
- Dedicated vhost `/loja`.
- Encryption: TLS config (port 5671) documented in `etapa3.md`, **not enabled** —
  a self-signed certificate on localhost demonstrates certificate generation, not
  security. Declared as a limitation.

---

## 3. Use Cases (Stage 4)

Each with input → processing → output, reproducible by command.

| # | Case | Command | Expected output | US |
|---|---|---|---|---|
| 1 | Happy path | `curl -X POST localhost:8080/pedidos -d @pedido.json` | Status reaches `CONCLUIDO`; 3 events in the logs | US1, US3 |
| 2 | Payment declined | Repeat until it hits the ~15% | Status `RECUSADO`, stock untouched | US3 |
| 3 | Insufficient stock | Order more than `produtos.disponivel` | Status `SEM_ESTOQUE` | US3 |
| 4 | Fan-out | One `pagamento.aprovado` | Appears in both `estoque` and `notificacao` logs | US4 |
| 5 | Retry | `FAIL_RATE=1 docker compose up worker` | Same message reprocessed every 10 s | US5 |
| 6 | DLQ | Same as above, after 3 attempts | Message visible in `dlq` in the Management UI | US5 |
| 7 | Consumer down | `docker compose stop worker`, create orders, restart | Queue piles up, then drains | US2 |
| 8 | Scaling | `docker compose up --scale worker=3` | 3 consumers per queue in the Management UI | US6 |
| 9 | Idempotency | Republish the same `id` | `disponivel` drops only once | US7 |

---

## 4. Implementation Rules

Non-negotiable — these are what the reliability and scalability claims in
`etapa2.md` rest on.

1. Manual ack **after** the database write, never before. No `autoAck` anywhere.
2. Publisher confirms — without a broker ACK, the publish counts as failed.
3. Durable queues; messages with `delivery_mode: 2`.
4. Idempotency before the side effect, same transaction:
   `INSERT INTO processados (id) VALUES ($1) ON CONFLICT DO NOTHING`.
5. `prefetch = 10` — without it one worker swallows the queue and scaling is invisible.
6. Stock decrement is a **single atomic statement**, never read-then-write:
   `UPDATE produtos SET disponivel = disponivel - $1 WHERE id = $2 AND disponivel >= $1`.
   `RowsAffected == 0` means insufficient stock.
7. Status updates are **monotonic** — the `UPDATE` only applies if the new status
   ranks later in the state machine. Competing consumers give no cross-queue
   ordering guarantee.
8. **One AMQP channel per handler.** Channels are not thread-safe.
9. Deserialization errors go straight to the DLQ — retrying does not fix broken JSON.
10. Money in cents (`int64`), never `float`.
11. Graceful shutdown: on `SIGTERM`, finish the in-flight message before exiting.
12. Reconnect to the broker with growing backoff; a dropped connection must not
    kill the process.

---

## 5. Code

**3 Go files + 3 infra files.** One binary; Compose varies the `command:`.

```
message-queue-ecommerce/
├── main.go                  # switch: "api" | "worker"
├── mq.go                    # connect, declare topology, publish, consume, retry/DLQ
├── handlers.go              # pagamento, estoque, notificacao
├── Dockerfile
├── deploy/
│   ├── docker-compose.yml
│   ├── init.sql
│   └── definitions.json     # RabbitMQ users and permissions
└── docs/
```

```yaml
# deploy/docker-compose.yml
services:
  rabbitmq: { image: rabbitmq:3.13-management, ports: ["5672:5672","15672:15672"] }
  postgres: { image: postgres:16, volumes: ["./init.sql:/docker-entrypoint-initdb.d/init.sql"] }
  api:      { build: .., command: ["/app","api"], ports: ["8080:8080"] }
  worker:   { build: .., command: ["/app","worker"] }
```

```sql
-- deploy/init.sql
CREATE TABLE pedidos (
  id UUID PRIMARY KEY,
  status TEXT NOT NULL,
  total_centavos BIGINT NOT NULL,
  atualizado_em TIMESTAMPTZ DEFAULT now()
);
CREATE TABLE produtos (
  id UUID PRIMARY KEY, nome TEXT,
  disponivel INT NOT NULL CHECK (disponivel >= 0)   -- last line of defence
);
CREATE TABLE processados (id UUID PRIMARY KEY);     -- idempotency
INSERT INTO produtos VALUES (gen_random_uuid(), 'Teclado', 10);
```

Stack: Go 1.23 · `rabbitmq/amqp091-go` · `net/http` · `database/sql` ·
Postgres 16 · Docker Compose. Estimate: **~400 lines of Go**.

---

## 6. User Stories

| ID | As a | I want | So that |
|---|---|---|---|
| US1 | Customer | an immediate order confirmation | I don't wait on the payment gateway |
| US2 | Dev | to publish with confirms to durable queues | no message is lost if the broker restarts |
| US3 | Dev | pagamento and estoque to run in sequence via routing keys | topic routing is demonstrated |
| US4 | Dev | two handlers to receive the same event | fan-out is demonstrated |
| US5 | Dev | failures to enter retry and then land in the DLQ | fault tolerance is demonstrated |
| US6 | Dev | to scale consumers with `--scale` | scalability is demonstrated |
| US7 | Dev | redelivery not to decrement stock twice | idempotency is demonstrated |
| US8 | Dev | to see queues, rates and the DLQ in the Management UI | we get a dashboard without writing a frontend |

**Acceptance** — each one is a check performed live during the presentation:

- US1: `POST /pedidos` returns 201 in < 300 ms with `worker` stopped.
- US2: `docker restart rabbitmq` with a full queue → messages are still there.
- US3: `estoque` only receives a message after `pagamento.aprovado`.
- US4: one `pagamento.aprovado` shows up in both `estoque` and `notificacao` logs.
- US5: `FAIL_RATE=1` → 3 attempts 10 s apart → message in `dlq`.
- US6: `--scale worker=3` → Management UI shows 3 consumers per queue.
- US7: republish the same `id` → `disponivel` drops only once.
- US8: `localhost:15672` shows the 5 queues, and `dlq` is inspectable.

---

## 7. Plan

### Stages 1–2 · 2026-09-17 · **done**

`etapa1.md`, `etapa2.md` and the 5 diagrams are written. No code is graded here.

### Stages 3–5 · 2026-09-24

| # | Task | Output | US |
|---|---|---|---|
| 3.1 | `mq.go`: connect, declare 3 exchanges + 5 queues, publish with confirms | code | US2 |
| 3.2 | `mq.go`: consume with prefetch, manual ack, `x-death` retry/DLQ routing | code | US5 |
| 3.3 | `definitions.json`: vhost, users, permissions | config | — |
| 3.4 | `etapa3.md`: every parameter with its rationale + TLS config | doc | — |
| 4.1 | `handlers.go`: the 3 handlers, with rules 4, 6 and 7 | code | US3, US4, US7 |
| 4.2 | `main.go`: mode switch + HTTP api | code | US1 |
| 4.3 | `deploy/`: Compose, `init.sql`, Dockerfile | config | US6 |
| 4.4 | Run the 9 use cases, capture evidence | `etapa4.md` | all |
| 5.1 | `etapa5.md`: stack, message format, practices (§4 here, in Portuguese) | doc | — |
| 5.2 | README: step-by-step run instructions | `README.md` | — |

**Order:** 3.1 → 3.2 → 4.1 → 4.2 → 4.3, then the docs. Everything depends on `mq.go`.

### Presentation · 2026-09-24 · 10 min

| Time | Content |
|---|---|
| 0–2 | Scenario and why messaging (`etapa1.md`) |
| 2–4 | Architecture: `arquitetura.png`, topic exchange, the 5 queues |
| 4–6 | Demo: an order flowing through, Management UI moving (cases 1, 4) |
| 6–8 | Demo: `FAIL_RATE=1` → retry → DLQ on screen; `stop worker` → pile up → drain (cases 5, 6, 7) |
| 8–10 | Demo: `--scale worker=3` (case 8), close with the decisions table |

### Split by team member

| Track | Owns | Presents |
|---|---|---|
| A | `mq.go` — topology, publisher | Architecture and RabbitMQ configuration |
| B | `handlers.go` + `main.go` | Happy-path and fan-out demo |
| C | Retry/DLQ logic + `definitions.json` | Failure demo and security |
| D | `deploy/`, docs, diagrams | Scenario and scaling demo |

With fewer members, merge adjacent tracks. **Anyone who does not present gets a
zero; if any member leaves before all presentations finish, the team's grade is
divided by 7.** GitHub link goes to the AVA.

---

## 8. Cut, and When to Add It

| Cut | Add when |
|---|---|
| Separate service per domain | The team needs independent deploys |
| `ROLES` env var for selective scaling | One queue needs to scale apart from the others |
| Outbox pattern | Losing a message between commit and publish becomes a real problem |
| 3-level retry (5s/30s/2m) | A single 10 s level no longer covers real recovery time |
| TLS enabled | It leaves the local machine |
| Quorum queues | There is more than one broker node |
| Migration tool | There is a second migration |
| Dedicated audit service | The audit trail needs its own retention or queries |
