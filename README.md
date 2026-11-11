# ChartSafe Engineering Blueprint (MVP)

## Project Goal
Build a secure, usable, multi-team robotics inventory system that proves a new user can independently find, check out, and return parts without assistance.

---

## 1) Recommended Technology Stack (with tradeoffs)

### Backend
- **Node.js + TypeScript + Express**
  - **Why:** Fast iteration, large ecosystem, easy API structure for judges to understand.
  - **Tradeoff:** Less out-of-box structure than NestJS; we enforce architecture by folder boundaries.

### Database
- **SQLite (WAL mode) + Prisma ORM (or Drizzle as alternative)**
  - **Why:** Required for local MVP, simple setup, persistent relational model, migrations, parameterized queries.
  - **Tradeoff:** SQLite has limited write concurrency vs PostgreSQL. Acceptable for MVP team testing; schema designed to migrate to PostgreSQL later.

### Frontend
- **React + TypeScript + Vite**
  - **Why:** Clear component architecture, fast dev loop, easy responsive UI.
  - **Tradeoff:** More design responsibility than template systems (which we intentionally avoid).

### Auth/session
- **Server-managed sessions with HttpOnly secure cookies** (not localStorage JWT)
  - **Why:** Better XSS resistance and easier session invalidation.
  - **Tradeoff:** Requires CSRF defenses and session store management.

### UI
- **Custom CSS design system (tokens + reusable components)**
  - **Why:** Original implementation, polished UX, judge-friendly explanation.
  - **Tradeoff:** More up-front effort than prebuilt templates.

### Testing
- **Vitest + React Testing Library + Supertest + Playwright**
  - **Why:** Unit/integration/E2E coverage for critical flows.
  - **Tradeoff:** More setup, but required to prove reliability.

---

## 2) System Architecture

- **Frontend (React SPA):** Dashboard, inventory, checkout/return, admin screens.
- **Backend API (Express):** Auth, team-scoped business logic, transaction-safe checkout/return.
- **Database (SQLite):** Teams, users, categories, items, checkout ledger.

### Core boundary rules
- Frontend never decides authorization.
- Backend derives `team_id` from authenticated session, never from user-provided payload.
- Database enforces relational integrity and uniqueness constraints.

---

## 3) Database Schema and Relationships

## teams
- `id` (PK)
- `name` (unique)
- `join_code_hash` (unique)
- `created_at`, `updated_at`

## users
- `id` (PK)
- `team_id` (FK -> teams.id, indexed)
- `nickname` (team-unique)
- `role` (`admin` | `member`)
- `password_hash`
- `is_active`
- `created_at`, `updated_at`
- **Constraint:** `UNIQUE(team_id, nickname)`

## categories
- `id` (PK)
- `team_id` (FK -> teams.id, indexed)
- `name`
- `description`
- `created_at`, `updated_at`
- **Constraint:** `UNIQUE(team_id, name)`

## inventory_items
- `id` (PK)
- `team_id` (FK -> teams.id, indexed)
- `category_id` (FK -> categories.id)
- `name`
- `description`
- `item_type` (`consumable` | `reusable`)
- `total_quantity` (CHECK >= 0)
- `available_quantity` (CHECK >= 0)
- `low_stock_threshold` (CHECK >= 0)
- `storage_location`
- `is_archived`
- `created_at`, `updated_at`
- **Constraints:**
  - `UNIQUE(team_id, name, storage_location)`
  - `available_quantity <= total_quantity`

## checkouts
- `id` (PK)
- `team_id` (FK -> teams.id, indexed)
- `item_id` (FK -> inventory_items.id, indexed)
- `user_id` (FK -> users.id, indexed)
- `quantity` (CHECK > 0)
- `checked_out_at`
- `due_at`
- `returned_at` (nullable)
- `returned_quantity` (default 0, CHECK >= 0)
- `status` (`open` | `partial` | `returned` | `overdue`)
- **Constraints:**
  - `returned_quantity <= quantity`

## optional activity_logs (future)
- auditable actions (who, what, entity, before/after summaries, timestamp).

### Why this works
- Team is explicit on all tenant-owned tables to simplify and harden team filtering.
- Checkout ledger preserves history and supports “who has item now” by querying open/partial records.

---

## 4) API Endpoint Architecture

All endpoints under `/api`.

### Auth
- `POST /auth/register-admin` (bootstrap first admin + team creation)
- `POST /auth/join-team` (join by code)
- `POST /auth/login`
- `POST /auth/logout`
- `GET /auth/me`

### Team
- `GET /team/settings` (admin)
- `PATCH /team/settings` (admin)
- `POST /team/join-code/rotate` (admin)

### Categories
- `GET /categories`
- `POST /categories` (admin)
- `PATCH /categories/:id` (admin)
- `DELETE /categories/:id` (admin)

### Inventory
- `GET /items?search=&category=&lowStockOnly=`
- `POST /items` (admin)
- `GET /items/:id`
- `PATCH /items/:id` (admin)
- `DELETE /items/:id` (admin/archive)

### Checkout/Return
- `POST /checkouts` (member/admin)
- `POST /checkouts/:id/return` (member/admin)
- `GET /checkouts/open`
- `GET /checkouts/history`

### Admin Users
- `GET /users` (admin)
- `PATCH /users/:id/role` (admin)
- `PATCH /users/:id/active` (admin)

---

## 5) Authentication and Authorization Model

- Passwords hashed with **Argon2id** (or bcrypt with strong cost if Argon2 unavailable).
- Session cookie flags: `HttpOnly`, `Secure`, `SameSite=Lax`, short idle timeout + absolute timeout.
- Middleware chain:
  1. `requireAuth`
  2. `attachSessionUser`
  3. `requireRole('admin')` where needed.
- Rate-limit login and join endpoints.
- Generic auth errors to prevent account enumeration.

---

## 6) Multi-Tenant Security Model

- Session contains `user_id` and `team_id`.
- Every DB query includes server-side `team_id` predicate from session context.
- Client-supplied team identifiers are ignored/rejected.
- Cross-team resource IDs return `404` (not `403`) to avoid information leakage.
- Input validation via schema validators (Zod/Joi).
- Strict parameterized queries via ORM/query builder.

---

## 7) Frontend Page and Component Architecture

### Pages
- `/login`
- `/join-team`
- `/dashboard`
- `/inventory`
- `/inventory/:id`
- `/checkouts`
- `/admin/items`
- `/admin/categories`
- `/admin/users`
- `/admin/team-settings`

### Key components
- `AppShell`, `TopNav`, `TeamBadge`, `RoleGate`
- `SearchBar`, `CategoryFilter`, `LowStockBanner`
- `InventoryTable` (large-screen) + `InventoryCards` (mobile)
- `CheckoutModal`, `ReturnModal`
- `StatusPill` (available, low-stock, overdue)

### UX principles
- Big touch targets for younger users.
- Fast scanability for high-volume FRC inventory.
- Always show: location, available qty, who has it, due date.

---

## 8) Checkout/Return Transaction Design

### Checkout (single transaction)
1. Validate quantity > 0.
2. Select item row constrained by `team_id`.
3. Ensure `available_quantity >= requested_quantity`.
4. Decrement `available_quantity` atomically.
5. Insert checkout record (`status='open'`, due date set).
6. Commit.

### Return (single transaction)
1. Load checkout by `id + team_id` and verify open/partial.
2. Validate return quantity.
3. Increment item `available_quantity`.
4. Update checkout `returned_quantity`, `returned_at`, `status`.
5. Commit.

### Concurrency controls
- SQLite transaction modes (`BEGIN IMMEDIATE`) for write locking.
- Retry policy for transient busy lock errors.
- DB constraints prevent negative quantities even if logic bugs occur.

---

## 9) Testing Strategy

### Unit tests
- Quantity math, overdue calculation, role guards, validation schemas.

### Integration tests (API + DB)
- Team isolation checks across all endpoints.
- Checkout and return happy paths + edge cases (insufficient stock, cross-team ID, partial returns).
- Auth/session lifecycle.

### E2E tests (Playwright)
- New user flow: find item → view location → checkout → due date visible → return.
- Admin flow: create category/item, low-stock indicator visibility.

### Scale testing prep
- Deterministic seed scripts for hundreds of items and many checkouts.

---

## 10) Deployment Strategy

### MVP deployment
- Single container/app service running Node API + static frontend.
- SQLite file on persistent volume.
- Environment variables for secrets and cookie config.

### Production-ready evolution
- Move to PostgreSQL when concurrency/volume requires it.
- Keep ORM models and migrations portable to reduce migration risk.

---

## 11) Development Milestones (dependency order)

1. **Project scaffolding and quality gates** (TS config, linting, test runners, env validation).
2. **Database schema + migrations + seed framework**.
3. **Auth/session foundation + role middleware**.
4. **Team isolation enforcement at repository/service layer**.
5. **Inventory and category CRUD (admin)**.
6. **Checkout/return transactional service + API**.
7. **Member inventory/search/filter UX**.
8. **Dashboard alerts (low stock, overdue, checked-out visibility)**.
9. **E2E usability test scripts + seed datasets for realistic load**.
10. **Hardening: rate limits, audit logs, backup/restore docs, deployment scripts**.

---

## 12) Major Risks and Mitigations

- **Risk:** Tenant data leakage due to missing team filter.
  - **Mitigation:** central query helpers that always require session team context; integration tests for cross-team access attempts.

- **Risk:** Inventory inconsistency under concurrent checkout.
  - **Mitigation:** transaction boundaries + constraints + lock retry strategy + focused concurrency tests.

- **Risk:** Weak auth/session handling.
  - **Mitigation:** secure cookies, strong password hashing, CSRF protection, login rate limiting.

- **Risk:** MVP UI too complex for younger users.
  - **Mitigation:** usability sessions with FTC/FLL/FRC users and iterative simplification.

- **Risk:** SQLite write limits under heavy parallel use.
  - **Mitigation:** WAL mode now; PostgreSQL migration plan prepared early.

---

## First Concrete Implementation Milestone

**Milestone 1: Scaffold backend/frontend project with TypeScript, establish database schema/migrations for teams/users/categories/items/checkouts, and ship a tested auth/session foundation with enforced team context middleware.**

This is the minimum architectural base needed before any inventory workflow can be safely built.
