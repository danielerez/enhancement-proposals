# Clarification Log — OSAC-1644

## Status

- Rounds completed: 1
- Open gaps: 1
- Exit criteria met: No

## Round 1 — scope, ownership, lifecycle, and interfaces

### R1.Q1: Identity scope

Should tenant-cluster workloads use one Keycloak service-account client per tenant, reused across that tenant's clusters, or a distinct client per cluster?

#### Answer

Tenant-cluster workloads require a tenant-scoped Keycloak service-account client carrying that tenant's organization claim. Multiple clusters belonging to the same tenant may reuse the client, but clusters belonging to different tenants must never share a client. Per-cluster clients are a possible future isolation improvement and are not required by this Feature. Workload authentication uses an OAuth2 access token obtained through the `client_credentials` flow over server-authenticated TLS; this Feature does not require mutual TLS.

#### Impact

The PRD must state that tenant-cluster credentials use the owning tenant's identity and may be reused by that tenant's clusters. Fulfillment service must validate the configured issuer, audience, required scope, and tenant `organization` claim, and reject a token when any value is missing or does not match the expected tenant or service. A tenant-cluster credential must not grant access to another tenant.

#### Decision (D1)

Use a tenant-scoped Keycloak service-account client for tenant-cluster workloads. The same client may be used by multiple clusters owned by that tenant, while cross-tenant reuse is prohibited. Per-cluster identities remain optional future hardening. The access token is sent to fulfillment service only after the workload has successfully validated the fulfillment-service hostname against a scoped trust anchor.

#### Shared-client transition boundary

OSAC-4197 owns any transition from the shared CSI client to tenant-scoped identities. Its implementation may provide the owning tenant's credentials to each tenant cluster, and tenant-cluster workloads adopt that identity as part of the rollout. OSAC-1644 does not mandate a migration, backfill, rollback, or deployment-wide cutover sequence; those lifecycle details belong to OSAC-4197. The transition must preserve the requirement that clients are never shared across tenants.

---

### R1.Q2: Ownership of client lifecycle

Does OSAC-1644 create the tenant-scoped Keycloak clients and credentials, or does OSAC-4197 own their lifecycle while OSAC-1644 defines how consumers use credentials?

#### Answer

OSAC-1644 establishes server-authenticated TLS trust and Keycloak access-token consumption for supported tenant-cluster consumers. OSAC-4197 owns automated creation, tenant isolation, delivery, and any transition from the shared CSI client for tenant-scoped Keycloak clients. OSAC-5179 owns credential rotation, revocation, and lifecycle management. The current consumers are the CSI driver and future tenant-cluster workloads. `osac-operator` authentication changes, including per-hub credentials, are out of scope for this Feature; the credential/API contract must remain extensible for that future use case.

#### Impact

The PRD must list OSAC-4197 as a dependency for tenant-scoped identity creation and delivery, and OSAC-5179 for credential lifecycle, while retaining OSAC-1644 responsibility for supported tenant-cluster consumers' secure connectivity and credential-consumption contract. The contract must not require a cluster-specific identity so that future per-hub operator credentials remain possible.

#### Decision (D2)

OSAC-4197 owns tenant-scoped Keycloak-client creation, tenant isolation, delivery, and transition; OSAC-5179 owns credential lifecycle; OSAC-1644 owns secure connectivity and credential consumption for supported tenant-cluster consumers. Operator authentication and per-hub operator credentials are not part of this Feature.

#### Credential handoff contract

OSAC-4197 provisions credential delivery for each tenant cluster during `ClusterOrder` provisioning. Where a Kubernetes Secret is used, it contains the owning tenant's Keycloak `client_id`, `client_secret`, and `issuer_url`; the same tenant credentials may be delivered to multiple clusters belonging to that tenant, but never to a cluster belonging to another tenant. The expected audience and required scope are platform configuration. The `issuer_url` must match an approved HTTPS Keycloak issuer and host configured by the platform; a tenant-provided or otherwise unapproved issuer is rejected. Only the intended consumer service accounts may read the Secret through tenant-scoped RBAC, and a consumer must never read credentials belonging to another tenant. The handoff contract identifies the credential and tenant scope without requiring a cluster-specific client, leaving room for future per-hub operator identities.

Credential rotation, revocation, and lifecycle management are owned by OSAC-5179 and are not required by this Feature. The initial token request must validate the issuer hostname and trusted CA and must not follow redirects; in particular, the `client_secret` must never be forwarded to another host. If the credential delivery is missing or invalid, or if the issuer or token endpoint cannot be validated, the consumer must not send a fulfillment request; it reports the appropriate provisioning or health failure and retries. Consumers must not use credentials belonging to another tenant; shared-client transition behavior is owned by OSAC-4197.

---

### R1.Q3: CSI delivery boundary

Does OSAC-1644 need to deliver CSI connectivity end-to-end, or establish the trust and credential prerequisites that OSAC-3291 and other workload-cluster consumers use?

#### Answer

OSAC-1644 establishes cross-cluster trust and authentication for the CSI driver and future tenant-cluster workloads. Success is a supported tenant-cluster consumer using its owning tenant's Keycloak credentials to obtain an access token through the OAuth2 `client_credentials` flow and establish an authenticated gRPC connection to fulfillment service. The client validates the fulfillment-service hostname and scoped trust anchor before transmitting the token. CSI-driver deployment and vendor-specific volume provisioning are follow-up responsibilities. `osac-operator` authentication changes are out of scope.

#### Impact

The PRD must require cert-manager availability, management-CA injection during `ClusterOrder` post-install, delivery of the owning tenant's credentials, and a verified secure workload-to-fulfillment-service connection without assigning CSI deployment, vendor-storage functionality, or operator authentication changes to this Feature.

#### Decision (D3)

Success requires the CSI driver or another supported tenant-cluster workload to use its owning tenant's Keycloak credentials over validated server-authenticated TLS to complete an authenticated gRPC call to fulfillment service. The CSI driver remains a dependent validation consumer, not a delivered workload; `osac-operator` is not a current validation consumer.

---

### R1.Q4: Existing clusters and rotation

Does OSAC-1644 require an upgrade, migration, or backfill path for existing tenant clusters, and who owns credential rotation or revocation?

#### Answer

OSAC-1644 requires tenant-cluster workloads to receive and use their owning tenant's credentials as part of the supported provisioning flow. It does not mandate an upgrade, migration, or backfill path for existing tenant clusters. OSAC-4197 owns creation, tenant isolation, delivery, and transition of tenant-scoped Keycloak credentials; OSAC-5179 owns their rotation, revocation, and lifecycle management.

#### Impact

The PRD must cover cert-manager availability, management-CA injection, automatic trust setup, and delivery of the owning tenant's credentials during the supported tenant-cluster provisioning flow. It must identify OSAC-4197 as the dependency for identity creation/delivery and OSAC-5179 for credential lifecycle, without prescribing an upgrade or backfill implementation in this Feature.

#### Decision (D4)

OSAC-1644 automatically establishes trust and enables tenant-cluster workloads to consume their owning tenant's credentials. OSAC-4197 owns identity creation, delivery, and any required transition for existing clusters; OSAC-5179 owns credential lifecycle. An upgrade, migration, or backfill implementation is not a requirement of this PRD.

---

### R1.Q5: Interfaces and status

Is this administrator-managed installation/provisioning behavior only, or must dedicated UI, CLI, and troubleshooting support be delivered beyond the existing Kubernetes status model?

#### Answer

A trust-specific status signal is needed during provisioning and recovery, but a new trust-only condition is not required by this Feature. The owning resource is `ClusterOrder`, using the existing status model defined by OSAC-1604: initial trust setup is part of the provisioning readiness gate, and later trust loss is reported as an independent health problem.

#### Impact

The PRD must define the existing `ClusterOrder` status behavior. During initial provisioning, `PROGRESSING=True` uses reasons such as `TrustSetupPending` or `TrustSetupRetrying`; a terminal trust failure sets `FAILED=True` with reason `TrustSetupFailed`, while successful trust setup allows the normal readiness transition. After a cluster is usable, lost or stale trust sets `DEGRADED=True` with a trust-specific reason and makes `AVAILABLE=False` while leaving lifecycle state `READY`; recovery clears `DEGRADED` and restores `AVAILABLE` when the other health requirements are satisfied. Each condition records the evaluated `observedGeneration`. The remaining scope decision covers only dedicated CLI, UI, and additional troubleshooting indicators beyond the existing `ClusterOrder` status.

## Remaining Gaps

- Whether dedicated CLI, UI, or additional troubleshooting indicators beyond the existing `ClusterOrder` status are in scope for this Feature.
