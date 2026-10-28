# 📝 Task Management API

Secure Java REST API with **Authentication and Role-Based Access** — built for the Java Development assignment of **Ikaansh Edutech Private Limited**.

JWT login/signup, `USER` / `ADMIN` roles, Task CRUD with owner association, MySQL + Spring Data JPA, pagination & sorting, async event publishing, Swagger UI and JUnit/Mockito tests.

---

## 📑 Table of Contents

- [Features](#-features)
- [Tech Stack](#️-tech-stack)
- [Project Structure](#-project-structure)
- [Setup (Local)](#-setup-local)
- [Docker Setup](#-docker-setup)
- [API Endpoints](#-api-endpoints)
- [Role-Based Access](#-role-based-access-rbac)
- [Sample Requests & Responses](#-sample-requests--responses)
- [Database Schema & Index Notes](#️-database-schema--index-notes)
- [Concurrency, Double Click & Performance](#-concurrency-double-click--performance)
- [Error Handling](#️-error-handling)
- [Async Event Publishing](#-async-event-publishing)
- [Running Tests](#-running-tests)
- [Configuration Reference](#️-configuration-reference)

---

## ✨ Features

**Core**
- Sign-up and login with **BCrypt password hashing** and **JWT** authentication (stateless Spring Security)
- **RBAC** with two roles: `USER` and `ADMIN`
- **Task CRUD**: `title`, `description`, `status`, `dueDate`, `owner` (many tasks → one user)
- **MySQL** + Spring Data JPA/Hibernate, Flyway migrations, index optimisation notes
- Bean Validation on every request + **global exception handling** with one consistent error JSON

**Bonus**
- Pagination and sorting on list endpoints
- Async event publishing using Spring `ApplicationEvent`
- Unit tests (JUnit 5 + Mockito) for the authentication and service layers, plus a full-stack MockMvc test
- Swagger / OpenAPI UI with Bearer-token support

---

## 🛠️ Tech Stack

| Technology | Purpose |
|---|---|
| Java 17, Spring Boot 3.4 | Application framework |
| Spring Security + JJWT 0.11.5 | Authentication, RBAC, JWT |
| Spring Data JPA (Hibernate) | ORM |
| MySQL 8 | Database |
| Flyway | Schema migrations |
| Jakarta Validation | Input validation |
| springdoc-openapi 2.8.5 | Swagger UI |
| JUnit 5, Mockito, H2 (tests only) | Testing |
| Lombok, Docker | Tooling |

---

## 📂 Project Structure

```
dhananjay/src/main/java/com/ikaansh/dhananjay/
├── DhananjayApplication.java
└── TaskModule/
    ├── config/        SecurityConfig, JwtUtils, JwtAuthenticationFilter, SwaggerConfig,
    │                  PasswordConfig, AsyncConfig, RestAuthenticationEntryPoint,
    │                  RestAccessDeniedHandler, SecurityErrorWriter, AppConfig
    ├── controller/    AuthController, TaskController, AdminTaskController
    ├── dto/           AuthDTO/*, TaskDTO/*
    ├── entity/        User, UserType (USER, ADMIN), Task, TaskStatus
    ├── event/         TaskEvent, TaskEventType, TaskEventListener
    ├── exception/     GlobalExceptionHandler, ApiErrorResponse + specific exceptions
    ├── mapper/        TaskMapper
    ├── repository/    UserRepository, TaskRepository
    └── service/       Interface/*, Implementation/*
```

---

## 🚀 Setup (Local)

### Prerequisites
- JDK 17+
- Maven 3.6+ (or use `./mvnw`)
- MySQL 8+

### 1. Database
The database is created automatically on first start (`createDatabaseIfNotExist=true`). Tables are created by Flyway from `src/main/resources/db/migration`.
To create everything manually instead, run the schema/dump file:

```bash
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS ikaansh_task_db CHARACTER SET utf8mb4;"
mysql -u root -p ikaansh_task_db < dhananjay/database/schema.sql
```

### 2. Configure credentials (environment variables — no code change needed)

```bash
export DB_USERNAME=root
export DB_PASSWORD=your_mysql_password
# optional
export DB_URL="jdbc:mysql://localhost:3306/ikaansh_task_db?createDatabaseIfNotExist=true&useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=UTC&sessionVariables=innodb_lock_wait_timeout=5"
export JWT_SECRET=<at least 32 characters>
```
Defaults (used when nothing is exported): user `root`, password `root`.

### 3. Run

```bash
cd dhananjay
mvn clean install        # runs all tests
mvn spring-boot:run
```

### 4. Verify
- Swagger UI: http://localhost:8080/swagger-ui/index.html
- Click **Authorize** and paste the JWT returned by login/signup.

---

## 🐳 Docker Setup

```bash
cd dhananjay
docker compose up --build      # MySQL + application
docker compose down -v         # stop and remove data
```

| Service | Port |
|---|---|
| Spring Boot app | 8080 |
| MySQL | 3306 |

Override secrets with `DB_PASSWORD` / `JWT_SECRET` environment variables.

---

## 📋 API Endpoints

| Method | Endpoint | Access | Description |
|---|---|---|---|
| POST | `/api/auth/signup` | Public | Register (returns JWT) |
| POST | `/api/auth/login` | Public | Login (returns JWT) |
| POST | `/api/v1/tasks` | USER, ADMIN | Create a task (owner = logged-in user) |
| GET | `/api/v1/tasks` | USER, ADMIN | **My** tasks — paginated, sortable, `status` filter |
| GET | `/api/v1/tasks/{id}` | Owner or ADMIN | Get one task |
| PUT | `/api/v1/tasks/{id}` | Owner or ADMIN | Update a task |
| DELETE | `/api/v1/tasks/{id}` | Owner or ADMIN | Delete a task |
| GET | `/api/v1/admin/tasks` | **ADMIN only** | All tasks of all users — paginated, sortable, `status` filter |

**List query params**

| Param | Default | Rules |
|---|---|---|
| `page` | `0` | ≥ 0 |
| `size` | `10` | 1 – 100 |
| `sortBy` | `createdAt` | `id`, `title`, `status`, `dueDate`, `createdAt`, `updatedAt` |
| `direction` | `desc` | `asc` / `desc` |
| `status` | – | `TODO`, `IN_PROGRESS`, `DONE` |

---

## 🛡️ Role-Based Access (RBAC)

| Action | USER | ADMIN |
|---|---|---|
| Signup / login | ✅ | ✅ |
| Create task | ✅ (own) | ✅ (own) |
| List my tasks | ✅ | ✅ |
| Read / update / delete **own** task | ✅ | ✅ |
| Read / update / delete **another user's** task | ❌ 403 | ✅ |
| `GET /api/v1/admin/tasks` | ❌ 403 | ✅ |

Enforced twice: URL rules in `SecurityConfig` and `@PreAuthorize` on controllers; ownership is checked in the service layer.

> `app.security.allow-admin-signup` (env `ALLOW_ADMIN_SIGNUP`, default `true`) lets the assignment reviewer register an ADMIN easily. Set it to `false` in a real deployment.

---

## 📝 Sample Requests & Responses

### Signup — `POST /api/auth/signup`
```json
{ "name": "Dhananjay", "email": "dhananjay@example.com", "password": "Password@123", "userType": "ADMIN" }
```
`201 Created`
```json
{ "id": 1, "email": "dhananjay@example.com", "name": "Dhananjay", "token": "<jwt>", "message": "Signup successful" }
```

### Login — `POST /api/auth/login`
```json
{ "email": "dhananjay@example.com", "password": "Password@123" }
```

### Create task — `POST /api/v1/tasks`  (`Authorization: Bearer <jwt>`)
```json
{ "title": "Prepare report", "description": "Quarterly numbers", "status": "TODO", "dueDate": "2026-11-05" }
```
`201 Created`
```json
{
  "id": 1, "title": "Prepare report", "description": "Quarterly numbers",
  "status": "TODO", "dueDate": "2026-11-05",
  "ownerId": 1, "ownerName": "Dhananjay", "ownerEmail": "dhananjay@example.com",
  "createdAt": "2026-10-27T10:00:00.123456Z", "updatedAt": "2026-10-27T10:00:00.123456Z"
}
```
`status` is optional (defaults to `TODO` on create, unchanged on update). `dueDate` is required (`yyyy-MM-dd`) and cannot be in the past on create.

### List — `GET /api/v1/tasks?page=0&size=10&sortBy=dueDate&direction=asc&status=TODO`
```json
{
  "content": [ { "id": 1, "title": "Prepare report", "...": "..." } ],
  "page": 0, "size": 10, "totalElements": 1, "totalPages": 1, "first": true, "last": true
}
```

### Delete — `DELETE /api/v1/tasks/1`
```json
{ "message": "Task deleted successfully" }
```

---

## 🗄️ Database Schema & Index Notes

Schema file: `dhananjay/database/schema.sql` (same as the Flyway migrations).

**users** — `id` PK, `email` UNIQUE, `name`, `user_type` (USER/ADMIN), `password` (BCrypt), `created_at`

**tasks** — `id` PK, `title`, `description`, `status`, `due_date`, `owner_id` FK → `users.id` (ON DELETE CASCADE), `created_at`, `updated_at`, `version`

Relationship: **User 1 — N Task** (`@ManyToOne(fetch = LAZY)` on `Task.owner`).

| Index | Columns | Why |
|---|---|---|
| `uk_users_email` (unique) | `email` | Login / JWT filter lookup + duplicate-email guarantee |
| `idx_tasks_owner_status_due` | `owner_id, status, due_date` | "My tasks", "my tasks by status", sort by due date. Its `owner_id` prefix also serves the foreign key |
| `idx_tasks_owner_created` | `owner_id, created_at` | Default newest-first listing and the double-click duplicate check |
| `idx_tasks_status_due` | `status, due_date` | ADMIN list filtered by status |

---

## ⚡ Concurrency, Double Click & Performance

| Concern | Solution |
|---|---|
| **Double click on Create** | The creator's `users` row is locked with `SELECT … FOR UPDATE` (pessimistic lock), then an identical task (same owner + title + due date) created in the last `app.task.duplicate-window-seconds` (5s) is rejected with **409**. Concurrent creates cannot slip through |
| **Double click on Update** | Task row loaded with `PESSIMISTIC_WRITE`; the second request waits, then sees identical values → no write, no event, same response |
| **Double click on Delete** | Row locked first; the second request gets a clean **404** |
| **Double click on Signup** | `existsByEmail` fast path + unique index; a race is caught (`DataIntegrityViolationException`) and returned as **409** |
| **Lock waits** | `innodb_lock_wait_timeout=5` in the JDBC URL — a stuck lock returns **409 "please retry"** instead of hanging |
| **Lost updates** | `@Version` optimistic lock as a last safety net → 409 |
| **N+1 queries** | `Task.owner` is LAZY; every list/detail query uses `@EntityGraph(attributePaths = "owner")` → one SQL (+ one count) per page |
| **Large result sets** | Page size capped at 100; sort fields whitelisted (no arbitrary property sorting) |
| **Login/JWT** | Token parsed once per request; `existsByEmail` instead of loading rows; `open-in-view=false`; read-only transactions for reads |
| **Security** | Same error text for unknown email / wrong password (no user enumeration), password length capped at 72 (BCrypt), emails normalised to lower case |

---

## ⚠️ Error Handling

Every error (controllers **and** security filters) uses the same JSON:

```json
{
  "timestamp": "2026-10-27T10:00:00.123Z",
  "status": 400,
  "error": "Bad Request",
  "message": "title: title is required",
  "path": "/api/v1/tasks",
  "errors": [ { "field": "title", "message": "title is required" } ]
}
```

| Status | When |
|---|---|
| 400 | Validation failure, malformed JSON / bad enum / bad date, invalid `sortBy` / `size`, invalid user type, past due date |
| 401 | Missing / expired / invalid JWT, wrong email or password |
| 403 | Wrong role, or accessing another user's task |
| 404 | Task / user not found, unknown URL |
| 405 / 415 | Wrong HTTP method / content type |
| 409 | Duplicate email, duplicate (double-click) create, lock timeout / concurrent modification |
| 500 | Unexpected error (details are logged, never leaked to the client) |

---

## 📣 Async Event Publishing

`TaskService` publishes a `TaskEvent` (extends Spring's `ApplicationEvent`) for `CREATED`, `UPDATED` and `DELETED`.
`TaskEventListener` handles it **asynchronously** (`@Async` on the `taskEventExecutor` pool) and only **after the transaction commits** (`@TransactionalEventListener(AFTER_COMMIT)`), so rolled-back operations never emit events. Today the listener logs the event — plug in e-mail/audit/notifications there.

---

## 🧪 Running Tests

```bash
cd dhananjay
mvn test
```

| Test class | Type | Covers |
|---|---|---|
| `AuthServiceImplTest` | JUnit 5 + Mockito | Signup (hashing, normalisation, duplicates, race, invalid role, admin-signup flag), login (success, unknown email, wrong password) |
| `TaskServiceImplTest` | JUnit 5 + Mockito | Create (default status, duplicate/double click, validation), read, list/pagination/sorting rules, update (no-op double click, RBAC, past dates), delete, events |
| `JwtUtilsTest` | JUnit 5 | Token generate/parse, tampered, malformed, expired, wrong secret |
| `TaskApiIntegrationTest` | `@SpringBootTest` + MockMvc + H2 | Real security chain: 401/403 handling, RBAC, CRUD, validation, pagination, double click |

Integration tests use an in-memory H2 database (`application-test.properties`), so no MySQL is needed to run `mvn test`.

---

## ⚙️ Configuration Reference

| Property / env | Default | Meaning |
|---|---|---|
| `DB_URL` | local MySQL `ikaansh_task_db` | JDBC URL |
| `DB_USERNAME` / `DB_PASSWORD` | `root` / `root` | Database credentials |
| `JWT_SECRET` | dev key | HMAC secret, ≥ 32 chars |
| `jwt.expiration-ms` | `28800000` (8h) | Token lifetime |
| `ALLOW_ADMIN_SIGNUP` | `true` | Allow self-registration as ADMIN |
| `app.task.duplicate-window-seconds` | `5` | Double-click detection window |

---

**Author:** Dhananjay Kumar  
**Assignment by:** Ikaansh Edutech Private Limited
