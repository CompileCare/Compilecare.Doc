# CompileCare public website deployment baseline

**Status:** Container build implemented and CI-verified. Hosting provider, custom-domain deployment and live-site verification are not yet confirmed.

## Application

- Repository: CompileCare/CompileCare.Site
- Runtime: .NET 8 / ASP.NET Core / Blazor Server
- Container: multi-stage Docker build using the .NET 8 SDK and ASP.NET Core runtime images
- Runtime user: non-root app
- Container port: 8080
- Health probe: GET /health
- Custom domain requested: compilecare.site

The public information website is separate from the authenticated hospital workspace in CompileCare.Web. It must not receive or process patient records, hospital user credentials or clinical data.

## Hosting requirements

The selected hosting provider must support:

1. Docker image deployment from the repository root.
2. HTTP traffic to container port 8080 and HTTPS termination at the trusted ingress.
3. WebSocket upgrade/persistence for the Blazor Server SignalR connection.
4. Health checking against /health.
5. Custom-domain mapping for compilecare.site and, if required, www.compilecare.site.
6. Managed TLS certificate issuance/renewal and correct forwarded-host/proxy behavior.
7. A documented rollback to the previous image digest.
8. Build/deployment logs and an operator-visible deployment status.

Do not select a provider or change DNS records until the owner confirms the existing hosting account, project/service, region, billing tier and DNS authority. Do not assume that a successful GitHub build deploys the domain.

## Container build and run

From the repository root:

- docker build -t compilecare-site .
- docker run --rm -p 8080:8080 compilecare-site
- Verify http://localhost:8080/health returns HTTP 200.
- Verify /, /features, /technology, /about and /contact render.
- Verify Blazor Server WebSocket negotiation and live circuit connectivity.

The GitHub Actions workflow builds the .NET 8 application and the Docker image on pull requests. The container image is provider-neutral; no provider-specific deploy command is currently committed.

## Public-site release checklist

- [ ] Confirm the hosting provider and service/project.
- [ ] Confirm the active custom-domain DNS records and who controls them.
- [ ] Confirm whether www.compilecare.site should redirect to the apex domain.
- [ ] Configure the service to deploy the reviewed image digest from main.
- [ ] Configure HTTPS/TLS and custom domain.
- [ ] Verify WebSocket support and Blazor circuit stability through the ingress.
- [ ] Verify /health and all public routes from the internet.
- [ ] Verify technology logo/CDN assets, fonts and Bootstrap/MudBlazor static assets load under the production Content Security Policy.
- [ ] Verify mobile and desktop layouts on the live domain.
- [ ] Verify no secrets, credentials, patient data or authenticated application routes are exposed by the public site.
- [ ] Record deployment ID, image digest, domain check time, TLS expiry and rollback target.

## Content and availability boundary

The public site labels current implementation foundations separately from planned/deferred modules. A successful public website deployment is not evidence that the hospital platform itself is production-ready or that the listed planned capabilities are operational.

Do not publish pricing, customer references, support SLAs, regulatory certifications or general-availability claims without approved evidence.


## Additional public-site technology and container runtime verification — 2026-11-27

- Site PR #6 and #7 expanded the technology inventory with exact shared-package versions, OpenTelemetry instrumentation, HTML5/Razor, CSS3, JavaScript, npm, Git, Chromium, Google Fonts (DM Sans/Manrope), jsDelivr, Simple Icons, Bash, PowerShell and Windows Batch. The technology page contains 66 logo images and each image has a non-empty alt attribute.
- Site PR #8 added a real container smoke test: CI starts the .NET 8 image on port 8080, verifies GET /health and HTTP 200 for /, /features, /technology, /about and /contact, captures logs on failure and always removes the container.
- Site PR #10 completed exact .NET security, migration and observability package version details. The public-site build, Docker build, runtime startup, route smoke tests and cleanup passed.
- The public site remains an informational website; it is not connected to patient data or the authenticated hospital workspace. The /demo page does not present a fake login or fabricated operational records.

**Live deployment remains unverified.** Site issue #9 tracks deployment to compilecare.site, but provider/service, DNS authority and deployment connection have not been confirmed. Do not claim the custom domain is live until provider deployment, TLS, WebSocket, route and CDN/CSP checks are executed against the actual domain.
