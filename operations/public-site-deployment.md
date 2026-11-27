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
