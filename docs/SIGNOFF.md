# Human sign-off for high-impact changes

**AI cannot sign off.** A human engineer reviews each item and records the decision. Status: all **PENDING**.

## S1. Schema migrations V1–V4
- [ ] `V1`: types, constraints, idempotency table; `V2`: `link_click_stats` (no FK; privacy: host and browser only);
      `V3`: `deleted_by`, expiry index dropped, `fillfactor 90`; `V4`: `created_by` (no index)
- [ ] All additive after first release; rollback = a corrective forward migration
- [ ] `JdbcLinkRepositoryIT` and `LinkApiIT` pass

## S2. URL validation and host guard (security)
- [ ] `UrlValidator` rules (D5), including the accepted gap: hostnames resolving to private IPs are not blocked
- [ ] No DNS lookups (only IP literals reach `InetAddress`)
- [ ] `UrlValidatorTest` passes

## S3. API keys and public contract (security)
- [ ] Only SHA-256 hashes configured; no key in the repository; constant-time comparison
- [ ] Without a configured key, startup fails in every profile except `local`
- [ ] `local` (a generated key printed in the log) never runs in production: deployments use the image (no `config/`)
      or set an explicit profile
- [ ] Scopes: `POST /api/v1/links` needs `links:create`; `DELETE /api/v1/links/{code}` needs `links:delete`;
      `GET /api/v1/links/{code}/stats` needs `stats:read`; redirects and `GET /api/v1/links/{code}` stay public
- [ ] Expiry policy agreed (e.g. 90 days) and rotation owner named; production hashes come from a secret store
- [ ] Failed-auth throttle (burst 10, 5/min per IP) and the `shortener.auth.failures` alert threshold agreed
- [ ] `/v3/api-docs` reviewed: paths, status codes, error format, `Idempotency-Key`

## S4. Rate limits (availability)
- [ ] Values: create burst 10 / 10 per min **per API key**; redirect burst 200 / 6000 per min per client IP (IPv6 /64)
- [ ] Behind a load balancer, forwarded headers are enabled for **its** address range only (otherwise all clients share
      one bucket)
- [ ] Per-instance enforcement accepted (D8)
- [ ] Management port 8081 is not exposed through the public load balancer

| Item | Reviewer | Date | Decision | Notes |
|---|---|---|---|---|
| S1 | | | | |
| S2 | | | | |
| S3 | | | | |
| S4 | | | | |
