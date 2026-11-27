# Backup, Restore and Disaster-Recovery Operational Baseline

**Status:** Required design and acceptance baseline. Not an executable provider-specific runbook until the hosting provider, database topology, storage locations, key-management owner, retention obligations and recovery objectives are approved.

## 1. Purpose and safety boundary

HeapCare stores sensitive hospital and patient workflow data. Backup and recovery must preserve confidentiality, integrity, tenant isolation, auditability and availability.

This document defines the controls and evidence required before a deployment can be considered operationally recoverable. It does not claim that backups, point-in-time recovery (PITR), cross-region recovery or tenant-level restoration are currently configured.

**Never run a restore over the live source database as an initial recovery step.** Restore into an isolated target, verify it, obtain the required operational approval, and only then execute a separately reviewed cutover plan. Do not use real patient data in local development or public test evidence.

## 2. Protected data and dependency inventory

Before selecting a backup design, the deployment owner must complete and approve this inventory.

| Data / dependency | Recovery requirement | Current evidence / open decision |
|---|---|---|
| PostgreSQL clinical and platform database | Consistent backups; transaction-log/WAL retention if PITR is required; integrity verification | PostgreSQL is the primary relational store. Provider-native backup/PITR is not yet evidenced. |
| Identity provider database and realm/client configuration | Recover identity records, realm configuration, signing/credential dependencies and redirect/client configuration | Local stack uses Keycloak. Production identity hosting and database topology are not selected here. |
| Uploaded documents and object storage | Versioned or otherwise recoverable content; preserve metadata-to-object consistency | Production document/object-storage design and retention are not yet evidenced. |
| Encryption and signing keys | Recoverability without placing keys beside the backups they protect | KMS/HSM/provider key custody, rotation and break-glass ownership are open. |
| Deployment configuration and secrets | Reproducible, versioned configuration; secrets recoverable from an approved secret manager | Production hosting and secret manager are not selected. Never put secret values in this repository. |
| Audit and security logs | Retention, tamper resistance, access and recovery requirements | Production log destination, immutability and retention are open. |
| Search and observability indexes | Determine whether rebuildable from source or require backup | Elasticsearch/Kibana are currently part of the local observability stack; production topology is open. |

For every additional data store, queue, cache, integration or file store introduced later, update this inventory and its recovery dependency order.

## 3. Recovery objectives — approval required

Do not infer service-level objectives from local performance tests or development defaults. Product, hospital operations, security/privacy and hosting owners must approve values per deployment tier.

| Objective | Required decision | Approval / evidence |
|---|---|---|
| Recovery Point Objective (RPO) | Maximum acceptable data loss for the database and each external data store | Pending hospital/client and operational risk approval |
| Recovery Time Objective (RTO) | Maximum time to restore service, including identity, database, secrets, ingress and validation | Pending hospital/client and operational risk approval |
| Backup frequency | Full/base backup cadence and transaction-log/WAL archive cadence | Pending provider capability and approved RPO |
| Retention | Daily/weekly/monthly retention, legal hold, deletion and expiry rules | Pending contractual, privacy and legal review |
| Geographic resilience | Same-region, cross-zone and/or cross-region requirements | Pending hosting and data-residency review |
| Restore-test cadence | Frequency and evidence retention for restore drills | Recommended starting point: at least monthly, subject to owner approval |
| Tenant recovery | Whether recovery is whole-database only or requires tenant-level export/recovery | Not implemented or evidenced; requires a separately designed, tested workflow |

Record the approved values, owner, approval date and review date in the deployment-specific operations record before production onboarding.

## 4. Minimum control requirements

### 4.1 Backup creation and storage

- Use the selected PostgreSQL provider's supported backup and WAL/PITR mechanism; document the exact product, configuration and recovery limitations.
- Encrypt backups in transit and at rest. Record the algorithm and key-management boundary in the deployment security record.
- Keep backup storage separate from the primary database failure and administrative boundary where the provider supports it.
- Restrict backup creation, listing, download, deletion and restore permissions by least privilege. Separate routine operator access from destructive retention/key administration.
- Enable immutable retention, object lock or equivalent deletion protection where required and supported by the approved provider.
- Monitor backup completion, WAL/archive continuity, storage capacity, retention expiry and restore-point availability. Alert on missing, failed or stale backups.
- Never log connection strings, access tokens, encryption keys, patient data or backup contents.
- Verify that backup jobs do not overload the production database or degrade clinical workflows.

### 4.2 Key and secret custody

- Store encryption keys in the approved KMS/HSM or equivalent managed key system, not in the database backup bucket or source repository.
- Define key administrators, recovery/break-glass roles, rotation, revocation, audit and emergency access.
- Test that an authorised recovery operator can retrieve/decrypt a backup without granting broad permanent access.
- Ensure key loss, key revocation and key rotation scenarios are included in the disaster-recovery exercise.
- Keep application secrets in the approved secret manager; rotate credentials after suspected exposure and after recovery if the incident requires it.

### 4.3 Restore integrity and tenant safety

- Restore to a new isolated target with network access restricted to the recovery team and validation services.
- Verify PostgreSQL version/extension compatibility, migration history, schema, referential integrity and expected tenant boundaries.
- Confirm that the restored target cannot accidentally send real notifications, messages, payments or external integrations. Disable outbound side effects during validation.
- Validate representative synthetic workflows and record counts without exposing patient data in evidence.
- Validate audit continuity and identify the last durable transaction boundary.
- Do not claim tenant-level point-in-time restore: a shared multi-tenant database restore affects the database as a whole unless a separate tenant extraction/reconciliation design is implemented and tested.
- Preserve the original source and its logs until the incident owner approves disposition.

## 5. Recovery procedure outline

This sequence is a controlled checklist, not a provider command script. Populate the provider-specific console/CLI steps and account roles only after the platform is selected.

1. **Declare and scope:** incident commander records the incident, affected deployment, data stores, suspected compromise/outage window, requested recovery point and decision authority.
2. **Contain:** prevent further writes or destructive automation where safe; preserve logs and evidence; isolate compromised credentials and systems without destroying forensic evidence.
3. **Select recovery point:** identify the last verified backup and, where configured, a continuous WAL/PITR point. Record the expected RPO gap and obtain approval for the selected timestamp.
4. **Prepare isolated target:** provision a clean recovery environment with approved network boundaries, database version/extensions, secret access and identity dependencies.
5. **Restore:** use the provider's documented supported procedure to restore the backup and transaction logs to the isolated target. Capture operation IDs and timestamps.
6. **Validate:** run database integrity checks, migration-history checks, tenant-isolation tests, synthetic application smoke tests and audit/log continuity checks. Confirm the identity provider and external storage dependencies are usable.
7. **Assess data consistency:** reconcile database references with document/object storage, identity, payment/notification integrations and any asynchronous work. Record any unrecoverable gap or manual reconciliation requirement.
8. **Approve cutover:** incident commander, database owner and security/privacy owner approve the target, recovery point, RPO impact and cutover plan.
9. **Cut over:** apply the separately reviewed DNS/ingress/connection-secret changes; verify health, authentication, authorization, tenant isolation and clinical workflow smoke tests.
10. **Monitor and close:** monitor error rates, latency, queue/backlog, security alerts and data integrity; document RTO achieved, RPO achieved, decisions, exceptions and follow-up actions.
11. **Preserve and dispose:** retain the original environment and evidence under incident/legal hold rules. Dispose of temporary recovery copies only after explicit approval and verified secure deletion.

## 6. Restore-drill acceptance checklist

Every drill must produce a dated, access-controlled evidence record containing no patient-identifiable information or secrets.

- [ ] Approved scenario, scope, recovery point and drill owner.
- [ ] Backup completion and retention evidence.
- [ ] Key retrieval/decryption test performed by an authorised operator.
- [ ] Restore into an isolated target completed; source was not overwritten.
- [ ] PostgreSQL integrity, schema and migration history validated.
- [ ] Tenant isolation and authorization regression suite passed.
- [ ] Authentication/identity provider recovery validated.
- [ ] Object/document storage consistency validated where applicable.
- [ ] Synthetic patient registration, appointment, OPD and billing smoke workflows passed.
- [ ] Audit and security event continuity checked.
- [ ] External side effects remained disabled during validation and were re-enabled only with approval.
- [ ] Measured RPO and RTO recorded against approved objectives.
- [ ] Exceptions, data gaps, corrective actions, owners and due dates recorded.
- [ ] Incident/recovery contacts and escalation path verified.
- [ ] Recovery environment and temporary data disposed of under approved policy.

A drill is a failure if the target is not isolated, the backup cannot be decrypted, tenant isolation fails, material data dependencies are missing, or the measured RPO/RTO exceeds approved objectives. Record partial success as a failure with remediation, not as a pass.

## 7. Required deployment-specific records

Maintain these records in the approved restricted operations system, not in public source control:

- Hosting provider, region, account/project and PostgreSQL service/version.
- Database topology, replicas, backup/PITR settings and WAL/archive configuration.
- Backup storage location, immutability, encryption and retention policy.
- KMS/HSM key identifiers and named key custodians (never key material).
- Secret-manager location and emergency access procedure (never secret values).
- Approved RPO/RTO and data-retention schedule with sign-offs.
- Restore runbook version, operator roles and provider-specific command references.
- Latest successful backup and restore-drill evidence references.
- Incident contacts, escalation path and cutover authority.

## 8. Current HeapCare status and release gates

- PostgreSQL is the selected primary relational database.
- The local Docker stack and CI migration rehearsal validate application migrations and local integration behavior; they do **not** prove managed production backups, PITR, key recovery or disaster recovery.
- The production hosting provider, database service, backup configuration, retention, KMS ownership, RPO/RTO and restore cadence are not established by the repository evidence available for this baseline.
- Tenant-level recovery/export is not implemented or certified.
- Production/pilot readiness remains blocked until the deployment-specific controls are configured, evidenced by a successful isolated restore drill, and approved by the hospital/client and operational owners.

## 9. Change control

Any change to database topology, tenant isolation, document storage, identity provider, key management, backup retention or recovery process requires this baseline and the deployment-specific runbook to be reviewed. Add a testable recovery acceptance criterion to the feature's Definition of Done whenever a change introduces a new durable data store.


### Local CI logical backup/restore smoke evidence — 2026-11-27

HeapCare.Local PR #34 merged the isolated PostgreSQL logical backup/restore step. The merged-main run [#46041076667](https://github.com/HeapCare/HeapCare.Local/actions/runs/46041076667) passed the nine-service stack, real-Keycloak persisted-role API acceptance, logical backup/restore step, real-OIDC browser acceptance and cleanup.

The CI step:
- Creates a custom-format logical archive from the disposable local PostgreSQL CI database with `pg_dump`.
- Restores the archive into a separate empty database with `pg_restore`; it does not overwrite the source.
- Compares every application table's row count between source and restored databases.
- Removes the temporary archive and restore database on success or failure.

**Evidence boundary:** this is a repeatable logical dump/restore smoke test using one ephemeral local PostgreSQL service. It is not a provider-native backup, physical/base backup, continuous WAL archive, PITR, encryption/key-custody test, off-site or cross-region recovery, tenant-level restore, or production disaster-recovery drill. No production RPO/RTO is measured or approved by this CI run. All provider, storage, encryption, retention, geographic and recovery objectives in this baseline remain open until configured and independently exercised.
