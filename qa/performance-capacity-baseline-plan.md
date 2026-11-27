# HeapCare Performance and Capacity Baseline Plan

**Status:** Planned; execution requires a controlled local/staging environment  
**Created:** 2026-11-27  
**Scope:** HeapCare hospital workflows only

## 1. Purpose

Establish a repeatable, evidence-based performance baseline for the current HeapCare implementation after the successful post-migration local QA run. This plan is not a claim that load testing has already been executed.

The baseline must measure the API, PostgreSQL, Web application, authentication dependency, and observability overhead under representative synthetic hospital workloads. It must identify saturation points and bottlenecks before any production capacity recommendation is made.

## 2. Safety and environment

- Run first against an isolated local or dedicated staging stack, never a production hospital environment.
- Use synthetic tenants, users, patients, appointments, queue entries and billing records only.
- Use a dedicated test tenant and deterministic data seeding. Do not reuse or mutate real patient records.
- Confirm the target base URL and environment before every run.
- Start with a smoke test, then ramp gradually. Stop on sustained elevated errors, unhealthy containers, database saturation, or unexpected resource growth.
- Do not run destructive cleanup against shared PostgreSQL databases or persistent Docker volumes.
- Do not include credentials, access tokens, patient data or secret-bearing headers in artifacts.
- Record the exact commit SHA, container image identifiers, configuration profile, machine resources and test-tool version for reproducibility.

## 3. Workload model

Use a workload mix based on the currently implemented OPD MVP. Keep each scenario independently identifiable in the results.

| Scenario | Representative operations | Primary signals |
|---|---|---|
| Authentication/session | OIDC login setup outside the hot request loop; authenticated token reuse for API load | Auth failures, token validation latency |
| Tenant/hospital reads | Tenant context, hospital and department listing | API latency, DB query latency |
| Patient search and registration | Duplicate-candidate search, MRN lookup, synthetic registration | p95/p99 latency, DB writes, uniqueness conflicts |
| Scheduling | Availability reads, appointment creation and check-in | Throughput, contention, conflict rate |
| Queue operations | Queue listing, walk-in registration, call/start transitions | Latency, state conflicts, write contention |
| Billing | Service-item reads, invoice creation, payment and receipt reads | Transaction latency, DB pool and lock waits |
| Clinical encounter | Encounter read, sign and amendment workflow | Authorization overhead, transaction latency |
| Audit and observability | Audit writes/reads plus normal telemetry ingestion | Audit persistence latency, collector/ES resource use |

Do not combine unrelated high-cost operations into one undifferentiated request loop. Report each scenario separately and as a weighted mixed workload.

## 4. Test stages

1. **Preflight:** Verify health/readiness endpoints, migrations, database connectivity, telemetry targets and clean synthetic test data.
2. **Smoke:** One to five virtual users for two minutes; validate scripts and assertions.
3. **Warm-up:** Gradually increase to 10 virtual users; allow caches and JIT to stabilize.
4. **Baseline:** Run 25 concurrent virtual users for 10 minutes.
5. **Step load:** Run 50, then 100 virtual users for 10 minutes per step, only while health and error guardrails remain satisfied.
6. **Stress/limit discovery:** Increase in controlled increments only after reviewing step-load results. Stop at the first sustained saturation signal; do not pursue a failure for its own sake.
7. **Soak:** Run the agreed representative load for 30–60 minutes after short-run stability is demonstrated.
8. **Recovery:** Reduce load to zero and verify that queues, connection pools, CPU/memory and telemetry backlogs return to baseline.

The values above are initial test stages, not product capacity claims or production SLAs. Adjust them to the available test machine and observed saturation signals.

## 5. Measurements

For each endpoint and scenario capture:

- Request count, throughput (requests/second), successful and failed requests.
- HTTP status distribution, timeouts and retries.
- p50, p90, p95, p99 and maximum latency.
- API process CPU, memory, GC pause/allocation indicators and thread-pool pressure where available.
- PostgreSQL CPU, memory, active/idle connections, pool wait, slow queries, lock waits, deadlocks, transaction duration and storage I/O where available.
- Container CPU/memory, restarts, health status and network I/O.
- OTel Collector queue/backpressure/dropped telemetry, Prometheus scrape health, Jaeger trace sampling and Elasticsearch indexing/ingest pressure.
- Browser/Web response metrics for representative user journeys, separately from API-only load.

Correlate test timestamps with trace IDs and structured logs. Redact sensitive values before saving artifacts.

## 6. Initial stop conditions

Stop the current stage and preserve evidence if any of the following occurs:

- Readiness/health check fails or a required container becomes unhealthy.
- Sustained HTTP 5xx, unexpected 429s, timeouts or correctness assertion failures.
- Database connection pool exhaustion, deadlocks or sustained lock waits.
- Unbounded memory growth, repeated container restarts or OOM events.
- Telemetry queue growth or sustained dropped spans/metrics/logs.
- Any cross-tenant data exposure, authorization bypass or unexpected mutation outside the synthetic test tenant.

A 429 caused by an intentionally configured rate limit must be reported separately from application failures and must not be hidden.

## 7. Correctness and isolation assertions

Performance results are invalid unless the workload also verifies correctness:

- Every request is scoped to its intended synthetic tenant.
- Cross-tenant reads and writes remain denied.
- Patient MRNs remain unique and immutable under concurrent registration.
- Appointment slot and queue state transitions remain consistent under contention.
- Invoice/payment totals and settlement state remain consistent.
- Audit records are persisted for required actions and denials.
- No patient or tenant records leak between test identities.

## 8. Reporting format

Publish a report under `HeapCare.Doc/qa/performance-capacity-baseline-YYYY-MM-DD.md` containing:

1. Executive summary and verdict.
2. Commit SHA and environment inventory.
3. Test tool, scripts, command lines and scenario definitions.
4. Machine/container/database resource profile.
5. Per-scenario throughput and latency percentiles.
6. HTTP status/error distribution.
7. API, database and observability resource graphs.
8. Bottlenecks and trace/log evidence.
9. Correctness and tenant-isolation results.
10. Recommended fixes ranked by impact and confidence.
11. Retest comparison after each fix.
12. Explicit limitations and untested dimensions.

Store raw machine-readable output and dashboards/screenshots alongside the report. Never claim capacity based on the earlier functional QA request timings alone.

## 9. Exit criteria

The baseline is complete only when:

- The test is repeatable from documented commands.
- The environment and data are isolated and identified.
- All scenarios have correctness assertions.
- Latency, throughput, errors and resource consumption are captured.
- At least one controlled step-load and one recovery run complete.
- Bottlenecks are either fixed and retested or accepted as documented constraints.
- The resulting capacity statement is explicitly limited to the tested environment and workload.

Production capacity and SLA targets must be set only after this baseline and stakeholder review.