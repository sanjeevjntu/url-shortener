# Scenarios: greenfield, brownfield, ambiguous

Each shows decomposition, AI-assisted execution and validation. Decisions: [DECISIONS](DECISIONS.md); AI interactions:
[AI-LOG](AI-LOG.md).

## 1. Greenfield: the core shortener

**Decomposition** (tasks and acceptance criteria: [ENGINEERING-SUMMARY](ENGINEERING-SUMMARY.md#1-plan)): build + gates →
schema + repository → URL guard and code generator (pure, verified first) → create/get/delete + idempotency + API key →
redirect + cache → analytics → rate limits, error model, OpenAPI. Uniqueness and soft delete live in the schema first,
so later layers rely on a proven guarantee instead of re-checking.

| Step | AI produced | Engineer decision |
|---|---|---|
| Schema | Tables, JDBC repository, ITs | UUID PK rejected (index bloat); hex `CHAR(64)` rejected for `bytea`; an index-size claim corrected after measuring |
| Code generator | Per-character draws, then one draw + fixed-width Base58, with measurements | One draw (D3). The engineer's own version, when tested, had a 10× too small bound and a `Math.abs(Long.MIN_VALUE)` bug |
| URL guard | Validator | Query strings kept; numeric-host tricks and IPv4-mapped IPv6 added after review |
| Cache | Caffeine | Redis rejected for one node (D2) |

**Validation:** 32-way race on one code → one winner (IT), plus a 40-way psql run; Base58 vectors and an exhaustive
width-3 bijection; chi-square uniformity per position; forced collisions → retry, then `503`; the HTTP status matrix; 27
malicious URL shapes rejected.

## 2. Brownfield: changing code that already exists

**A. Inherited prototype (Kafka/Redis/JPA).** Reviewed before writing anything: 10 findings, e.g. a public `DELETE`,
spoofable `X-Forwarded-For` rate limiting with a non-atomic `INCR` + `EXPIRE`, double-encoded `%20` (reproduced), stored
client geo headers, silent analytics loss, unbounded global idempotency keys. Rebuilt rather than patched (removing
Kafka/Redis/Resilience4j left about half the code). Each finding has a regression test: `LinkApiIT.delete_requiresApiKey_*`,
`RateLimitIT`, `UrlValidatorTest` (`%20`), `AsyncClickRecorderTest`, `LinkApiIT.idempotencyKey_*`, `ClickEventTest`.

**B. Extending the running codebase.** Impact analysis before each change:

| Existing piece | Constraint found | Decision |
|---|---|---|
| Flyway V1, applied on the engineer's DB | Editing it breaks the checksum | Additive V2 (analytics), V3 (audit, tuning), V4 (creator) |
| `@Container` per IT class | A second class restarts Postgres under a cached Spring context | Singleton container |
| `Link.isServableAt` | Already encodes lazy expiry | Reused by redirects; the expiry sweeper removed as redundant |

**C. Analytics off the critical path (sync → async).** The baseline (`mode=sync`) writes inside the redirect, so a slow
analytics write slows every redirect. Change: bounded queue, 1 s aggregated flush, drop-and-count, graceful drain; the
property is the rollback switch (both modes shipped in one change, behind the flag). Validation: `RedirectResilienceIT`
(storage failing → still `302`), `AsyncClickRecorderTest`, `ClickAggregatorTest` (1,000 clicks → 3 upserts); k6 sync vs
async in `load/` (numbers by the engineer).

**D. Refactor after review feedback.** The engineer judged parts over-engineered or not production-grade:

| Change | Impact traced | Validation |
|---|---|---|
| Custom aliases removed | DTO, service, alias rules, tests; `is_custom` column kept (V1 applied) | `LinkServiceTest`, `LinkApiIT` |
| Hand-written rate limiter → Bucket4j | Interceptors, configuration | `RateLimitIT`, `RateLimiterTest` |
| Plaintext key → hashed, named, scoped, expiring keys; none committed; guessing throttled | Properties, interceptor, V3 `deleted_by`, Compose, Dockerfile, key script | `ApiKeySecurityIT`, `RateLimitIT`, `ShortenerPropertiesTest` |
| Uniform errors: `ErrorCode`, catch-all, `traceId` | Every throw site, the advice, a new filter | `ErrorHandlingIT` |
| Replay of a deleted link → `410` | `LinkService.replay` | `LinkServiceTest`, `LinkApiIT` |
| HEAD not counted; actuator on 8081; IPv6 /64; one transaction per flush | Redirect, config, rate limiter, recorders, error handler (new `CannotCreateTransactionException` kept from failing redirects) | `LinkApiIT`, `RateLimiterTest` |
| Reviewer experience: per-run key in `local`, `local` by default from a checkout, smoke test | Key registry, `config/application.yml`, test task, scripts | `ApiKeyRegistryTest`, `scripts/smoke-test.sh` |
| Create closed: `links:create` scope, `created_by` (V4), idempotency and create limit per key | Controller, interceptor order, rate limiter, service, repository, V4, key script, smoke test, k6 setup, every IT that creates links | `ApiKeySecurityIT`, `LinkApiIT`, `RateLimitIT`, `LinkServiceTest` |

## 3. Ambiguous: "links should expire, and some should be private"

| Phrase | Readings | Chosen | Why |
|---|---|---|---|
| expire | absolute time; TTL; after N clicks | Absolute UTC `expiresAt` | Unambiguous across time zones; click limits need a write on the hot path |
| expired link | `404`; `410`; notice page | `410 Gone` | It existed and won't return; `404` stays for never-existed |
| reuse after expiry | yes; no | Never | A recycled code hijacks old printed links |
| private | unlisted; sign-in; password | Deferred (D9) | Codes are already unguessable; access control needs user identity |

**Validation:** the expiry boundary is exclusive (`LinkTest`); expired → `410` and status `EXPIRED` (`LinkApiIT`); a past
`expiresAt` is rejected; deleted codes are never reissued (`JdbcLinkRepositoryIT`).
**Open questions for a product owner:** is a click limit needed? What does "private" mean to users? Should private links
hide `410` vs `404`? Who may read analytics once ownership exists?
