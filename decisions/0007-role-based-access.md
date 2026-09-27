# 7. Role-based access: customer, staff, admin

**Status:** Accepted

## Context

After ADR 0006 every signed-in user can reach the same routes, scoped to their own data. A shop
also needs people who work there: staff who handle orders and admins who manage the catalogue.
Customer login is moving to Microsoft Entra External ID, which puts app roles in the token's
`roles` claim - the claim the gateway already maps to authorities for both token issuers.

## Decision

Three roles, each including the ones below it (a role hierarchy: `admin > staff > customer`):

| | customer | staff | admin |
|---|---|---|---|
| Browse products, place orders, see **own** orders | yes | yes | yes |
| See **all** orders, change order status | | yes | yes |
| Create / edit / delete products, change prices | | | yes |

- **No role = customer.** Anyone may sign up and buy. Sign-up never grants anything higher;
  staff and admin are only ever assigned (Entra app-role assignment, or the seeded demo accounts).
- **Enforced at the gateway only**, in the same default-deny rule list as ADR 0006: role checks by
  path + HTTP method. Staff-only rules come before the general "signed-in may use orders" rule.
  Services keep trusting the gateway (private network, no public ports).
- **New endpoints:** `GET /orders/all` and `PATCH /orders/{id}/status` (staff), `POST /products`,
  `PUT /products/{id}`, `DELETE /products/{id}` (admin).
- **Order statuses** gain `SHIPPED`, `DELIVERED`, `CANCELLED`. Allowed moves only:
  `CONFIRMED -> SHIPPED -> DELIVERED` and `CONFIRMED -> CANCELLED`.
- **Audit trail:** every staff/admin change is logged with who (`X-User-Id`), what, before/after.
- **Demo staff and admin accounts** are created at startup by auth-service when their passwords
  are configured. The passwords are generated (Terraform `random_password` -> SSM on AWS; random
  per run in CI) and never committed, so the E2E suite can prove each role's limits.

## Consequences

- One place to read and review who can do what - the gateway's rule list plus the role hierarchy.
- A service reachable without the gateway would trust anyone; the network rules from ADR 0006
  stay load-bearing. Adding checks in the services too (defense in depth) is the upgrade path.
- Deleting a product leaves its stock row and past order lines (they copy the price) untouched.
  New products start with no stock until inventory management exists.
- Local (auth-service) and Entra tokens carry roles the same way, so switching customers to Entra
  (E5) changes configuration, not these rules.
