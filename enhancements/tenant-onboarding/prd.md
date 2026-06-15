# Tenant On-boarding

| Field       | Value   |
|-------------|---------|
| Author(s)   | To be determined |
| Jira        | https://redhat.atlassian.net/browse/OSAC-24 |
| Date        | 2026-06-15 |

## 1. Problem Statement

OSAC provides a multi-tenant cloud infrastructure platform where Cloud Provider Admins can create Organizations (being renamed to Tenants for terminology consistency) via the fulfillment API. However, when a new tenant is created, there is currently no automated mechanism to provision the required infrastructure components—storage backends, networking resources, and compute namespaces—across the management hubs. This forces manual intervention for each tenant onboarding, creating operational overhead and delaying tenant readiness. Without an automated trigger mechanism, infrastructure teams must coordinate separately to configure per-tenant storage classes, virtual networks, and namespace isolation, leading to inconsistent provisioning patterns and increased time-to-ready for new tenants.

## 2. Goals and Non-Goals

### 2.1 Goals

- Automatically create a Tenant Custom Resource (CR) on management hubs when a new tenant is added to the OSAC fulfillment database, triggering asynchronous infrastructure provisioning by component controllers.
- Rename the "Organization" API entity to "Tenant" throughout OSAC APIs and Keycloak to ensure consistent terminology across the platform.
- Enable the Core team to orchestrate tenant creation while individual working groups (Storage, Networking) own reconciliation and configuration of their respective infrastructure components.
- Ensure the Fulfillment Service acts as the source of truth for Tenant CRs, automatically recreating them if manually deleted during reconciliation cycles.

### 2.3 Non-Goals

- Complex status reporting for tenant infrastructure readiness. Basic Tenant CR creation and core platform readiness are in scope; aggregated status from downstream components (storage, networking) is deferred to a follow-up enhancement.
- Manual Tenant CR creation for production use. Manual CR creation may occur for demo or development purposes but is not supported or validated by the system. The Fulfillment Service only manages CRs it creates.
- gRPC-based alternative to Tenant CR for multi-hub object watching. This architectural pattern is deferred to a follow-up enhancement after initial CR-based approach is implemented.
- Quota enforcement, billing integration, or advanced infrastructure policies during tenant onboarding. These are configured separately after tenant creation.

## 3. Requirements

### 3.1 Functional Requirements

#### API Renaming

- **FR-1:** The "Organization" API entity must be renamed to "Tenant" throughout OSAC APIs, Keycloak nomenclature, and all system components to ensure consistent terminology.

#### Tenant CR Lifecycle

- **FR-2:** When a new tenant entry is added to the fulfillment database, the Fulfillment Service must automatically create a Tenant CR on the relevant management hubs with the tenant name and email domains from the database entry.
- **FR-3:** The Tenant CR must reside in the default namespace on each management hub.
- **FR-4:** When a tenant is deleted from the fulfillment database, the Fulfillment Service must automatically delete the Tenant CRs it created on the management hubs.
- **FR-5:** If a Tenant CR managed by the Fulfillment Service is manually deleted, the Fulfillment Service controller must detect the absence during the next reconciliation cycle and automatically recreate the CR to maintain consistency.
- **FR-6:** The Fulfillment Service must only create and delete Tenant CRs that it initiated; it must not interfere with manually created CRs (used for demos).

#### Tenant CR Specification

- **FR-7:** The Tenant CR specification must include the tenant name as a required field.
- **FR-8:** The Tenant CR specification must include a list of email domains as a required field.

#### Component Integration

- **FR-9:** Storage controllers must watch for Tenant CR creation events and asynchronously provision per-tenant storage classes based on configured storage tiers.
- **FR-10:** Networking controllers must watch for Tenant CR creation events and asynchronously create tenant-specific namespaces and networking resources on the relevant clusters.
- **FR-11:** Authentication controllers (from OSAC-66) must watch for Tenant CR creation events and asynchronously create Keycloak realms for the tenant.

#### Multi-Hub Distribution

- **FR-12:** The Fulfillment Service must create Tenant CRs on the management hubs where infrastructure components need to provision tenant-specific resources. The hub selection logic is to be determined (see Open Question 8.2).

### 3.2 Non-Functional Requirements

- **NFR-1:** Tenant CR creation must be idempotent. Re-running creation for an existing tenant must skip already-created resources without errors.
- **NFR-2:** Component controllers must reconcile asynchronously after Tenant CR creation without blocking the core platform's tenant readiness state.
- **NFR-3:** The Fulfillment Service controller must reconcile Tenant CRs independently of whether the CR was initiated by the Fulfillment Service or manually created.

## 4. Acceptance Criteria

- [ ] The Organization API entity has been renamed to Tenant in all API definitions, Keycloak configurations, and system documentation.
- [ ] When a Cloud Provider Admin creates a new tenant via the fulfillment API, a Tenant CR is automatically created on the management hubs within one reconciliation cycle.
- [ ] The Tenant CR includes the tenant name and email domains from the fulfillment database.
- [ ] When a tenant is deleted via the fulfillment API, the Tenant CRs managed by the Fulfillment Service are automatically deleted from the management hubs.
- [ ] When a Tenant CR is manually deleted, the Fulfillment Service recreates it during the next reconciliation cycle.
- [ ] Storage controllers create per-tenant storage classes when a Tenant CR appears (integration with OSAC-43 and OSAC-56 when those epics complete).
- [ ] Networking controllers create tenant-specific namespaces and networking resources when a Tenant CR appears.
- [ ] The Tenant CR does not include complex status fields in the initial implementation; status reporting is deferred to a follow-up enhancement.

## 5. Assumptions

- Hub selection for Tenant CR creation is determined by tenant configuration or deployment topology. The specific hub selection logic has not been defined in the source material.
- The Storage and Networking controllers' integration with Tenant CR watching is part of their respective epics (OSAC-43, OSAC-56) and will be implemented when those epics complete. OSAC-24 provides the Tenant CR trigger mechanism; component integration happens asynchronously.
- The Tenant CR schema (beyond name and email domains) will be defined during implementation and may include additional metadata fields for component controllers.
- OSAC-66 (Organizations and Authentication) must be substantially complete to provide the foundational API and Keycloak integration that the Tenant CR depends on.

## 6. Dependencies

- **OSAC-66 (Extend organizations and authentication management)**: Must be complete to provide the foundational Organization/Tenant API framework and Keycloak realm management. OSAC-24 builds on this foundation by adding the Tenant CR orchestration layer.
- **OSAC-43 (VAST for VMaaS)**: Storage component integration with Tenant CR watching. OSAC-24 creates the Tenant CR trigger; OSAC-43 implements the storage controller that responds to it. This epic is currently in Backlog and deprioritized.
- **OSAC-56 (VMaaS Tenant Storage Setup)**: TenantStorage CR integration and StorageClass installation on VMaaS clusters. OSAC-24 provides the trigger mechanism; OSAC-56 implements the storage setup workflow. This epic is In Progress as a stretch goal for v0.1.
- **Fulfillment Service database schema**: Tenant name and email domain fields must exist in the database for the Fulfillment Service to read during Tenant CR creation.

## 7. Risks

### 7.1 Dependency Epic Completion Timeline

- **Owner:** To be determined (Product Owner or Engineering Lead)
- **Mitigation:** OSAC-24 can proceed to implement the Tenant CR infrastructure independently. Storage and Networking components will integrate when their respective epics complete. The Tenant CR serves as a stable trigger mechanism that components can watch asynchronously.

### 7.2 Multi-Hub CR Synchronization

- **Owner:** To be determined (Core Team Technical Lead)
- **Mitigation:** Implement retry logic with exponential backoff for CR creation failures. Track per-hub CR creation status in fulfillment-service (pending/created/failed). Add monitoring alerts for CR synchronization failures. Document operational runbooks for manual recovery.

### 7.3 Backward Compatibility Impact

- **Owner:** To be determined (Architectural Review Board)
- **Mitigation:** To be determined — decision needed on API versioning strategy (Option 1: v2 bump with v1 deprecation period, or Option 2: aliasing in v1).

## 8. Open Questions

### 8.1 Cloud Provider Admin Workflow

**Question:** When a Cloud Provider Admin onboards a new tenant, what is the end-to-end sequence? Do they create a tenant via the fulfillment-service API (POST to `/tenants` with name + email domains), or does tenant creation happen via some other admin interface or CLI? Is there a confirmation step or does tenant creation happen immediately?

- **Owner:** Product Owner
- **Impact:** Affects user stories and API specification for FR-2

### 8.2 Multi-Hub CR Distribution Logic

**Question:** Which management hubs receive the Tenant CR when a new tenant is created? Is it all configured management hubs (every hub where osac-operator is deployed), or does the fulfillment service use tenant-specific configuration to determine which hubs should get the CR (based on geographic region, cluster capacity, or other tenant metadata)?

- **Owner:** Core Team Technical Lead
- **Impact:** Affects FR-12 implementation and hub selection logic

### 8.3 Error Observability Without Status Reporting

**Question:** Given that status reporting is deferred to a follow-up enhancement, how should administrators detect that storage or networking setup failed during component reconciliation? Should they inspect individual Storage/Networking controller logs or CR conditions directly on each hub, or is there a minimal observability mechanism that should be documented?

- **Owner:** SRE Lead or Product Owner
- **Impact:** Affects operational documentation and admin troubleshooting workflows

### 8.4 Tenant CR Spec Field Details

**Question:** Are there any fields in the Tenant CR spec beyond name and email domains? Is email domains a single string (e.g., `"example.com"`) or a list of domains (e.g., `["example.com", "example.org"]`)? Are there any optional fields that should be supported now (tenant description, contact info, quota hints) or is the spec intentionally minimal for OSAC-24?

- **Owner:** Core Team Technical Lead
- **Impact:** Affects FR-7 and FR-8 implementation and Tenant CR schema design

### 8.5 Component Cleanup Expectations on Deletion

**Question:** When the fulfillment service deletes a Tenant CR, should OSAC-24 document expected cleanup behavior for Storage/Networking even if those components aren't fully implemented yet (design contract for how cleanup should work once components are ready), or is deletion scoped to "just delete the Tenant CR" and cleanup integration is entirely out of scope for OSAC-24?

- **Owner:** Product Owner
- **Impact:** Affects FR-4 scope and deletion behavior documentation

### 8.6 API Renaming Backward Compatibility Strategy

**Question:** Should the Organization→Tenant API renaming use Option 1 (bump API version to v2, deprecate v1 Organizations service for one release cycle, requiring coordinated client updates) or Option 2 (alias Organizations → Tenants in v1, non-breaking but creates ongoing maintenance burden)? Who has authority to approve this decision (Product Owner, Engineering Lead, or Architectural Review Board)? What is the acceptable impact on existing deployments and clients (osac CLI, osac-operator, osac-installer)?

- **Owner:** Architectural Review Board
- **Impact:** Affects FR-1 implementation, Epic 1 effort, deployment complexity, and technical debt
