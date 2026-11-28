# HeapCare public-site deployment baseline

**Status:** Container build and CI smoke tests are implemented. Hosting provider, production service, custom domain, DNS authority and live deployment are not yet verified.

## Application

- Repository: `HeapCare/HeapCare.Site`
- Runtime: .NET 8 / ASP.NET Core / Blazor Server
- Container: multi-stage Docker build using the .NET 8 SDK and ASP.NET Core runtime
- Runtime user: non-root `app`
- Container port: `8080`
- Health probe: `GET /health`
- Custom domain: **Not selected/verified**

The public information website is separate from the authenticated hospital workspace in `HeapCare.Web`. It must not receive or process patient records, hospital-user credentials or clinical data.

## Hosting requirements

The selected hosting provider must support:

1. Docker image deployment from the `HeapCare.Site` repository root.
2. HTTP traffic to container port 8080 and HTTPS termination at a trusted ingress.
3. WebSocket upgrade and persistent connections for the Blazor Server SignalR circuit.
4. Health checks against `/health`.
5. A custom domain only after the organization confirms ownership and DNS authority.
6. Managed TLS issuance/renewal and correct forwarded-host/proxy behavior.
7. A documented rollback to the previous image digest.
8. Build/deployment logs and operator-visible deployment status.

The available repository evidence does not establish a production hosting provider, service/project, region, billing tier, domain purchase, DNS authority, TLS certificate or deployment connection. Do not configure or claim a live domain from a candidate name or a repository example.

## Container build and run

From the `HeapCare.Site` repository root:

```bash
dotnet restore src/HeapCare.Site/HeapCare.Site.csproj
dotnet run --project src/HeapCare.Site/HeapCare.Site.csproj

docker build -t heapcare-site .
docker run --rm -p 8080:8080 heapcare-site
```

Verify:

- `GET http://localhost:8080/health` returns HTTP 200.
- `/`, `/features`, `/technology`, `/about` and `/contact` render.
- Blazor Server WebSocket negotiation succeeds and the live circuit remains stable.
- Static logos/fonts/styles load successfully.

The GitHub Actions workflow restores and builds the .NET 8 application and builds the deployment container on pull requests. The image is provider-neutral; no provider-specific deploy command is committed.

## Public-site release checklist

- [ ] Confirm the hosting provider and service/project with the owner.
- [ ] Confirm the hosting region, billing tier and deployment identity.
- [ ] Confirm the purchased custom domain and authoritative DNS account.
- [ ] Configure deployment from a reviewed HeapCare.Site main commit/image digest.
- [ ] Configure HTTPS/TLS and the approved domain only after ownership is verified.
- [ ] Verify WebSocket support and Blazor circuit stability through the real ingress.
- [ ] Verify `/health` and all public routes from the internet.
- [ ] Verify technology logo/CDN assets, fonts and static resources load under the production Content Security Policy.
- [ ] Verify mobile and desktop layouts on the deployed site.
- [ ] Verify no secrets, credentials, patient data or authenticated application routes are exposed.
- [ ] Record deployment ID, image digest, domain verification time, TLS expiry and rollback target.

## Content and availability boundary

The public site labels implemented foundations separately from planned/deferred modules. A successful public website deployment is not evidence that the hospital platform itself is production-ready or that every catalogue capability is operational.

Do not publish pricing, customer references, support SLAs, regulatory certifications or general-availability claims without approved evidence. The official HeapCare support contact must not be invented or inferred from a domain candidate.

## Additional verification evidence

Site CI builds the .NET 8 website and the deployment container. The runtime smoke starts the image on port 8080 and checks `/health` plus the public routes. The technology page uses descriptive alternative text for its logo images. These are container/application checks; they do not establish public DNS, TLS, CDN, live hosting or custom-domain readiness.
