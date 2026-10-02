# Integrity map

| Invariant | Affected tables / columns | Protection / classification | Limitation | Expected failure | Evidence |
| --- | --- | --- | --- | --- | --- |
| 1. Trip capacity cannot be negative | `trips.capacity` | CHECK `trips_capacity_non_negative` — Direct constraint | None at row level | CHECK violation, SQLSTATE 23514 | Negative test #1 |
| 2. Reserved seats must be between 0 and capacity | `trips.reserved_seats`, `trips.capacity` | CHECK `trips_reserved_seats_valid` — Direct constraint | Does not prevent concurrent purchases from competing for the last seat | CHECK violation, SQLSTATE 23514 | Negative test #2 |
| 3. A ticket must reference an existing trip | `tickets.trip_id` → `trips.id` | FK `tickets_trip_fk` — Direct constraint | Does not check whether the trip has available capacity | FK violation, SQLSTATE 23503 | Negative test #3 |
| 4. Ticket validity cannot end before it starts | `tickets.valid_from_utc`, `tickets.valid_to_utc` | CHECK `tickets_validity_window_valid` — Direct constraint | Does not automatically expire a ticket when time passes | CHECK violation, SQLSTATE 23514 | Negative test #4 |
| 5. Ticket codes must be unique | `tickets.ticket_code` | UNIQUE `tickets_ticket_code_unique` — Unique constraint | None for unambiguous code lookup | UNIQUE violation, SQLSTATE 23505 | Negative test #5 |
| 6. Ticket status must be a known value | `tickets.status` | CHECK `tickets_status_known` — Direct constraint | Does not enforce which status transitions are allowed | CHECK violation, SQLSTATE 23514 | Negative test #6 |
| 7. Prices and payment amounts cannot be negative | `products.price`, `tickets.price`, `payments.amount` | CHECK constraints — Direct constraint | Does not guarantee product price, ticket price and payment amount match | CHECK violation, SQLSTATE 23514 | Negative test #7 + migration DDL |
| 8. A payment must reference an existing ticket | `payments.ticket_id` → `tickets.id` | FK `payments_ticket_fk` — Direct constraint | Payment workflow still requires transaction/application logic | FK violation, SQLSTATE 23503 | Negative test #8 |
| 9. External payment references must be unique | `payments.external_payment_reference` | UNIQUE `payments_external_reference_unique` — Unique constraint | Cannot control what happens at the external payment provider | UNIQUE violation, SQLSTATE 23505 | Negative test #9 |
| 10. Validation ticket ID and code must belong to the same ticket | `validations.ticket_id`, `validations.ticket_code` → `tickets(id, ticket_code)` | Composite FK `validations_ticket_id_code_fk` — Direct constraint | `ticket_code` is duplicated in `validations` | FK violation, SQLSTATE 23503 | Negative test #10 |
| 11. Currency must be present and consistently represented | `products.currency`, `tickets.currency`, `payments.currency` | NOT NULL + three-character CHECK — Direct constraint | Three characters does not prove it is a real ISO currency, nor that related rows use the same currency | NOT NULL / CHECK violation | Migration DDL |
| 12. Sold/reserved seats must never exceed real availability | `trips.reserved_seats`, `tickets` | Cross-row / transaction invariant | Cannot be fully protected by a simple CHECK; concurrent purchases require transaction handling | Current constraints may not detect race conditions | To be handled in transactions lecture |
| 13. Disabled users should not be able to buy tickets | `users.is_disabled`, `tickets.user_id` | Unresolved domain/workflow decision | Requires checking another row and deciding the exact business rule | Not currently rejected | Open domain decision |



## Issue register

### Issue 1 — Concurrent purchases can oversell a trip

- Evidence: `trips_reserved_seats_valid` checks that `reserved_seats` is between 0 and `capacity`, but it only checks the values in one row when the write happens.
- Problem: Two customers could try to buy the last available seat at the same time. Both could read the same `reserved_seats` value before either purchase is completed.
- Consequence: A simple CHECK constraint is not enough to guarantee that the last seat is only sold once.
- Specific improvement: Handle the seat reservation inside a database transaction. For example, lock the trip row while updating `reserved_seats`, or use an atomic update that only succeeds when `reserved_seats < capacity`.
- Open question: Which transaction/concurrency strategy should be used for ticket purchase? 

### Issue 2 — Payment amount may not match the ticket price

- Evidence: `payments_amount_non_negative` prevents negative payment amounts, and `payments_ticket_fk` guarantees that the referenced ticket exists. However, there is no constraint requiring `payments.amount` to equal `tickets.price`.
- Problem: A payment can therefore reference a valid ticket but contain a different amount.
- Consequence: The database could contain inconsistent financial data, for example a ticket priced at 36 DKK with a payment of 30 DKK.
- Specific improvement: The purchase workflow should verify the ticket price and payment amount together and perform the related database changes within the same transaction.
- Open question: Should payment amount always equal ticket price, or should the model also support discounts, fees and refunds?

## State-transition trace

### Ticket purchase

1. Read the selected `trip` and `product`. Check that the trip is scheduled and has available capacity, and obtain the product price and currency.
2. Create the ticket with references to the existing user, trip and product. The foreign keys, ticket-code uniqueness, validity window, price, currency and status constraints protect the inserted ticket.
3. Create the payment referencing the ticket. The payment must have a valid amount/status and a unique external payment reference.
4. If the purchase succeeds, increment `trips.reserved_seats`. The `trips_reserved_seats_valid` constraint prevents a single update from setting the value above capacity.
5. The complete purchase should eventually be handled as one transaction so that a failure does not leave only part of the purchase stored.

### Ticket validation

1. Look up the ticket using `ticket_code`. Because the ticket code is unique, it identifies at most one ticket.
2. Check whether the ticket can currently be used, including its status and validity period (`valid_from_utc` to `valid_to_utc`).
3. Insert a row into `validations`. The composite foreign key ensures that `ticket_id` and `ticket_code` belong to the same ticket, while the result must be a recognised value.
4. For a successfully used single-use ticket, update the ticket status to `Validated`.
5. The current constraints do not guarantee that the validation insert and ticket-status update happen together; that requires transaction handling.

## Delete and update behaviour

Tickets, payments and validations represent historical events and should normally not be deleted once they have been used or recorded. The foreign keys therefore use PostgreSQL's default restrictive behaviour rather than cascading deletes.
For example, a ticket referenced by a payment or validation should not be deleted, because this would remove information needed to understand the historical payment or validation. Similarly, trips and products referenced by sold tickets should normally be cancelled or deactivated rather than deleted.
Primary and foreign-key identifiers should also not normally be changed after they have been referenced, because changing them could alter the meaning of historical records.
A future design could use status fields or soft deletion for data that should no longer be active while still preserving historical information.

## Successful and rejected writes

The migration was applied successfully to the existing seed data.
A valid capacity update was accepted:

sql
UPDATE trips
SET capacity = 121
WHERE id = 'TRIP-M2-20260429-0800';

where `reserved_seats` was 120. The update succeeded because the new capacity is greater than or equal to `reserved_seats`.

The supplied negative tests were also executed. All ten invalid writes were rejected by the intended database constraints.