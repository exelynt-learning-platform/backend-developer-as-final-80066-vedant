# Resource Booking System

A RESTful **Resource Booking System** built with **Spring Boot 3 / Java 17**, secured with **Spring Security + JWT**,
and backed by **PostgreSQL** (or **MySQL**). Users can browse resources and manage their own reservations;
administrators have full CRUD access over resources and all reservations.

---

## Tech Stack

| Layer          | Technology                                   |
|----------------|-----------------------------------------------|
| Language       | Java 17                                       |
| Framework      | Spring Boot 3.2 (Web, Data JPA, Security, Validation) |
| Auth           | JWT (jjwt), BCrypt password hashing           |
| Database       | PostgreSQL or MySQL (JPA/Hibernate)           |
| Docs           | springdoc-openapi (Swagger UI) + Postman collection |
| Build          | Maven                                         |
| Tests          | JUnit 5, Spring Boot Test, MockMvc, H2 (in-memory) |

---

## Project Structure

```
src/main/java/com/exelynt/booking/
├── BookingApplication.java
├── config/          # OpenApiConfig, DataSeeder
├── security/        # JwtUtil, JwtAuthFilter, SecurityConfig, UserDetailsServiceImpl, ...
├── entity/           # User, Resource, Reservation, Role, ReservationStatus
├── repository/       # Spring Data JPA repositories (+ Specification support)
├── dto/               # Request/response DTOs
├── service/          # Business logic (Auth, Resource, Reservation)
├── controller/        # REST controllers
└── exception/         # Custom exceptions + GlobalExceptionHandler
```

---

## Prerequisites

- Java 17+
- Maven 3.8+
- PostgreSQL 13+ **or** MySQL 8+ (or use the provided `docker-compose.yml` for Postgres)

---

## Setup

### 1. Clone & configure environment variables

Copy `.env.example` to `.env` (or export the variables in your shell) and adjust as needed:

```bash
cp .env.example .env
```

Key variables:

| Variable                | Description                                   | Default |
|--------------------------|------------------------------------------------|---------|
| `DB_URL`                | JDBC URL                                        | `jdbc:postgresql://localhost:5432/booking_db` |
| `DB_USERNAME`           | DB username                                     | `postgres` |
| `DB_PASSWORD`           | DB password                                     | `postgres` |
| `DB_DRIVER`             | JDBC driver class                               | `org.postgresql.Driver` |
| `DB_DIALECT`            | Hibernate dialect                               | `org.hibernate.dialect.PostgreSQLDialect` |
| `DDL_AUTO`              | Hibernate schema strategy                       | `update` |
| `JWT_SECRET`            | HMAC signing secret (256-bit min)               | (dev default — **change in production**) |
| `JWT_EXPIRATION_MS`     | Token lifetime in ms                            | `86400000` (24h) |
| `SEED_DATA`             | Seed ADMIN/USER accounts + sample resources     | `true` |
| `SEED_ADMIN_USERNAME` / `SEED_ADMIN_PASSWORD` | Seed admin credentials       | `admin` / `Admin@123` |
| `SEED_USER_USERNAME` / `SEED_USER_PASSWORD`   | Seed user credentials        | `user` / `User@123` |

### 2. Start a database

**Option A — Docker (PostgreSQL):**

```bash
docker compose up -d
```

**Option B — MySQL:** create a database and switch the `DB_*` variables to the MySQL block
commented in `.env.example` (driver `com.mysql.cj.jdbc.Driver`, dialect `org.hibernate.dialect.MySQLDialect`).

### 3. Run the application

```bash
# export the variables from .env, then:
mvn spring-boot:run
```

On startup, `DataSeeder` creates the seed ADMIN and USER accounts (if they don't already exist)
plus a handful of sample resources, and Hibernate creates/updates the schema (`ddl-auto=update`).

The API is available at `http://localhost:8080`.

### 4. Run tests

Tests run against an in-memory H2 database (`src/test/resources/application-test.yml`) and don't
require an external database:

```bash
mvn test
```

---

## API Documentation

- **Swagger UI:** `http://localhost:8080/swagger-ui.html`
- **OpenAPI JSON:** `http://localhost:8080/v3/api-docs`
- **Postman collection:** `postman_collection.json` (import into Postman; it auto-captures
  `adminToken` / `userToken` / `resourceId` / `reservationId` via test scripts on each request)

To call protected endpoints in Swagger UI, click **Authorize**, log in via `/auth/login` to get a
token, then paste it as `Bearer <token>`.

---

## Seed Users

| Username | Password    | Role  |
|----------|-------------|-------|
| `admin`  | `Admin@123` | ADMIN |
| `user`   | `User@123`  | USER  |

New users can also self-register via `POST /auth/register` (always created with role `USER` —
ADMIN accounts are never created through the public API, only via seeding).

---

## Authentication

```
POST /auth/login
Content-Type: application/json

{ "username": "admin", "password": "Admin@123" }
```

Response:

```json
{
  "token": "eyJhbGciOi...",
  "tokenType": "Bearer",
  "username": "admin",
  "role": "ADMIN",
  "expiresInMs": 86400000
}
```

Use the token on subsequent requests:

```
Authorization: Bearer eyJhbGciOi...
```

Passwords are hashed with **BCrypt**; the API is fully **stateless** (no server-side sessions —
`SessionCreationPolicy.STATELESS`), with the JWT validated on every request by a custom
`OncePerRequestFilter`.

---

## Authorization Model (RBAC)

| Endpoint                              | ADMIN | USER                          |
|-----------------------------------------|:-----:|:------------------------------:|
| `GET /api/resources`, `GET /api/resources/{id}` | ✅ | ✅ |
| `POST/PUT/DELETE /api/resources/**`     | ✅ | ❌ (403) |
| `POST /api/reservations`                | ✅ | ✅ (creates for *self* only) |
| `GET /api/reservations`                 | ✅ all | ✅ own only |
| `GET /api/reservations/{id}`            | ✅ any | ✅ own only (403 otherwise) |
| `PATCH /api/reservations/{id}/cancel`   | ✅ any | ✅ own only |
| `PUT /api/reservations/{id}` (full update, incl. status) | ✅ | ❌ (403) |
| `DELETE /api/reservations/{id}`         | ✅ | ❌ (403) |

Enforcement happens at two levels:
1. **Coarse-grained**, via `SecurityConfig` (`hasRole(...)`) and `@PreAuthorize` on controller methods.
2. **Fine-grained ownership**, in `ReservationService`, which always derives the acting user's
   identity from the authenticated JWT principal (`SecurityContextHolder`) — **never** from the
   request body — and throws `AccessDeniedException` (→ HTTP 403) if a USER tries to access another
   user's reservation.

---

## Endpoints Overview

### Auth
- `POST /auth/login` — authenticate, receive JWT
- `POST /auth/register` — self-register a USER account

### Resources
- `GET /api/resources?page=&size=&sortBy=&sortDir=` — paginated list (ADMIN, USER)
- `GET /api/resources/{id}` — get one (ADMIN, USER)
- `POST /api/resources` — create (ADMIN)
- `PUT /api/resources/{id}` — update (ADMIN)
- `DELETE /api/resources/{id}` — delete (ADMIN)

### Reservations
- `POST /api/reservations` — create for the current user
- `GET /api/reservations?status=&minPrice=&maxPrice=&page=&size=&sortBy=&sortDir=` — list
  (filtered/paginated/sorted; USER scoped to own, ADMIN sees all)
- `GET /api/reservations/{id}` — get one (ownership enforced for USER)
- `PUT /api/reservations/{id}` — full update incl. status (ADMIN)
- `PATCH /api/reservations/{id}/cancel` — cancel (own for USER, any for ADMIN)
- `DELETE /api/reservations/{id}` — delete (ADMIN)

**Filtering & pagination example:**

```
GET /api/reservations?status=CONFIRMED&minPrice=20&maxPrice=200&page=0&size=10&sortBy=price&sortDir=desc
```

`status` accepts `PENDING`, `CONFIRMED`, `CANCELLED`. Filters are combined with AND and applied via
a dynamic JPA `Specification`. Pagination defaults: `page=0`, `size=10`. Sorting defaults: `sortBy=id`,
`sortDir=asc`.

---

## Validation & Error Handling

Requests are validated with Bean Validation (`@NotBlank`, `@NotNull`, `@Future`, `@DecimalMin`, `@Digits`, etc.),
plus cross-field checks in the service layer (e.g. `endTime` must be after `startTime`, price must be
non-negative, `minPrice` must not exceed `maxPrice`). All errors are returned as a consistent JSON body
via a `@RestControllerAdvice` global exception handler:

```json
{
  "timestamp": "2026-09-07T10:15:30",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "path": "/api/reservations",
  "details": ["price: Price must not be negative"]
}
```

| Scenario                          | HTTP Status |
|-------------------------------------|:-----------:|
| Missing/invalid request body fields | 400 |
| No/invalid/expired JWT              | 401 |
| Valid JWT, insufficient role/ownership | 403 |
| Entity not found                    | 404 |
| Unique constraint violation (e.g. duplicate username) | 409 |
| Unhandled server error              | 500 |

---

## Database Schema (high level)

- **users**(id, username [unique], password [BCrypt hash], role [ADMIN/USER], enabled, created_at)
- **resources**(id, name, type, description, location, capacity, available, created_at, updated_at)
- **reservations**(id, resource_id → resources.id, user_id → users.id, start_time, end_time,
  price [decimal(10,2)], status [PENDING/CONFIRMED/CANCELLED], created_at, updated_at)

Relationships: `Reservation` → `Resource` (many-to-one), `Reservation` → `User` (many-to-one).
`ddl-auto=update` lets Hibernate manage the schema for local/dev use; use a migration tool
(Flyway/Liquibase) for production.

---

## Testing

Located under `src/test/java/com/exelynt/booking`:

- **`AuthControllerTest`** — login success/failure, validation, unauthenticated access rejection.
- **`ResourceControllerTest`** — RBAC on resource CRUD (ADMIN vs USER), validation, 404 handling.
- **`ReservationControllerTest`** — reservation creation always bound to the JWT identity, ownership
  enforcement (USER can't view/delete another user's reservation, ADMIN can), status transitions,
  price/date validation, filtering, and pagination.
- **`JwtUtilTest`** — unit tests for token generation, claim extraction, and expiration handling.

Run everything with:

```bash
mvn test
```

---

## Notes on Design Decisions

- **Identity from JWT, not request body:** `ReservationRequest` intentionally has no `userId`/`username`
  field. `ReservationService.create(...)` always resolves the owner from the authenticated principal.
- **Stateless JWT auth:** no HTTP sessions; every request is authenticated independently via the
  `Authorization: Bearer <token>` header, validated in `JwtAuthFilter`.
- **Defense in depth for RBAC:** URL-level rules in `SecurityConfig` + method-level `@PreAuthorize` +
  service-level ownership checks, so a misconfigured route still can't leak another user's data.
- **Dynamic filtering:** `ReservationSpecification` builds a JPA `Specification` combining `status`,
  `minPrice`, `maxPrice`, and (for USER) a mandatory `userId` predicate — filters can never be used
  to bypass ownership scoping.
