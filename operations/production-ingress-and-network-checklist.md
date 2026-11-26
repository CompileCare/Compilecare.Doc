# Production Ingress and Network-Security Acceptance Checklist

**Status:** Deployment acceptance baseline. This checklist is provider-neutral and is not evidence that a production ingress, domain, certificate, firewall or network policy has been configured.

## 1. Scope and trust boundaries

CompileCare consists of a browser workspace, ASP.NET Core API, PostgreSQL persistence, OIDC identity provider, and internal observability services. The browser-facing Web and API origins are separate trust boundaries unless deliberately deployed same-origin.

The API maps `/metrics` through `MapPrometheusScrapingEndpoint()` without user-authentication metadata, but outside Development/Testing it now requires an explicit `Observability:Metrics:AllowedNetworks` CIDR allowlist. Requests outside those networks receive 404, and startup fails if the allowlist is missing, malformed or unrestricted (`/0`). Treat this as an internal monitoring endpoint: **do not expose `/metrics` through the public Internet ingress**. Permit access only from the approved monitoring network or an authenticated internal scrape path. The application CIDR guard is defense in depth; it does not replace the ingress/network boundary.

The API also exposes liveness/readiness endpoints through the shared observability package. Decide which probes must be reachable by the platform load balancer and prevent unrelated public access. Do not expose Development-only OpenAPI in production.

## 2. Required deployment decisions

Record these before production deployment:

| Decision | Required value / evidence | Current status |
|---|---|---|
| Public Web hostname | Exact FQDN and DNS owner | Not selected |
| Public API hostname | Exact FQDN or same-origin routing decision | Not selected |
| OIDC issuer/authority | Exact HTTPS issuer and discovery URL | Not selected |
| TLS termination | Provider/ingress, certificate source, renewal and expiry alerting | Not selected |
| Internal service network | API, database, identity, observability and worker network boundaries | Not selected |
| Trusted proxy addresses | Exact proxy/load-balancer IPs observed by API | Not selected |
| Public path routing | Explicit allowlist of Web/API paths; block internal routes | Not selected |
| Metrics scrape source | Monitoring identity/network and credential strategy; configure `Observability:Metrics:AllowedNetworks` | Not selected |
| Health probe source | Load balancer/orchestrator source and permitted paths | Not selected |
| Rate limiting | Edge and API thresholds, burst behavior and shared quota strategy | Requires deployment sizing |
| Data residency | Region and any cross-region replication constraints | Requires hosting/legal decision |

## 3. TLS, HTTPS and HSTS

- Terminate TLS at the approved ingress/load balancer using a valid certificate for every public hostname.
- Redirect all public HTTP requests to HTTPS before reaching the browser application or API. Do not rely on HSTS as a substitute for the initial HTTP redirect.
- Configure certificate issuance, automated renewal, expiry alerts, and a tested emergency replacement procedure.
- The API currently emits a one-year HSTS max-age on requests it recognizes as trusted HTTPS. `includeSubDomains` and `preload` are intentionally disabled. Configure edge HSTS consistently and do not enable either option until every affected subdomain and preload commitment has been reviewed.
- Ensure forwarded scheme and client-IP headers are accepted only from the exact trusted proxy IPs configured in `ReverseProxy:KnownProxies`. The API uses one trusted hop and requires symmetric forwarded headers; do not trust arbitrary client-supplied `X-Forwarded-*` values.
- Verify that the application observes HTTPS after forwarded-header processing and that HSTS is not emitted for an untrusted forwarded scheme.

## 4. Browser security policy alignment

- Configure the Web container's `CSP_CONNECT_SRC` with the exact HTTPS API and OIDC origins used by the browser.
- Configure `CSP_FRAME_SRC` with the exact OIDC origin required for silent renewal. Do not add broad wildcard sources or guessed third-party domains.
- The Web Nginx container validates these origins at startup, rejects missing/malformed/wildcard origins and rejects HTTP unless the local-development-only opt-in is explicitly enabled.
- The enforced CSP restricts scripts and styles to `'self'`, blocks objects and framing, and does not allow inline scripts, inline styles or `unsafe-eval`.
- Configure API `Cors:AllowedOrigins` with the exact HTTPS Web origin only when the Web/API deployment is cross-origin. Omit the list for same-origin deployment.
- CORS and CSP are separate controls: verify both with the actual deployed Web, API and identity-provider origins. A valid CSP does not replace API CORS, and CORS does not replace CSP.
- Test OIDC discovery, authorization redirect, callback, token exchange, silent renewal, logout and API requests in the actual browser/ingress topology. Verify any additional required origin individually before allowing it.

## 5. Public and internal route exposure

| Route / service | Exposure rule |
|---|---|
| Web workspace `/` and static assets | Public HTTPS entry point; authentication/authorization enforced by the application for protected workflows |
| API business routes `/api/v1/*` | Public HTTPS only when required by the Web client or approved integrations; every route must retain its authentication, tenant, organization-scope and permission policy |
| API `/metrics` | **Internal monitoring only**; never route from the public Internet ingress |
| API liveness/readiness | Allow only the orchestrator/load balancer or approved monitoring source as required; return minimal status |
| API `/openapi/v1.json` | Development only; verify unavailable in production |
| PostgreSQL | Private service network only; no public listener |
| Keycloak administration | Private management access only; public end-user OIDC endpoints limited to the required HTTPS routes |
| Prometheus, Jaeger, Elasticsearch and Kibana | Private observability network; no public unauthenticated dashboards or data endpoints |
| Docker/container management | Restricted operator access; never expose Docker daemon or orchestration control plane publicly |

Review all default ingress routes and wildcard host rules. An unrecognized host or path must not fall through to an administrative or internal service.

## 6. Network segmentation and egress

- Place API-to-database, API-to-identity, and monitoring-to-scrape traffic on private networks with least-privilege firewall/security-group rules.
- Do not publish PostgreSQL, Prometheus, Jaeger, Elasticsearch or Kibana ports to public interfaces.
- Restrict administrative interfaces and database management access to approved operator VPN/bastion or provider-native private access.
- Define outbound egress for OIDC, email/SMS, payment, document storage and other approved integrations. Deny unnecessary egress where the provider supports it.
- Separate production, staging and development networks, credentials, databases, identity realms/clients and observability data.
- Bind local development published ports to loopback; do not copy the local HTTP CSP exception or local credentials into production.

## 7. API and edge protection

- Apply request-size limits, connection/time-out limits and appropriate DDoS/WAF protections at the edge; align them with API-level request/rate limits.
- Calibrate API rate limits for real hospital traffic and expected replica count. Per-instance limits do not provide a shared global quota across multiple replicas.
- Preserve correlation IDs and trusted client-IP attribution without logging access tokens, credentials or patient-identifiable data.
- Confirm server-version disclosure is disabled in the Web Nginx response (`server_tokens off`) and the API Kestrel Server header remains disabled.
- Confirm secure response headers and strict CSP on the deployed Web response, not only on a local or CI container.
- Ensure the ingress does not cache authenticated or patient-specific API responses. Keep no-store semantics for patient search/detail and other sensitive data routes.

## 8. Acceptance tests

Capture dated, sanitized evidence for each deployment:

- [ ] DNS resolves each approved hostname to the intended ingress.
- [ ] Valid HTTPS certificate chain and hostname validation succeed; expiry monitoring is enabled.
- [ ] HTTP redirects to HTTPS; HSTS is present on trusted HTTPS responses and absent on untrusted forwarded schemes.
- [ ] Only configured proxy IPs are trusted; spoofed forwarded headers from an untrusted source do not change scheme/client IP.
- [ ] The production Web CSP contains only the approved API/OIDC origins and blocks inline script/style execution, objects and framing.
- [ ] API CORS accepts the approved Web origin and rejects an unapproved origin.
- [ ] Real OIDC discovery, login, callback, token exchange, silent renewal and logout work through the deployed ingress.
- [ ] Configure `Observability:Metrics:AllowedNetworks` to the actual Prometheus source CIDR(s); verify requests outside the allowlist return 404.
- [ ] `/metrics` is unreachable from a public client at the ingress and reachable from the approved Prometheus scrape source.
- [ ] Health endpoints are limited to approved probe/monitoring sources and disclose no dependency details beyond the intended status.
- [ ] Development-only OpenAPI is unavailable in production.
- [ ] PostgreSQL and observability services have no public listener.
- [ ] Unknown Host headers and unapproved paths are rejected.
- [ ] Edge and API rate limits behave as designed under expected concurrency.
- [ ] Authenticated patient/API responses are not cached by the ingress/CDN.
- [ ] Browser E2E and tenant-isolation suites pass against the deployed test topology.
- [ ] Hospital security/privacy and operational owners approve the evidence and residual risks.

A failed or untested item remains an open deployment gate. Do not mark production or hospital-pilot readiness based solely on CI or local Docker evidence.

## 9. Current implementation evidence and open gates

- Web PR #56/#57/#61/#62 implement runtime CSP origin validation, strict no-inline CSP and 20/20 browser E2E acceptance against the built Nginx container.
- Web PR #63 disables Nginx version disclosure and CI verifies the response does not contain the exact Nginx version.
- API PR #83 implements one-year HSTS on trusted HTTPS requests and explicit trusted-proxy configuration.
- API PR #84 validates production CORS as explicit HTTPS origins; API PR #85 restricts Development CORS to explicit loopback origins.
- CompileCare.Local binds its published local ports to loopback and explicitly opts into HTTP only for local development.
- Actual production domains, TLS provider/ingress, proxy IPs, public/private route rules, metrics access boundary, monitoring source, rate-limit sizing and real OIDC acceptance remain deployment-specific and are not yet evidenced.

## 10. Ownership and change control

The deployment owner must assign an accountable owner and evidence reference to every checklist item. Review this checklist whenever public hostnames, ingress, identity provider, proxy topology, monitoring topology, network segmentation or browser origins change. A change to API/Web route exposure must include a security review and a regression test for the public/internal boundary.
