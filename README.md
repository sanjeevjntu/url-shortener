# URL Shortener

[![ci](https://github.com/sanjeevjntu/url-shortener/actions/workflows/ci.yml/badge.svg)](https://github.com/sanjeevjntu/url-shortener/actions/workflows/ci.yml)

Short links, fast redirects, click analytics that never slow a redirect, and safe behaviour under failure and abuse.
Java 25 · Spring Boot 4.1 · PostgreSQL 17 · plain JDBC · Caffeine · Bucket4j · Flyway · Testcontainers.
An **AI-assisted engineering** exercise: AI drafted; the engineer decided, verified and owns the result
([AI-LOG](docs/AI-LOG.md)).

## Quick start (Docker only)

```bash
git clone https://github.com/sanjeevjntu/url-shortener.git && cd url-shortener
docker compose --profile app up --build -d    # first build takes a few minutes
docker compose logs app | grep X-API-Key       # this run's API key (empty? wait a few seconds)
scripts/smoke-test.sh <key>                    # every endpoint in ~10 s, ✓/✗ per check
```

Swagger UI: <http://localhost:8080/swagger-ui.html> (**Authorize** with the key) · health and metrics:
<http://localhost:8081/actuator> · stop: `docker compose --profile app down` (`-v` also deletes the data).
Port 5432, 8080 or 8081 taken? Change the left-hand port in `docker-compose.yml`.

## Documentation

| | |
|---|---|
| [ARCHITECTURE](docs/ARCHITECTURE.md) | Start here: components, flows, code algorithm, data model, failure modes |
| [DECISIONS](docs/DECISIONS.md) | D1–D9, each with the alternatives rejected |
| [SCENARIOS](docs/SCENARIOS.md) | Greenfield, brownfield and ambiguous scenarios |
| [ENGINEERING-SUMMARY](docs/ENGINEERING-SUMMARY.md) | Plan, quality gates, test results, risks, limitations |
| [AI-LOG](docs/AI-LOG.md) · [SIGNOFF](docs/SIGNOFF.md) | Each AI interaction and its disposition · human approval of high-impact changes |

## Reviewer checklist (assignment brief → evidence)

| Brief asks for | Where to verify | How to check it yourself |
|---|---|---|
| Requirement understanding | [ARCHITECTURE §1](docs/ARCHITECTURE.md#1-problem-and-assumptions): each ambiguity and its resolution | Compare with the API table below |
| Task decomposition | [ENGINEERING-SUMMARY §1](docs/ENGINEERING-SUMMARY.md#1-plan): T1–T14 with acceptance criteria | Each criterion names the test that proves it |
| Greenfield, brownfield, ambiguous | [SCENARIOS](docs/SCENARIOS.md) | Brownfield: V2–V4 are additive, V1 untouched (`src/main/resources/db/migration`) |
| Defensible decisions | [DECISIONS](docs/DECISIONS.md) D1–D9, each with rejected alternatives | e.g. D3 (codes), D6 (API keys), D8 (rate limits) |
| AI traceability | [AI-LOG](docs/AI-LOG.md): 14 entries, accepted / edited / rejected and why | Entries 4, 7, 10, 14 show AI output rejected or corrected |
| Quality gates | `./gradlew build`: Spotless, `-Xlint`, SpotBugs, tests, JaCoCo ≥ 80%; CI in `.github/workflows` | Run it; reports in `build/reports/` |
| Testing | Unit + Testcontainers ITs on real PostgreSQL ([§3](docs/ENGINEERING-SUMMARY.md#3-testing-approach-and-validation)) | `scripts/smoke-test.sh <key>` against the running app |
| Security | API-key scopes, hash-only keys, URL host guard, rate limits ([D5, D6, D8](docs/DECISIONS.md)) | Swagger: `POST` without key → `401`, `http://169.254.169.254/` → `400`; missing scope → `403` in `ApiKeySecurityIT` |
| Reliability | Redirects survive analytics/DB failures ([ARCHITECTURE §7](docs/ARCHITECTURE.md#7-failure-modes)) | `RedirectResilienceIT`; `docker compose stop postgres`, then a cached link still redirects |
| Human sign-off | [SIGNOFF](docs/SIGNOFF.md) S1–S4 | Signed by the engineer, never by AI |
| Limitations, risks | [ENGINEERING-SUMMARY §4, §6](docs/ENGINEERING-SUMMARY.md#4-risks-and-guardrails) | Stated openly, with upgrade paths |

## Develop (JDK 25 + Docker)

```bash
docker compose up -d postgres     # database only
./gradlew bootRun                 # or the IntelliJ run button on ShortenerApplication
./gradlew spotlessApply build     # format, compile, unit + Testcontainers tests, SpotBugs, coverage ≥ 80%
```

Started from the project folder, the app uses the `local` profile (`config/application.yml`) and prints a **generated API
key** (`X-API-Key: …`, new on every restart). For a stable key run `scripts/new-api-key.sh dev --local` once and restart.
Anywhere else, including the production image, it refuses to start without configured key hashes. `Ctrl+C` stops
gracefully (queued clicks are flushed). Reports: `build/reports/`. Load test: [load/README.md](load/README.md).

## API

| Method | Path | Auth | Responses |
|---|---|---|---|
| `POST` | `/api/v1/links` `{url, expiresAt?}`, optional `Idempotency-Key` | key with `links:create`, rate limited per key | `201`, `200` (replay), `400`, `401`, `403`, `409`, `410`, `429` |
| `GET` | `/api/v1/links/{code}` | public | `200`, `400`, `404`, `410` |
| `DELETE` | `/api/v1/links/{code}` | key with `links:delete` | `204`, `401`, `403`, `404` |
| `GET` | `/{code}` | public, rate limited | `302`, `404`, `410`, `429` |
| `GET` | `/api/v1/links/{code}/stats?days=30` | key with `stats:read` | `200`, `400`, `401`, `403`, `404` |

Redirects and link lookups are public: whoever clicks a short link has no key. Every error is
`application/problem+json` with `errorCode`, `timestamp` and `traceId` (also the `X-Trace-Id` header).
**Idempotency-Key:** the client generates a UUID per new link (`uuidgen`) and resends it only when retrying, getting the
original link back instead of a duplicate; keys are scoped to the API key. Different body: `409`. That link since
deleted or expired: `410`.

```bash
KEY=<the X-API-Key from the log>
curl -s -XPOST localhost:8080/api/v1/links -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
     -H "Idempotency-Key: $(uuidgen)" -d '{"url":"https://spring.io/projects/spring-boot"}'  # 201 {"code":"8QKHT6R5",…}
curl -si localhost:8080/8QKHT6R5                                              # 302 Location: https://spring.io/…
curl -s  localhost:8080/api/v1/links/8QKHT6R5/stats -H "X-API-Key: $KEY"
curl -si -XDELETE localhost:8080/api/v1/links/8QKHT6R5 -H "X-API-Key: $KEY"   # 204; the short link now answers 410
```

## Production keys and configuration

Only SHA-256 **hashes** of named, scoped, optionally expiring keys are configured, never the keys
([D6](docs/DECISIONS.md#d6-api-keys-named-scoped-hash-only)). Scopes: `links:create`, `links:delete`, `stats:read`.

```bash
scripts/new-api-key.sh dashboard --scopes stats:read   # prints the key (for the client) and its hash (for the server)
SHORTENER_APIKEYS_0_NAME=dashboard SHORTENER_APIKEYS_0_SHA256=<hash> SHORTENER_APIKEYS_0_SCOPES=stats:read
```

Other settings live under `shortener.*` in `src/main/resources/application.yml` and can be overridden by environment
variables; database: `DB_URL`, `DB_USER`, `DB_PASSWORD`. Behind a load balancer, trust forwarded headers from its address
range only (example in `application.yml`).
