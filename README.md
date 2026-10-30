# AI Consumption Platform

Spring Boot backend for an AI text-generation service with **per-minute rate limiting**, **monthly token quotas**, **subscription plans** and **simulated upgrades**. The core of the design is the **Proxy pattern** with two chained proxies.

```
HTTP request
    |
    v
AIController                      (depends only on the "aiGenerationChain" bean)
    |
    v
RateLimitProxyService             (1st proxy: requests per minute, HTTP 429 + Retry-After)
    |
    v
QuotaProxyService                 (2nd proxy: monthly tokens, HTTP 402)
    |
    v
AIGenerationService  <-- interface
    |
    v
MockAIGenerationService           (replaceable by a real provider)
```

## Technology

Java 17+, Spring Boot 3.3, Maven, Spring Web / Data JPA / Validation / Actuator / Scheduling, Flyway, H2 (local), PostgreSQL (production), JUnit 5, Mockito, Docker.

## Project layout

```
com.example.aiplatform
  config/       PlanConfiguration (all limits), ProxyConfiguration (the chain), CorsConfig, TimeConfiguration (Clock + auditing)
  controller/   AIController, QuotaController
  service/
    ai/         AIGenerationService, MockAIGenerationService
    proxy/      RateLimitProxyService, QuotaProxyService
    ratelimit/  RateLimitService (windows), RequestUsageService (request log)
    quota/      QuotaService
    subscription/ SubscriptionService
    user/       UserService
  model/        entity, dto, enums
  repository/   Spring Data repositories
  exception/    custom exceptions + GlobalExceptionHandler
  scheduler/    RateLimitScheduler, MonthlyQuotaScheduler
  util/         TokenEstimationService
```

## Quick start (local, H2)

Requirements: JDK 17+ and Maven 3.9+.

```bash
mvn clean verify          # compile and run all tests
mvn spring-boot:run       # starts on http://localhost:8080 with the "dev" profile
```

The `dev` profile is the default. It uses an in-memory H2 database, applies the Flyway migrations and loads seed users:

| id | username | plan       |
|----|----------|------------|
| 1  | user1    | FREE       |
| 2  | user2    | PRO        |
| 3  | user3    | ENTERPRISE |

The mock AI sleeps 1200 ms per generation (`app.ai.mock-delay-ms`).

### Try it

```bash
curl -X POST localhost:8080/api/ai/generate -H 'Content-Type: application/json' \
  -d '{"userId":1,"prompt":"Explain the principles of software engineering."}'

curl 'localhost:8080/api/quota/status?userId=1'
curl 'localhost:8080/api/quota/history?userId=1'

curl -X POST localhost:8080/api/quota/upgrade -H 'Content-Type: application/json' \
  -d '{"userId":1,"targetPlan":"PRO"}'

curl localhost:8080/actuator/health
```

## API

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/ai/generate` | Body `{userId, prompt}`. Runs the full proxy chain. Returns `{generatedText, tokensConsumed, timestamp}`. |
| GET | `/api/quota/status?userId=` | `{tokensUsed, tokensRemaining, monthlyTokenLimit, unlimited, resetDate, currentPlan}`. For ENTERPRISE, `unlimited` is `true` and `monthlyTokenLimit` / `tokensRemaining` are `null`. |
| GET | `/api/quota/history?userId=` | Always 7 entries (today and the 6 previous days, oldest first), zero-filled: `[{date, tokensUsed}]`. |
| POST | `/api/quota/upgrade` | Body `{userId, targetPlan}`. Allowed: FREE -> PRO and PRO -> ENTERPRISE. No payment is processed. |
| GET | `/actuator/health` | Health check. |

### Error format

Every error uses the same body (all messages are in English):

```json
{ "status": 429, "error": "Too Many Requests",
  "message": "Request limit exceeded. Please try again later.",
  "timestamp": "2026-10-30T18:00:00" }
```

| Status | When |
|--------|------|
| 400 | Validation failure, malformed JSON, unknown plan name, invalid upgrade transition, missing/invalid parameter |
| 402 | Monthly token quota exhausted |
| 404 | Unknown user or unknown endpoint |
| 429 | Per-minute request limit exceeded. Includes the `Retry-After` header (seconds) |
| 500 | Unexpected error (details are logged, never returned) |

## Plans

All limits live in `PlanConfiguration`. Unlimited is an explicit state (`Limit.unlimited()`), never a large number.

| Plan | Requests / minute | Tokens / month |
|------|------------------|----------------|
| FREE | 10 | 50,000 |
| PRO | 60 | 500,000 |
| ENTERPRISE | unlimited | unlimited |

## How the key requirements are met

**Rate limiting (`RateLimitProxyService` + `RateLimitService`).** Each user has an independent fixed one-minute window in a `ConcurrentHashMap`. The check-and-increment runs inside `compute`, which is atomic per key, so concurrent requests cannot race and users never affect each other. `Retry-After` is the remaining window time rounded up to whole seconds. `RateLimitScheduler` evicts expired windows (they are already ignored at request time, so this only frees memory). Accepted requests are also logged in `request_usage`.

**Monthly quota (`QuotaProxyService` + `QuotaService`).** The proxy loads the user's plan, estimates the prompt tokens and rejects early with 402 when the prompt cannot fit in the remaining quota. After a successful generation it registers the *actual* token count. Tokens are never deducted if generation fails.

**Concurrency safety of the quota.** `QuotaService.recordConsumption` runs in a transaction that locks the user row (`SELECT ... FOR UPDATE`) and re-checks the limit before writing. Concurrent requests for one user are serialized at the database, so total usage can never exceed the limit, even across several application instances. If a response would push usage over the limit, nothing is deducted and the request is answered with 402.

**Billing periods.** A period is the calendar month (`yyyy-MM`) in the configured timezone, derived from the date of each usage row. A new month therefore starts from zero automatically, and history is never reset or deleted. `MonthlyQuotaScheduler` runs at 00:00 on the 1st and creates a zero-usage opening row per user (idempotent).

**Time.** All timestamps, periods and schedules use one injectable `Clock` in the timezone set by `APP_TIMEZONE` (default `UTC`). Tests replace it with a controllable clock, so no test needs `Thread.sleep` to cross a window or a month boundary.

**Replacing the mock AI.** Implement `AIGenerationService` for your provider and return it from the `aiGenerationTarget` bean in `ProxyConfiguration`. The proxies and controllers do not change.

## Security notes

- **`userId` is a development-only identity.** The request body or query parameter is not authentication and **must not be trusted in production**. Anyone can send any `userId`.
- User resolution is centralized in `UserService.getUserOrThrow`. When adding Spring Security, take the user id from the authenticated principal at that single point and remove `userId` from the request DTOs.
- Quota and history queries are always scoped to the requested user, and rate limit state is keyed by user id.
- Database credentials come from environment variables. Nothing secret is committed; copy `.env.example` to `.env` (git-ignored).
- CORS origins are configuration (`CORS_ALLOWED_ORIGINS`). `Retry-After` is exposed to browsers so the React frontend can read it on 429 responses.
- Error responses never include stack traces or internal messages.

## Tests

```bash
mvn test
```

| Area | Test class |
|------|-----------|
| Successful generation, 429 + `Retry-After`, 402, plan limits, history, upgrades, CORS, health | `ApiIntegrationTest` |
| FREE / PRO / ENTERPRISE limits, window reset, isolation, concurrency (100 threads) | `RateLimitServiceTest` |
| Rate limit proxy behavior | `RateLimitProxyServiceTest` |
| Quota proxy behavior (no deduction on failure, enterprise bypass) | `QuotaProxyServiceTest` |
| Proxy execution order | `ProxyChainOrderTest`, `ProxyChainWiringTest` |
| Quota math, history zero-fill, month change, concurrency (20 threads) | `QuotaServiceIntegrationTest` |
| Upgrades and history records | `SubscriptionServiceIntegrationTest` |
| Billing period scheduler | `MonthlyQuotaSchedulerIntegrationTest` |
| Persistence, seed data, constraints, auditing | `PersistenceIntegrationTest` |
| Token estimation | `TokenEstimationServiceTest` |
| Plan configuration and transitions | `PlanAndTransitionTest` |
| Mock AI provider | `MockAIGenerationServiceTest` |

Integration tests use the `test` profile (in-memory H2, Flyway migrations plus seed data, no artificial AI delay).

## Production deployment

### Environment variables

| Variable | Required | Description |
|----------|----------|-------------|
| `SPRING_PROFILES_ACTIVE` | yes | `prod` |
| `DB_URL` | yes | e.g. `jdbc:postgresql://host:5432/aiplatform` |
| `DB_USERNAME` | yes | Database user |
| `DB_PASSWORD` | yes | Database password |
| `CORS_ALLOWED_ORIGINS` | yes | Comma-separated frontend origins, e.g. `https://app.example.com` |
| `APP_TIMEZONE` | no | Default `UTC` |
| `SERVER_PORT` | no | Default `8080` |
| `DB_POOL_SIZE` | no | Default `10` |

The `prod` profile validates the schema (`ddl-auto=validate`) and applies only `db/migration` (no seed users). Flyway creates the schema on first start.

### Docker

```bash
docker build -t ai-consumption-platform .
docker run -p 8080:8080 --env-file .env ai-consumption-platform
```

### Docker Compose (app + PostgreSQL)

```bash
cp .env.example .env      # edit DB_USERNAME, DB_PASSWORD, CORS_ALLOWED_ORIGINS
docker compose up --build
```

### Without Docker

```bash
mvn -DskipTests package
SPRING_PROFILES_ACTIVE=prod DB_URL=... DB_USERNAME=... DB_PASSWORD=... CORS_ALLOWED_ORIGINS=... \
  java -jar target/ai-consumption-platform.jar
```

Use `GET /actuator/health` for load balancer and container health checks.

### Scaling note

Rate limit windows are kept in memory, so the per-minute limit is enforced **per application instance**. The monthly quota is stored in the database and is exact across instances. If you run several instances and need a global per-minute limit, replace `RateLimitService` with a shared store (for example Redis) or use sticky routing per user.

## Known limitations

- Fixed-window rate limiting allows a short burst at a window boundary (up to twice the limit across two adjacent windows).
- A request whose generated text would exceed the remaining quota is rejected after generation with 402 and is not charged.
- Payments are simulated; there is no downgrade or billing integration.
