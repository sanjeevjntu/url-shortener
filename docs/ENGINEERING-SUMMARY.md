# Engineering summary

> AI assists the engineer within tasks; the engineer owns execution and quality.

## 1. Plan

Done means: `./gradlew build` green, a test that would fail without the change, an AI-log entry, docs updated.
High-impact tasks also need a human sign-off ([SIGNOFF](SIGNOFF.md)).

| Task | Intent | Acceptance criteria | State |
|---|---|---|---|
| T1 | Reproducible build with gates | `build` runs format, tests, SpotBugs, coverage; Compose starts Postgres | Done |
| T2 | Schema + JDBC repository | Same URL twice → two links; taken code → empty; deleted code never reused; 32-way race → 1 winner | Done |
| T3 | URL validation + host guard | `%20` and query preserved; internal hosts rejected; normalisation idempotent | Done |
| T4 | Code generator | Exact vectors; bijection; uniformity; forced collision → retry → `503` | Done |
| T5 | Create/get/delete, idempotency, API key | `201/200/401/403/404/409/410` matrix; per-caller idempotency | Done |
| T6 | Fast redirect | Cache hit/miss/negative; `302/404/410`; single-flight | Done |
| T7/T9 | Analytics, then off the critical path | Clicks counted; redirect survives analytics failure; drops counted; flag rolls back | Done (k6: engineer) |
| T8 | Abuse protection + contract | `429` + `Retry-After`; spoofed XFF ignored; uniform errors; OpenAPI | Done |
| T10 | Ambiguous: expiry, private links | Decision with rejected alternatives; boundary tests | Done (private deferred) |
| T11 | Review-driven refactor | Aliases removed; Bucket4j; hashed mandatory keys; error model; docs consolidated | Done |
| T12 | Security hardening | No committed key; scopes; expiry; auth-failure throttle; replay → `410`; HEAD not counted; actuator on 8081; IPv6 /64 | Done |
| T13 | Reviewer experience | Docker-only quick start; per-run key in `local`; `local` by default from a checkout; smoke test; docs condensed | Done; build to re-run |
| T14 | Close link creation | `POST` without key `401`, without scope `403`; `created_by` stored; same Idempotency-Key from two clients → two links; create limit follows the key across IPs | Done; build to re-run |

## 2. Artifacts

Code by feature in `src/main/java`; Flyway `V1`–`V4`; contract at `/v3/api-docs`; `Dockerfile` (multi-stage, non-root,
AOT cache), `docker-compose.yml`; CI `ci.yml` + `security.yml`; `scripts/` (key generator, smoke test); k6 in `load/`;
traceability: [AI-LOG](AI-LOG.md), [SIGNOFF](SIGNOFF.md), PR template.

## 3. Testing approach and validation

**Pyramid:** pure unit tests → service tests with mocks → integration tests on **real PostgreSQL 17** via Testcontainers
(`JdbcLinkRepositoryIT`, `LinkApiIT`, `ApiKeySecurityIT`, `RateLimitIT`, `RedirectResilienceIT`, `ErrorHandlingIT`) →
smoke test of a running instance → k6. No H2: the guarantees under test are PostgreSQL behaviour.

| Gate | Where |
|---|---|
| Format | Spotless (Palantir) in `build`; `spotlessApply` fixes |
| Static analysis | `-Xlint:all`; SpotBugs (suppressions justified in `config/spotbugs/exclude.xml`) |
| Tests + coverage | JUnit 5 + Testcontainers; JaCoCo ≥ 80% instructions |
| End to end | `scripts/smoke-test.sh <key>`: 16 checks, non-zero exit on failure |
| Security | CodeQL, dependency review on PRs, Dependabot alerts |
| Performance | k6 at a fixed request rate ([load/README.md](../load/README.md)) |
| Human review | Sign-offs for security, schema and rate-limit changes |

**What has actually run** (engineer's machine): T11 green with 124 tests; T12 137 tests; T14 **148 tests, 0 failures**,
coverage 95.1% instructions / 80.8% branches. That run's SpotBugs findings were fixed (AI-LOG 14); the full `build` must
pass once more. The smoke test was run only against a mock server in the AI sandbox. k6: pending (engineer).

## 4. Risks and guardrails

| Risk | Guardrail | Evidence |
|---|---|---|
| Two creates race for one code | `UNIQUE(code)` + `ON CONFLICT DO NOTHING` | 32-way race IT; 40-way psql run |
| Code collision | Bounded redraw + metric | `LinkServiceTest` |
| Deleted code reissued (hijack) | Soft delete + `UNIQUE(code)` | `JdbcLinkRepositoryIT` |
| Redirects to internal hosts | Scheme allowlist, host guard, length cap | `UrlValidatorTest`; S2 |
| Analytics overload hurts redirects | Bounded queue, drop-and-count | `RedirectResilienceIT`, `AsyncClickRecorderTest` |
| Hot rows, cache stampede | Aggregated sorted upserts; Caffeine single-flight | `ClickAggregatorTest`, `LinkCacheTest` (64 threads → 1 load) |
| Spam/phishing via the shortener | Create needs a `links:create` key; per-key limit; `created_by` to find and remove a client's links | `LinkApiIT`, `RateLimitIT`; S3 |
| Abuse, key guessing | Per-IP redirect buckets (IPv6 by /64), failure throttle; spoofed XFF ignored | `RateLimitIT`, `RateLimiterTest` |
| Management vandalism | Hashed, scoped, expiring keys; none committed; create/delete audit | `ApiKeySecurityIT`, `ShortenerPropertiesTest` |
| Production started without keys | Fail-fast outside `local`; an explicit profile wins over the checkout default | `ApiKeyRegistryTest`; S3 |
| Inflated click counts | HEAD not counted | `LinkApiIT.stats_*` |
| Leaked internals | Generic `500` + `traceId`; actuator on its own port | `ErrorHandlingIT`; config |
| DB outage | Cached redirects continue; others fail fast with `503` | Design only (no automated outage test) |

## 5. Assumptions

As in [ARCHITECTURE §1](ARCHITECTURE.md#1-problem-and-assumptions), plus: single region; API keys are for operators;
analytics days are UTC; the service never fetches targets.

## 6. Limitations

- Per-instance cache and rate limits (D2 has the Redis path).
- Keys identify integrations, not end users: no per-user ownership or private links.
- No malware/phishing reputation check; hostnames resolving to private IPs are not blocked.
- Analytics approximate under overload; up to 1 s of clicks lost on a crash.
- Sync and async analytics shipped in one change behind a flag, not as baseline + refactor commits.
- No architecture-rule tests (ArchUnit); package boundaries rely on review and event-based decoupling.

## 7. Ownership

Every AI output is untrusted until reviewed and tested. The engineer approves each change (PR template), signs the
high-impact records, and owns correctness, maintainability and production readiness.
