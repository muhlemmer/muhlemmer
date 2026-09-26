# Tim Möhlmann

Backend engineer and software architect at [ZITADEL](https://github.com/zitadel), building open-source identity infrastructure in Go and PostgreSQL.
My work centres on **database-backed performance**, **authorization architecture** and **security primitives**.

This page supplements my CV. Every claim below links to the public issue, pull request or repository behind it.

---

## 🔐 [zitadel/passwap](https://github.com/zitadel/passwap): principal author

A Go library that puts many password hashing algorithms behind one API. It lets an identity provider change algorithms or cost parameters without forcing users to reset their passwords.

- **Created it in September 2022** and am its main author: 36 of 47 human-authored commits.
- One `Swapper` API: new hashes use the configured algorithm, while `Verify` still accepts legacy hashes and **transparently returns an upgraded hash** when the algorithm or parameters are outdated.
- Supports argon2i/id, bcrypt, scrypt, pbkdf2 and sha2-crypt, plus verify-only legacy formats (md5-crypt, salted MD5, phpass for WordPress/phpBB, Drupal 7) for migrating users from other systems.
- Uses the Modular Crypt Format, tested against reference hashes from Python's Passlib. The only dependencies are the Go standard library and `golang.org/x`.
- **Used in production by ZITADEL** for user passwords and, since [zitadel#7657](https://github.com/zitadel/zitadel/pull/7657), for machine and app secrets.

---

## ⚡ [zitadel/zitadel](https://github.com/zitadel/zitadel): performance engineering

Over 35 performance pull requests on ZITADEL's event-sourced core, running from 2023 to today. Measured with ZITADEL's own published k6 benchmarks:

| Endpoint | First benchmark | Latest ([v4.17.1](https://github.com/zitadel/zitadel/pull/12651)) | Improvement |
| --- | ---: | ---: | ---: |
| Token introspection | [18 req/s](https://zitadel.com/docs/apis/benchmarks/v4/introspect) (v4.0.0-rc2, p50 26 s) | [**2,297 req/s**](https://zitadel.com/docs/apis/benchmarks/v4.17.1/introspect) (p50 46 ms) | **~127×** |
| Service-account JWT-profile grant | [193 req/s](https://zitadel.com/docs/apis/benchmarks/v2.65.0/machine_jwt_profile_grant) (v2.65.0) | [**1,076 req/s**](https://zitadel.com/docs/apis/benchmarks/v4.17.1/machine_jwt_profile_grant) | **~5.6×** |

In the latest runs the database CPU is the limit (96–98%) while ZITADEL itself uses under 40% CPU, so the application is no longer the bottleneck.

### OIDC hot paths: introspection, token and service-account auth
Each approach collapses many database round trips into a single query and takes projection updates off the request path.

| PR | Change |
| --- | --- |
| [#6909](https://github.com/zitadel/zitadel/pull/6909) | Introspection endpoint reduced to ~3 queries. Client and token verification run in parallel, and userinfo is returned as a single JSON document from the DB. |
| [#6999](https://github.com/zitadel/zitadel/pull/6999) | Client verification on the token endpoint: config, roles, secret and keys fetched in **one** query. |
| [#7706](https://github.com/zitadel/zitadel/pull/7706) | Userinfo endpoint rebuilt on the same single-query design. |
| [#7822](https://github.com/zitadel/zitadel/pull/7822) | Token creation rewritten on a single query of the write models it needs. |
| [#8580](https://github.com/zitadel/zitadel/pull/8580) | JWT-profile grant: key and user fetched in one joined query. This removes a projection-triggering lookup that hurt concurrency. |
| [#8691](https://github.com/zitadel/zitadel/pull/8691), [#8481](https://github.com/zitadel/zitadel/pull/8481) | Removed per-request event writes that caused index lock contention and sequence collisions under concurrent machine logins and introspection. |
| [#8738](https://github.com/zitadel/zitadel/pull/8738) | Token verifier query now matches a full index instead of a sequential scan. |
| [#7657](https://github.com/zitadel/zitadel/pull/7657) | Machine and app secrets moved to passwap with a much cheaper default hash cost for high-entropy client secrets. |
| [#9092](https://github.com/zitadel/zitadel/pull/9092) | Event push function rewritten in PL/pgSQL, fixing throughput that fell from 836 to 130 req/s during a load test. Closed the *1000 machine authentications/s* goal ([#8352](https://github.com/zitadel/zitadel/issues/8352)). |

**Other current benchmarks** ([#12651](https://github.com/zitadel/zitadel/pull/12651), 600 VUs, 30 minutes): PAT login 3,452 req/s, client credentials 834 req/s.

### Eventstore and projections
- [#12753](https://github.com/zitadel/zitadel/pull/12753): made projection catch-up reads efficient under a large backlog. The batch query went from **832 ms to 4 ms (~200×)** and drain throughput from **975 to 7,811 events/s (~8×)** with 4.8M events of backlog.
- [#8940](https://github.com/zitadel/zitadel/pull/8940), [#8788](https://github.com/zitadel/zitadel/pull/8788), [#8747](https://github.com/zitadel/zitadel/pull/8747), [#10564](https://github.com/zitadel/zitadel/pull/10564): index-only event filtering, removal of costly always-running reducers, and direct queueing for Actions v2 instead of an all-instance projection.
- [#12449](https://github.com/zitadel/zitadel/pull/12449): autovacuum tuning for the ever-growing events table.

### Caching layer, designed and built from scratch
- [#8628](https://github.com/zitadel/zitadel/pull/8628): generic cache interface with in-memory and no-op backends.
- [#8703](https://github.com/zitadel/zitadel/pull/8703): PostgreSQL connector on unlogged tables, partitioned by cache.
- [#8822](https://github.com/zitadel/zitadel/pull/8822), [#8890](https://github.com/zitadel/zitadel/pull/8890), [#10658](https://github.com/zitadel/zitadel/pull/10658): Redis connector with atomic Lua scripts, a circuit breaker, and non-blocking `UNLINK` deletion.
- [#8903](https://github.com/zitadel/zitadel/pull/8903): instance and organization caching on every request.

### Authorization in the database
- [#9152](https://github.com/zitadel/zitadel/pull/9152), [#9677](https://github.com/zitadel/zitadel/pull/9677): role permission checks moved into PL/pgSQL over a denormalized fields table, replacing per-row checks in application code.

---

## 🏗️ [zitadel/nextgen](https://github.com/zitadel/nextgen): architecture epics

As an architect on ZITADEL's next-generation platform I write the epics that set its design direction. Both are in progress.

### Permission management: relational fine-grained authorization, [#419](https://github.com/zitadel/nextgen/issues/419)
I designed a **portable, relational FGA (fine-grained authorization) core**. One model covers ZITADEL's own resources and customers' application permissions. The design is recorded in ADRs 032–034 ([#493](https://github.com/zitadel/nextgen/pull/493)).
- **OpenFGA as the policy language only.** ZITADEL does its own validation, compilation and SQL query planning on **PostgreSQL and Spanner**.
- **Authorization data lives next to the resources it protects:** no sidecar and no synchronization lag.
- **Role implications are resolved once, when a policy is uploaded,** not on every write. Adding a user to a group of 100k members writes one row, not 100k.
- **List endpoints get the authorization filter injected into their SQL,** so there are no O(n) per-row checks.
- A strict 403-vs-404 rule avoids leaking whether a resource exists, and a full audit trail covers delegated and agent actions.
- Split into 9 sub-issues and delivered with the team. The core catalog, compiler, schema and resolver are done.

### Performance: benchmarking and diagnosability, [#1094](https://github.com/zitadel/nextgen/issues/1094)
This epic sets up the platform's first performance program, to measure the architectural bet of relational storage over an eventstore before public launch.
- An exploratory profiling run exposed bottlenecks nobody had diagnosed: an uncached RSA-4096 key unwrap on every request, per-request audit writes, and logging overhead costing ~23% of throughput.
- Designed a k6 load harness that runs the same deployment against **SQLite, PostgreSQL and Spanner**, so a throughput ceiling can be attributed to the database or to the application. It adds application-level tracing and metrics.
- Scoped into 19 sub-issues across six dependency waves, covering 28 API operations in the first wave. The first finding already turned into a key-chain cache (854 µs → 30 ns per lookup).

---

<sub>Go · PostgreSQL · Cloud Spanner · Redis · OIDC / OAuth 2.0 · Event sourcing · Authorization (FGA / ReBAC) · Performance engineering</sub>
