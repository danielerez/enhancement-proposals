# Cross-cluster authentication and TLS trust for tenant cluster workloads

| Field | Value |
|---|---|
| Author(s) | Daniel Erez |
| Jira | [OSAC-1644](https://redhat.atlassian.net/browse/OSAC-1644) |
| Date | 2026-09-07 |

## Problem Statement

In a multi-cluster OSAC deployment, tenant-cluster workloads cannot reliably authenticate to the fulfillment service using credentials issued by their own Kubernetes cluster. The resulting failure blocks the intended management-cluster and tenant-cluster topology. Tenant-cluster workloads also cannot establish a trusted TLS connection until they trust the management cluster's certificate authority. Without this capability, consumers such as the CSI driver cannot securely make required fulfillment-service calls across cluster boundaries.

## In Scope

- Consumption of a tenant-scoped Keycloak service-account client by tenant-cluster workloads for cross-cluster fulfillment-service access. Multiple clusters belonging to the same tenant may reuse that tenant's client; a client must never be shared across tenants. [Clarify: R1.Q1] [Clarify: R1.Q2]
- Secure connectivity for supported tenant-cluster consumers, including the CSI driver and future tenant-cluster workloads: server-authenticated TLS with hostname validation and a scoped trust anchor, followed by the OAuth2 `client_credentials` flow. Fulfillment service rejects tokens with a missing or mismatched issuer, audience, required scope, or tenant `organization` claim. [Clarify: R1.Q2] [Clarify: R1.Q3]
- The `ClusterOrder` post-install provisioning workflow is the sole owner of installing cert-manager, waiting for it to become ready, and injecting the management-cluster CA into the tenant-cluster trust store. [Clarify: R1.Q3] [Clarify: R1.Q4]
- A tenant-scoped credential handoff for supported tenant-cluster workloads, providing secure access to the owning tenant's Keycloak client credentials and issuer URL. The storage and delivery mechanism is owned by OSAC-4197. [Clarify: R1.Q2]
- Verified end-to-end authenticated gRPC connectivity from supported tenant-cluster consumers to fulfillment service. Credentials are transmitted only after successful TLS peer verification. [Clarify: R1.Q3]
- Trust setup participates in the existing `ClusterOrder` provisioning and health status model, including retry and post-provisioning recovery reporting. [Clarify: R1.Q5]

## Out of Scope

- CSI-driver deployment, StorageClass configuration, and vendor-specific volume-provisioning behavior, which are addressed by related follow-up work. [Clarify: R1.Q3]
- Per-cluster Keycloak identities for tenant-cluster workloads. A per-cluster identity may be added as future isolation hardening, but it is not required by this Feature. [Clarify: R1.Q1]
- `osac-operator` authentication changes, including replacing its current credential and introducing per-hub credentials. These are follow-up work; the credential/API contract introduced by this Feature must remain extensible for future per-hub identities. [Clarify: R1.Q2] [Clarify: R1.Q3]

## User Stories

### Cloud Infrastructure Admin

- As a Cloud Infrastructure Admin, I want tenant-cluster workloads on separate clusters to use a trusted tenant-scoped service identity when communicating with fulfillment service so that the supported multi-cluster deployment topology works securely.

- As a Cloud Infrastructure Admin, I want trust established automatically during tenant-cluster provisioning, with retry and failure status when setup cannot complete, so that supported workloads can connect securely without manual trust-store setup or repair. [Clarify: R1.Q4] [Clarify: R1.Q5]

### Tenant Admin / Tenant User

- As a Tenant Admin or Tenant User, I want workloads on my tenant cluster to communicate securely with fulfillment service so that tenant-cluster services such as storage can use the capabilities assigned to the tenant.

## Assumptions

- This PRD does not prescribe an upgrade, migration, or backfill path for existing tenant clusters. If a transition is needed, it is owned by OSAC-4197; credential rotation and lifecycle are owned by OSAC-5179. [Clarify: R1.Q4]
- To be determined — whether dedicated CLI, UI, or additional troubleshooting indicators beyond the existing `ClusterOrder` status are in scope for this Feature. [Clarify: R1.Q5]

## Dependencies

- **Keycloak:** Provides tenant-scoped service-account clients and the access tokens needed for tenant-cluster workload authentication.
- **Certificate-management capability on tenant clusters:** Provides the cert-manager/trust-distribution capability and interfaces consumed by the `ClusterOrder` post-install provisioning workflow; it does not own the post-install workflow or management-cluster CA injection.
- **OSAC-3291:** Deploys the tenant-cluster CSI driver and consumes the tenant-scoped credential handoff and trust/authentication model established by this Feature. [Clarify: R1.Q3]
- **OSAC-4197:** Owns creation, tenant isolation, delivery, and any transition from the shared CSI identity to tenant-scoped Keycloak clients. [Clarify: R1.Q1] [Clarify: R1.Q2] [Clarify: R1.Q4]
- **OSAC-5179:** Owns credential rotation, revocation, and lifecycle management after tenant-scoped credentials are established. [Clarify: R1.Q2] [Clarify: R1.Q4]

---

## Provenance

Authored: draft @ prd 0.9.0 - 562b610, workspace main @ ad9ec2979
Final: revise @ prd 0.9.0 - 562b610, workspace prd/OSAC-1644 @ ad9ec2979

> Context changed between draft and revise.

<!-- ai-workflow-provenance:{"schema_version":1,"provenance_kind":"session","workflow":"prd","workflow_version":"0.9.0","ai_workflows":"562b610","source_repo":"ad9ec2979","source_repo_branch":"prd/OSAC-1644","commits_behind_main":0,"commits_ahead_main":0,"main_ref":"main","phases":["draft","revise","revise","revise","revise","revise"],"authoring_modes":["skill"],"context_changed":true,"origin_untracked":false} -->
