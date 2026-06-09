# CLAUDE.md

RestaurantOS is a tablet-first dine-in POS for small/medium F&B businesses. Build from the PRD/TRD in:

- `C:\Users\user\Downloads\RestaurantOS_PRD.docx`
- `C:\Users\user\Downloads\RestaurantOS_TRD.docx`

## Goal

MVP features: floor plan, table sessions, order taking, modifiers, KDS, amendments, voids, billing, split/partial payments, menu management, 86/sold-out items, reports, and AI upsell/reorder suggestions.

Keep the product fast, clear, offline-tolerant, and suitable for busy restaurant staff. This is an operational POS, not a marketing UI.

## Stack

- Frontend: Kotlin Android, MVVM, Jetpack Compose, StateFlow, Room, Retrofit, OkHttp WebSocket.
- Backend: Go 1.22+, Gin, PostgreSQL 16, Redis 7, WebSocket, Docker Compose.
- Backend pattern: Handler -> Service -> Repository.
- Data: UUID primary keys, money stored as integer cents, parameterized SQL only.

## Architecture

Backend:

```text
backend/
  cmd/server/
  internal/{handler,service,repository,domain,middleware,ws}
  migrations/
```

Android:

```text
app/src/main/java/com/restaurantos/
  ui/{floorplan,order,kds,payment}
  data/{remote,local,repository}
  domain/models
```

Room should be the offline source of truth. Backend repositories are the only layer that directly accesses PostgreSQL.

## Auth

Use PIN-based staff unlock only. No signup, JWT, or refresh-token flow.

- Roles: `CASHIER`, `MANAGER`
- Store PINs with bcrypt
- Issue random session tokens stored in Redis with 8-hour TTL
- Require `X-Staff-Token` for protected APIs
- Manager-only: high-value voids, manual discounts, floor plan edits, reports

## Core Domain

Tables:

- `outlets`, `staff`, `restaurant_tables`
- `categories`, `menu_items`, `modifier_groups`, `modifier_options`
- `orders`, `order_items`, `order_item_modifiers`
- `payments`, `void_log`, `ai_item_pairs`

Statuses:

- Table: `AVAILABLE`, `OCCUPIED`, `BILL_REQUESTED`, `RESERVED`
- Order: `DRAFT`, `SENT`, `PARTIAL_READY`, `READY`, `BILLED`, `CLOSED`, `VOIDED`
- Item: `PENDING`, `IN_PROGRESS`, `READY`, `SERVED`, `VOIDED`

Use optimistic locking on orders with a `version` field.

## API Shape

Base path: `/api/v1`

Main groups:

- `/auth/pin`
- `/tables`
- `/orders`
- `/orders/kds`
- `/orders/:id/bill`
- `/orders/:id/payments`
- `/menu`
- `/ai/suggest`
- `/reports`

Errors:

```json
{ "error": "message", "code": "MACHINE_READABLE_CODE", "trace_id": "id" }
```

Use `409` for order version conflicts and `422` for business-rule failures.

## WebSocket

Endpoint:

```text
/ws?outlet_id={id}&token={token}
```

Events: `order.sent`, `order.amended`, `order.item_bumped`, `order.item_voided`, `table.status_changed`, `menu.item_86d`, `ai.suggestion`.

Client heartbeat every 30 seconds; reconnect after 3 missed pongs.

## AI Feature

Use an explainable co-occurrence model, not external ML.

- On order close, store item-pair frequencies in `ai_item_pairs`
- Suggest up to 2 items not already in the order
- Cold start returns empty suggestions
- Reorder nudge: occupied > 45 min and no drink reorder in last 30 min
- Suppress repeated nudges with Redis TTL

## Key Rules

- Cannot send empty order to kitchen.
- Sent order amendments must show `MOD` on KDS.
- Voids require reason; above RM30 requires manager PIN.
- Modifier limits must be enforced.
- Partial payments keep order open until fully paid.
- Closed order frees the table.
- 86 changes broadcast to POS and KDS.
- AI suggestions must be dismissible and never block order entry.

## Build Order

1. Backend scaffold, Docker Compose, migrations, health check.
2. Auth, staff seed data, role middleware.
3. Tables and menu APIs.
4. Order lifecycle and WebSocket broadcasts.
5. KDS feed and bump endpoints.
6. Billing, payments, void log, receipt payload.
7. AI suggestions.
8. Reports.
9. Android screens and API integration.
10. Room offline cache and sync.

Run relevant tests/builds before finishing each implementation task.
