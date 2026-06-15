# Tenant On-boarding

| Field       | Value   |
|-------------|---------|
| **Status**  | Proposed |
| **Feature** | OSAC-24 |
| **Authors** | TBD |
| **Date**    | 2026-06-15 |

## Summary

Automate tenant infrastructure provisioning by creating Tenant Custom Resources (CRs) on management hubs when new tenants are added to the OSAC fulfillment database. This provides a trigger mechanism for Storage, Networking, and Authentication components to asynchronously provision tenant-specific infrastructure. The enhancement includes renaming the "Organization" API entity to "Tenant" for terminology consistency across OSAC.

## Motivation

### Problem Statement

OSAC provides a multi-tenant cloud infrastructure platform where Cloud Provider Admins create Organizations (being renamed to Tenants) via the fulfillment API. However, when a new tenant is created, there is no automated mechanism to provision required infrastructure components—storage backends, networking resources, and compute namespaces—across management hubs. This forces manual intervention for each tenant onboarding, creating operational overhead and delaying tenant readiness.

Without an automated trigger mechanism, infrastructure teams must coordinate separately to configure per-tenant storage classes, virtual networks, and namespace isolation, leading to inconsistent provisioning patterns and increased time-to-ready.

### Terminology Inconsistency

OSAC currently uses "Organization" as the API entity name, but:
- The osac-operator uses "Tenant" for its Kubernetes CRD
- The broader OSAC architecture refers to "multi-tenant" infrastructure
- Keycloak realm management doesn't map cleanly to "Organization"

This inconsistency causes confusion for developers and users working across different OSAC components.

### Goals

- Automatically create a Tenant CR on management hubs when a new tenant is added to the OSAC fulfillment database
- Rename the "Organization" API entity to "Tenant" throughout OSAC APIs and Keycloak
- Enable the Core team to orchestrate tenant creation while individual working groups (Storage, Networking) own reconciliation and configuration
- Ensure the Fulfillment Service acts as the source of truth for Tenant CRs, automatically recreating them if manually deleted

### Non-Goals

- Complex status reporting for tenant infrastructure readiness (deferred to follow-up enhancement)
- Manual Tenant CR creation for production use (manual CRs supported for demos only)
- gRPC-based alternative to Tenant CR for multi-hub object watching (deferred to follow-up)
- Quota enforcement, billing integration, or advanced infrastructure policies during onboarding

## Proposal

### High-Level Design

The Tenant On-boarding enhancement consists of three parts:

1. **API Renaming**: Rename Organization → Tenant throughout OSAC APIs, database, and system components
2. **Tenant CR Schema**: Extend the Tenant CR with `name` and `emailDomains` spec fields
3. **Fulfillment Controller**: Implement a controller in fulfillment-service that watches the database and manages Tenant CR lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│  Cloud Provider Admin                                           │
│  └─> POST /v2/tenants (name, emailDomains)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Fulfillment Service (PostgreSQL)                               │
│  └─> INSERT INTO tenants (name, email_domains)                  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Fulfillment Service Controller (watches DB)                    │
│  └─> Detects new tenant → Create Tenant CR on management hubs   │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  Management Hubs (Kubernetes)                                   │
│  └─> Tenant CR created in default namespace                     │
└────────────────────────────┬────────────────────────────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
    ┌──────────┐      ┌──────────┐      ┌──────────┐
    │ Storage  │      │Network   │      │  Auth    │
    │Controller│      │Controller│      │Controller│
    └──────────┘      └──────────┘      └──────────┘
         │                 │                 │
         ▼                 ▼                 ▼
    Per-tenant       Tenant-specific    Keycloak
    StorageClasses   VirtualNetworks    Realm
```

### Tenant CR Lifecycle

1. **Creation**: When a tenant is added to the fulfillment database, the fulfillment-service controller creates a Tenant CR on relevant management hubs
2. **Reconciliation**: Component controllers (Storage, Networking, Authentication) watch for Tenant CR events and asynchronously provision infrastructure
3. **Deletion**: When a tenant is deleted from the database, the controller deletes the Tenant CRs it created
4. **Recovery**: If a Tenant CR is manually deleted, the controller recreates it during the next reconciliation cycle

### Tenant CR Schema

```yaml
apiVersion: osac.openshift.io/v1alpha1
kind: Tenant
metadata:
  name: acme-corp
  namespace: default
  annotations:
    osac.openshift.io/managed-by: fulfillment-service
spec:
  name: acme-corp
  emailDomains:
    - acme.com
    - acme.net
status:
  # Status reporting deferred to follow-up enhancement
```

### Multi-Hub Distribution

The Fulfillment Service creates Tenant CRs on management hubs where infrastructure components need to provision tenant-specific resources. Hub selection logic is based on tenant configuration or deployment topology (specific logic TBD during implementation).

## Implementation

### Epic Breakdown

| # | Epic | Size | Stories | Scope |
|---|------|------|---------|-------|
| 1 | [API Renaming: Organization → Tenant](#epic-1-api-renaming) | M | 5 | Rename throughout APIs, database, osac-operator, Keycloak |
| 2 | [Tenant CR Schema Extension](#epic-2-tenant-cr-schema) | S | 3 | Add `name` + `emailDomains` fields |
| 3 | [Fulfillment-Service Tenant Controller](#epic-3-fulfillment-controller) | L | 7 | Database watch + CR lifecycle automation |

**Total Effort: Large** (15 stories across 3 epics)

### Epic 1: API Renaming

**Size:** Medium  
**Stories:** 5

Rename the Organization API entity to Tenant throughout OSAC APIs, database, and system components.

**Implementation:**

1. **Proto Rename** (fulfillment-service)
   - Rename `organization_type.proto` → `tenant_type.proto`
   - Rename `organizations_service.proto` → `tenants_service.proto`
   - Update message names: `Organization` → `Tenant`
   - Update service names: `Organizations` → `Tenants`
   - Regenerate: `buf lint && buf generate`

2. **Database Migration** (fulfillment-service)
   - Migration: `ALTER TABLE organizations RENAME TO tenants;`
   - Update DAO: `GenericDAO[*v1.Tenant]`
   - Update server implementations: `TenantServer`, `PrivateTenantServer`

3. **Server Rename** (fulfillment-service)
   - Update all server method signatures and implementations
   - Update integration tests

4. **Operator Buf Update** (osac-operator)
   - Update buf dependency to new fulfillment-service proto version
   - Regenerate: `buf generate`
   - Update controller imports and type references

5. **Integration Tests** (cross-repo)
   - Validate end-to-end after all components updated

**Backward Compatibility Strategy:**

⚠️ **OPEN QUESTION - REQUIRES STAKEHOLDER DECISION**

This is a breaking API change. Two options have been identified but no formal decision has been documented:

- **Option 1:** Bump API version to v2, deprecate v1 Organizations service for one release cycle
  - **Impact:** Breaking change requiring coordinated client updates (osac CLI, osac-operator, osac-installer)
  - **Pros:** Clean separation, no technical debt
  - **Cons:** Multi-release deployment complexity, client migration effort
  
- **Option 2:** Alias Organizations → Tenants in v1
  - **Impact:** Non-breaking, clients can migrate on their own timeline
  - **Pros:** No forced client updates, gradual migration
  - **Cons:** Creates ongoing maintenance burden for dual naming support

**Decision needed:** Which option should be taken? Who has authority to approve (Product Owner, Engineering Lead, or architectural review)?

### Epic 2: Tenant CR Schema Extension

**Size:** Small  
**Stories:** 3

Extend the Tenant CR with spec fields (`name`, `emailDomains`) that the fulfillment-service controller will populate.

**Implementation:**

1. **CRD Field Additions** (osac-operator)
   - Add `spec.name` (string, required)
   - Add `spec.emailDomains` ([]string, required)
   - Update CRD manifests and generated types

2. **Validation Rules** (osac-operator)
   - Add kubebuilder validation markers
   - Add admission webhook validation (email domain format)

3. **Integration Tests** (osac-operator)
   - Validate CR creation with spec fields
   - Test validation rules

### Epic 3: Fulfillment-Service Tenant Controller

**Size:** Large  
**Stories:** 7

Implement the database watch and CR lifecycle management logic in fulfillment-service.

**Implementation:**

1. **Controller Scaffold**
   - Set up controller-runtime manager
   - Add Kubernetes client for multi-cluster CR creation

2. **Database Watch**
   - Implement PostgreSQL LISTEN/NOTIFY or polling mechanism
   - Detect tenant INSERT/UPDATE/DELETE events

3. **CR CRUD Logic**
   - Create Tenant CR on management hubs when tenant added to DB
   - Delete Tenant CR when tenant removed from DB
   - Update CR when tenant modified in DB

4. **Managed-By Annotation**
   - Add `osac.openshift.io/managed-by: fulfillment-service` to CRs
   - Filter reconciliation to only CRs with this annotation

5. **Reconciliation Loop**
   - Periodic reconciliation to detect manually deleted CRs
   - Recreate missing CRs that should exist based on DB state

6. **Hub Selection**
   - Implement logic to determine which management hubs receive the CR
   - Read hub configuration from fulfillment-service config

7. **Integration Tests**
   - Test full lifecycle: create tenant → CR appears → delete tenant → CR removed
   - Test reconciliation recovery (manual CR deletion)

## Requirements

### Functional Requirements

#### API Renaming
- **FR-1:** The "Organization" API entity must be renamed to "Tenant" throughout OSAC APIs, Keycloak nomenclature, and all system components

#### Tenant CR Lifecycle
- **FR-2:** When a new tenant entry is added to the fulfillment database, the Fulfillment Service must automatically create a Tenant CR on the relevant management hubs
- **FR-3:** The Tenant CR must reside in the default namespace on each management hub
- **FR-4:** When a tenant is deleted from the fulfillment database, the Fulfillment Service must automatically delete the Tenant CRs it created
- **FR-5:** If a Tenant CR managed by the Fulfillment Service is manually deleted, the controller must detect the absence during the next reconciliation cycle and automatically recreate the CR
- **FR-6:** The Fulfillment Service must only create and delete Tenant CRs that it initiated; it must not interfere with manually created CRs (used for demos)

#### Tenant CR Specification
- **FR-7:** The Tenant CR specification must include the tenant name as a required field
- **FR-8:** The Tenant CR specification must include a list of email domains as a required field

#### Component Integration
- **FR-9:** Storage controllers must watch for Tenant CR creation events and asynchronously provision per-tenant storage classes
- **FR-10:** Networking controllers must watch for Tenant CR creation events and asynchronously create tenant-specific namespaces and networking resources
- **FR-11:** Authentication controllers (from OSAC-66) must watch for Tenant CR creation events and asynchronously create Keycloak realms

#### Multi-Hub Distribution
- **FR-12:** The Fulfillment Service must create Tenant CRs on the management hubs where infrastructure components need to provision tenant-specific resources

### Non-Functional Requirements

- **NFR-1:** Tenant CR creation must be idempotent
- **NFR-2:** Component controllers must reconcile asynchronously after Tenant CR creation without blocking the core platform's tenant readiness state
- **NFR-3:** The Fulfillment Service controller must reconcile Tenant CRs independently of whether the CR was initiated by the Fulfillment Service or manually created

## Dependencies

- **OSAC-66 (Extend organizations and authentication management)**: Must be complete to provide the foundational Organization/Tenant API framework and Keycloak realm management
- **OSAC-43 (VAST for VMaaS)**: Storage component integration with Tenant CR watching (currently in Backlog, deprioritized)
- **OSAC-56 (VMaaS Tenant Storage Setup)**: TenantStorage CR integration and StorageClass installation (In Progress as stretch goal for v0.1)

**Note:** OSAC-24 provides the Tenant CR trigger mechanism. Storage and Networking components (OSAC-43, OSAC-56) will integrate when their respective epics complete.

## Risks and Mitigations

### Dependency Epic Completion Timeline

**Risk:** Storage (OSAC-43) and Networking (OSAC-56) epics are not complete, so component integration won't be fully validated in OSAC-24 timeline.

**Mitigation:** OSAC-24 can proceed to implement the Tenant CR infrastructure independently. Storage and Networking components will integrate when their respective epics complete. The Tenant CR serves as a stable trigger mechanism that components can watch asynchronously.

**Owner:** Product Owner or Engineering Lead

### Multi-Hub CR Synchronization

**Risk:** Tenant CRs must be created across multiple management hubs. Network failures, hub unavailability, or CR creation failures on some hubs could lead to inconsistent tenant state.

**Mitigation:** 
- Implement retry logic with exponential backoff for CR creation failures
- Track per-hub CR creation status in fulfillment-service (pending/created/failed)
- Add monitoring alerts for CR synchronization failures
- Document operational runbooks for manual recovery

**Owner:** Core Team Technical Lead

### Backward Compatibility Impact

**Risk:** API renaming (Organization → Tenant) is a breaking change that requires coordinated client updates if Option 1 (API v2 bump) is chosen.

**Mitigation:** 
- If Option 1: Plan multi-release deployment with v1 deprecation period, update all clients in parallel
- If Option 2: Accept technical debt of dual naming support for gradual migration

**Owner:** Architectural Review Board (decision needed)

## Acceptance Criteria

- [ ] The Organization API entity has been renamed to Tenant in all API definitions, Keycloak configurations, and system documentation
- [ ] When a Cloud Provider Admin creates a new tenant via the fulfillment API, a Tenant CR is automatically created on the management hubs within one reconciliation cycle
- [ ] The Tenant CR includes the tenant name and email domains from the fulfillment database
- [ ] When a tenant is deleted via the fulfillment API, the Tenant CRs managed by the Fulfillment Service are automatically deleted from the management hubs
- [ ] When a Tenant CR is manually deleted, the Fulfillment Service recreates it during the next reconciliation cycle
- [ ] Storage controllers create per-tenant storage classes when a Tenant CR appears (integration with OSAC-43/OSAC-56 when those epics complete)
- [ ] Networking controllers create tenant-specific namespaces and networking resources when a Tenant CR appears
- [ ] The Tenant CR does not include complex status fields in the initial implementation

## Open Questions

### 1. Backward Compatibility Strategy (Epic 1)

**Question:** Should the Organization→Tenant API renaming use Option 1 (API v2 bump, breaking change) or Option 2 (aliasing, non-breaking)?

**Decision needed from:** Product Owner, Engineering Lead, or Architectural Review Board

**Impact:** Affects deployment complexity, client migration effort, and technical debt

### 2. Multi-Hub Selection Logic (Epic 3)

**Question:** Which management hubs should receive the Tenant CR when a new tenant is created?
- All configured management hubs (every hub where osac-operator is deployed)?
- Tenant-specific configuration (based on region, capacity, or other metadata)?

**Decision needed from:** Core Team Technical Lead

**Impact:** Affects FR-12 implementation and hub selection logic in Story 3.06

### 3. Error Observability Without Status Reporting

**Question:** If status reporting is deferred to a follow-up enhancement, how should administrators detect that storage or networking setup failed during component reconciliation?

**Options:**
- Inspect individual component controller logs and CR conditions directly on each hub
- Provide minimal observability mechanism (e.g., "check TenantStorage CR status on hub X")
- Document operational runbooks for troubleshooting

**Decision needed from:** Product Owner or SRE Lead

**Impact:** Affects operational documentation and admin workflows

## Alternatives Considered

### gRPC-Based Multi-Hub Watching

**Alternative:** Instead of creating Tenant CRs across every hub, use the fulfillment-service gRPC API for object watching. Component controllers would watch the central fulfillment-service instead of local CRs.

**Pros:**
- Eliminates CR proliferation across hubs
- Centralized source of truth
- Simplifies multi-hub synchronization

**Cons:**
- Requires significant shift from existing Kubernetes-native patterns
- Increases coupling to fulfillment-service availability
- Adds network dependency for all component controllers

**Decision:** Deferred to follow-up enhancement. Proceed with Tenant CR approach as first pass, revisit gRPC-based approach after validating CR-based workflow.

### Include Status Reporting in OSAC-24

**Alternative:** Implement aggregated status reporting from component controllers (Storage, Networking, Auth) in the initial Tenant CR implementation.

**Pros:**
- Better observability for administrators
- Clear signal of tenant readiness

**Cons:**
- Increases complexity and scope of OSAC-24
- Blocks delivery while waiting for component integration (OSAC-43, OSAC-56)
- Architectural questions remain (how to aggregate status from multiple components)

**Decision:** Explicitly deferred to follow-up enhancement. OSAC-24 delivers the trigger mechanism; status reporting is a separate concern.

## References

- **Jira Feature:** [OSAC-24](https://redhat.atlassian.net/browse/OSAC-24)
- **Blocking Dependencies:**
  - [OSAC-66: Extend organizations and authentication management](https://redhat.atlassian.net/browse/OSAC-66)
  - [OSAC-43: VAST for VMaaS](https://redhat.atlassian.net/browse/OSAC-43)
  - [OSAC-56: VMaaS Tenant Storage Setup](https://redhat.atlassian.net/browse/OSAC-56)
- **Meeting Notes:** [Define OSAC-996: Tenant On-boarding](https://docs.google.com/document/d/1-vlpq5MwNBg3F15X6D6yjzPGrxhA_2lFO__KPn8lBw4/edit) (June 1, 2026)
