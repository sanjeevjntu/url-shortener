# AI interaction log

One entry per meaningful AI interaction: intent, output, **disposition** (accepted / edited / rejected) with rationale,
and verification. No secrets or real data in prompts. Entries are append-only; corrections are new entries.

### 1. Review of the inherited prototype
**Intent:** assess a Kafka/Redis/JPA prototype. **Output:** 10 ranked findings plus process gaps; recommendation to rebuild.
**Disposition:** accepted with verification. Two findings reproduced by running code (`%20` double-encoding; the repo's
own verify script failing); the rest labelled read-only until a test proves them. In-place migration **rejected**.

### 2. Stack and architecture
**Engineer decisions:** Java 25, Boot 4, Gradle; no Keycloak; Postgres plus caching. **Dispositions:** OAuth2/Keycloak
rejected (engineer); JPA rejected after the review (D1); Kafka rejected (D2/D4); Redis deferred; 7-char Base62 **edited**
to 8-char Base58.

### 3. Scaffold and schema
**Dispositions:** UUID PK rejected (index bloat); `CHAR(64)` hex rejected for `bytea`; index-size claim **edited** after
measuring (15 MB vs 18 MB); binding `Instant` directly rejected (pgjdbc cannot); H2 rejected. **Verified:** schema applied
to PostgreSQL 16, races and query plans checked. **Slip:** shell brace expansion created odd folders; caught and cleaned.

### 4. No URL deduplication
The AI had labelled "same URL → same code" an approved constraint; it came from the prototype, not the brief.
**Disposition: rejected/corrected.** Engineer chose no dedupe (D7); `url_hash` and its index removed; query strings kept.

### 5. Service, web and analytics layers; code generator
**Output:** URL guard, Base58 generator (one draw), service with idempotency, controllers, Caffeine cache, analytics,
rate limiting, errors, OpenAPI, Docker, tests. **Dispositions:** Redis rejected for now; editing V1 rejected (already
applied) → V2; per-class `@Container` rejected → singleton; negative-cache bug **edited** (create now evicts);
Spring API versioning rejected (unverified defaults); `@PreDestroy` **edited** to `DisposableBean`; `byte[]` records
**edited** to classes. **Verified:** pure classes compiled and run against 103 assertions; **later the full build ran green
on the engineer's machine**.

### 6. Gap review against the brief
**Output:** requirement-by-requirement table; process deliverables were stale. **Dispositions:** scenarios rewritten;
resilience IT added; CodeQL + dependency review added (OWASP dependency-check rejected: needs an NVD key); ArchUnit listed as
a limitation instead of claimed; k6 numbers **not fabricated**; sign-offs written unsigned.

### 7. Engineer's own generator compared with the AI's
The engineer wrote a Base58 generator; the AI tested it: bound `12806308171808` is 58^8 / 10 (every code starts with `1`–`6`),
and `Math.abs(Long.MIN_VALUE)` stays negative (index out of bounds). **Disposition:** AI version kept; findings recorded as
evidence that outputs are verified, not copied.

### 8. Review-driven refactor
**Engineer feedback:** API key not production-grade (but no Keycloak); rate limiter over-engineered; aliases unnecessary;
uniform errors wanted; fewer docs. **AI proposals and dispositions:**
- Hashed, named, mandatory API keys: **accepted**. Jasypt encrypted properties rejected (moves the secret; Boot 4 support
  unverified). OAuth2 resource server proposed, **rejected by the engineer**.
- Resilience4j for rate limiting: **rejected with rationale** (global limiter, not per client) → Bucket4j accepted; custom
  IP resolver replaced by container configuration.
- Aliases and the expiry sweeper: removed (engineer). V3 migration (audit + tuning): AI's call, accepted.
- `ProblemDetail` + `errorCode`/`timestamp`/`traceId` instead of a custom error class: **accepted** (standard format).
- Docs: ~30 files → 7.
**Verified in sandbox:** syntax of all sources; pure logic unchanged. **To verify:** full build on the engineer's machine.

### 9. SpotBugs findings after the refactor
Two findings: `HRS_REQUEST_PARAMETER_TO_HTTP_HEADER` in `TraceIdFilter` and `EI_EXPOSE_REP2` in `IdempotencyKeyCleanup`.
**Disposition:** the first is a false positive (the header is echoed only after matching `[A-Za-z0-9-]{8,64}`), suppressed
for that class and rule only with the reason in `config/spotbugs/exclude.xml`, rather than obscuring the data flow; the
second matches the existing documented rule for Spring-injected components. **Verified:** next build green (124 tests).

### 10. Security and review hardening
**Engineer feedback:** "why is the key still `dev-secret`?"; a replayed POST returned a deleted link with `200`.
**Dispositions:**
- Replay of a deleted/expired link → `410`: **accepted**. UUID guidance (client-generated) added to Swagger.
- Committed dev key removed; `scripts/new-api-key.sh --local` writes git-ignored files: **accepted**.
- Scopes, optional expiry, failed-auth log/metric/throttle: **accepted**. SHA-256 kept over bcrypt (random keys).
- HEAD not counted, actuator on 8081, IPv6 /64 buckets, one transaction per flush, leftover `custom` flag and create event
  removed: **accepted**.
- **AI self-correction during implementation:** the transactional flush introduced a new failure type
  (`CannotCreateTransactionException`, not a `DataAccessException`) that would have let a database outage fail redirects in
  sync mode. Caught in review before delivery; both recorders and the 503 handler now cover it.
**Verified in sandbox:** syntax of all sources; key script output parsed as YAML/env. **To verify:** full build.
**Follow-up (build feedback):** SpotBugs flagged `EI_EXPOSE_REP2` on the new `ApiKey.scopes` list. Fixed at the source with an
immutable `List.copyOf` in the record (a missing list now fails validation clearly instead of with a NPE), and the documented
config-record suppression was extended to nested records and both EI directions.

### 11. Reviewer experience
**Engineer goal:** "when I give this to the interviewer, I don't want him to struggle"; local `bootRun` and IntelliJ failed
without a key file. **AI proposal:** keep production fail-fast, but in the `local` profile generate a per-run key and print it
(Spring Security's generated-password pattern); run Compose and a shared IntelliJ configuration with `local`; add a
one-command smoke test and a reviewer quick start. **Disposition:** accepted. **Verified in sandbox:** the smoke-test script
was run against a mock server and **failed at first** (it assumed compact JSON); fixed to tolerate whitespace, then 16/16
checks passed, and a wrong key correctly fails. Full build to be run by the engineer.

### 12. Default profile for local runs; documentation pass
**Engineer feedback:** the IntelliJ run button still failed (its own run configuration, no profile), so the production
fail-fast fired. **AI proposal:** a committed `config/application.yml`, read from the working directory, sets
`spring.profiles.default: local`: any start from a checkout gets the generated key, while the Docker image (no `config/`)
and any explicit profile still fail fast; tests set their own default profile. **Rejected by the AI:** that default in
the packaged `application.yml` (any jar without a profile would run with a logged key); IDE detection; the shared run
configuration (removed: the run button ignores it). **Disposition:** accepted. Docs: stale statements fixed (create no
longer evicts; keys can expire; the startup rule), README restructured, wording cut. **Verified in sandbox:** syntax only;
two unit tests added for the profile rule. **To verify:** full build on the engineer's machine.

### 13. Closing link creation
**Engineer question:** "why are POST and GET not secured with the key?" **AI analysis:** redirects must stay public (the
person clicking has no key) and a lookup reveals no more than the redirect, but public create was a real weakness: spam
and phishing, a per-IP limit beaten by many IPs, no audit, and idempotency shared by everyone behind one NAT.
**Disposition:** engineer chose to close it. Implemented: `links:create` scope on `POST`, `created_by` (additive V4),
idempotency scoped per API client, create limit per key; authentication now runs before rate limiting. The engineer
asked whether the k6 script is still needed; the AI recommended keeping it (the brief's performance gate) and updated it
to send the key. **Verified in sandbox:** syntax of all sources; the smoke test against a mock server that requires
the key (17/17; a wrong key fails). **To verify:** full build on the engineer's machine.

### 14. SpotBugs after closing link creation
**Build feedback:** 148 tests passed; `spotbugsMain` failed, and the plugin printed only the exit code. **AI fix, from the
code rather than the (missing) report:** `ApiKeyRegistry` made `final`, because its constructor throws by design
(fail-fast without keys), which SpotBugs reports as `CT_CONSTRUCTOR_THROW` on non-final classes; the dev-key generator
reuses one `SecureRandom` instead of creating one per call. SpotBugs now also writes `build/reports/spotbugs/main.html`,
so findings are readable. **To verify:** full build on the engineer's machine.
