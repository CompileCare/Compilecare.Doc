# OIDC signing-key rotation runbook

**Status:** Provider-neutral operational procedure; production execution is not yet evidenced.  
**Applies to:** CompileCare API JWT validation against the configured OIDC issuer.  
**Current application configuration:** Authentication__Authority, Authentication__Audience and Authentication__RequireHttpsMetadata. Production requires HTTPS metadata.

This runbook does not select or configure a production identity provider. The production provider, issuer, token lifetimes, metadata/JWKS cache behavior, operational owners and change window must be recorded before this procedure is used. Do not treat the local Keycloak realm or local development keys as production configuration.

## Security principles

- The identity provider owns private signing keys. CompileCare API instances consume public signing keys through the issuer's OIDC discovery/JWKS metadata; do not copy private signing keys into the API, browser, repository, CI variables or deployment manifests.
- Use a planned overlap rotation: publish the new public key before activating it for signing, keep the previous public key available while unexpired tokens signed with it may still be presented, then retire the previous key.
- Never remove or disable the only valid key before confirming that all API instances can validate tokens signed by the replacement key.
- If compromise is suspected, prioritize containment over compatibility. Do not re-enable a compromised key merely to restore availability.
- Key rotation is distinct from client-secret rotation, user password resets, TLS certificate renewal and database credential rotation; track those separately.

## Required inventory before a planned rotation

- [ ] Production OIDC provider, tenant/realm, issuer URL and provider version are recorded.
- [ ] API audience and accepted signing algorithms are confirmed from the deployed configuration and provider metadata.
- [ ] Maximum access-token lifetime, allowed clock skew and refresh-token/session policy are recorded.
- [ ] API metadata/JWKS refresh and caching behavior is confirmed for the exact deployed .NET/runtime configuration; do not assume that every replica refreshes immediately.
- [ ] Number of API replicas, ingress topology, deployment regions and monitoring owners are recorded.
- [ ] Current active and passive signing-key identifiers (kid), activation state and public JWKS publication are recorded. Never export private key material into the evidence record.
- [ ] A change owner, approver, maintenance window, rollback owner and incident contact are assigned.
- [ ] A staging rehearsal has validated old-key/new-key overlap and metadata refresh using non-production keys.

## Planned rotation procedure

1. **Open and approve the change.** Record the issuer, target key identifier, expected overlap window, token lifetime, API replicas and rollback decision-maker. Confirm monitoring and on-call coverage.
2. **Create the replacement key at the identity provider.** Generate it using the provider's supported key-management workflow and approved algorithm/strength. Keep it passive/inactive for signing initially. Private material remains inside the provider's protected key store/HSM where supported.
3. **Publish the replacement public key.** Confirm the issuer's JWKS endpoint returns the new key with a unique kid and correct public parameters. Confirm OIDC discovery still advertises the expected issuer and JWKS URI. Do not change CompileCare's issuer or audience to work around key publication.
4. **Activate the replacement for signing.** Mint a fresh access token and confirm its header references the expected new kid. Do not paste the token into tickets, chat, logs or screenshots; inspect it only in an approved secure test environment and retain only non-secret metadata such as kid and timestamps.
5. **Verify API acceptance on every serving instance.** Exercise an authenticated, non-sensitive health/context request with the new-key token. Confirm all API replicas accept it after metadata refresh. Also confirm a still-valid token signed by the previous key remains accepted during the overlap window. Record HTTP result, replica/build identity and timestamp without retaining bearer tokens.
6. **Observe the overlap.** Monitor authentication failures, 401 rates, identity-provider signing errors, metadata/JWKS fetch failures and clock synchronization. Keep the previous public key published for at least the maximum lifetime of tokens signed by it plus configured clock skew and the measured metadata propagation/cache interval. Use the longest applicable value and document the calculation.
7. **Retire the previous key.** Only after the overlap condition is met and no valid tokens should depend on it, disable/remove the old key using the provider's supported process. Verify newly minted tokens continue to use the replacement kid and API validation remains healthy.
8. **Close with evidence.** Record old/new kid values, timestamps, provider audit/change reference, metadata verification, per-replica validation results, monitoring window and approver. Never include private keys or bearer tokens.

## Emergency rotation after suspected compromise

1. Declare the security incident and involve the identity-provider owner, CompileCare security owner and incident lead.
2. Disable/revoke the compromised signing key at the provider immediately and ensure it is no longer published as trusted signing material. Do not preserve a compatibility window for a compromised key.
3. Activate a clean replacement key and verify its public JWKS publication and API acceptance across every serving replica.
4. Revoke affected identity-provider sessions/refresh tokens as supported, investigate token issuance and access logs, and assess whether client credentials or other secrets were also exposed.
5. Expect access tokens signed by the compromised key to fail validation. Communicate the re-authentication impact and provide a controlled recovery path.
6. Preserve provider audit evidence and incident timelines without exporting tokens, private key material or patient data.
7. Complete incident review, determine scope, rotate any additionally exposed secrets and update the threat model and access review.

## Rollback and failure handling

- If the new key is unavailable or API replicas cannot validate it, pause retirement of the previous key. If the new key is not compromised, restore the provider's previous active signing configuration and verify the prior key through JWKS and all API replicas.
- Do not roll back by disabling JWT signature, issuer, audience, lifetime or HTTPS metadata validation; do not add static production signing keys to the application.
- If the previous key is compromised, do not restore it. Continue with emergency containment and re-authentication even if this causes temporary sign-in disruption.
- If replicas disagree, stop the change, identify stale metadata/cache or deployment differences, and resolve them before retiring any key.
- Treat persistent 401s, discovery/JWKS failures, unknown kid errors or issuer mismatch as a failed rotation until root cause is understood.

## Verification matrix

| Case | Expected result | Evidence to retain |
|---|---|---|
| Fresh token signed by replacement key | Accepted by all API replicas | New kid, timestamp, replica/build and non-sensitive response status |
| Still-valid token signed by previous key during planned overlap | Accepted until its expiry/overlap condition | Old kid and expiry timestamp only; never retain token |
| Token with unknown or retired kid | Rejected | Test case and 401 outcome |
| Wrong issuer or audience | Rejected | Test case and 401 outcome |
| Expired token | Rejected | Test case and 401 outcome |
| Discovery/JWKS endpoint unavailable | Alert raised; no signature-validation bypass | Alert and recovery record |

**Important test boundary:** Existing CompileCare API integration tests use synthetic local signing configuration. They prove authorization behavior under that test setup; they do not prove production-provider key rollover, JWKS cache refresh or multi-replica behavior. A staging rehearsal against the selected provider and deployed runtime is required.

## Rotation record

| Field | Value to record |
|---|---|
| Environment / provider / realm |  |
| OIDC issuer / API audience |  |
| Change or incident reference |  |
| Old kid / new kid |  |
| Maximum access-token lifetime / clock skew |  |
| Metadata propagation/cache interval |  |
| Planned start / activation / retirement timestamps |  |
| API replicas and validation result |  |
| Monitoring outcome / alerts |  |
| Rollback or incident decision |  |
| Owner / approver / completion date |  |

## Production activation gate

Do not mark signing-key rotation production-ready until the production identity provider is selected, a staging rehearsal has exercised key overlap and JWKS refresh across all API replicas, emergency compromise handling is approved, evidence is retained, and the operational owner accepts the procedure.