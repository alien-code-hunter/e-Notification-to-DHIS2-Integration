# System Architecture - e-Notification to DHIS2 Integration

## 1. High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                      e-Notification System                          │
│  (Death Notification Data Entry & Management)                       │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             │ HTTPS/REST API
                             │ JSON Payload
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    OpenHIM Mediator Layer                           │
│                (Health Information Mediator)                        │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │           OpenHIM Core Mediator Engine                       │  │
│  │                                                              │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐   │  │
│  │  │   Endpoint  │  │  Channels    │  │   Routes &      │   │  │
│  │  │  Registry   │  │ - E-Notif In │  │  Orchestration  │   │  │
│  │  │ - Auth      │  │ - DHIS2 Out  │  │ - Request/Resp  │   │  │
│  │  │ - Roles     │  │ - Webhooks   │  │   Mapping       │   │  │
│  │  │ - Certs     │  │              │  │ - Transformation│   │  │
│  │  └─────────────┘  └──────────────┘  └──────────────────┘   │  │
│  │                                                              │  │
│  │  ┌──────────────────────────────────────────────────────┐   │  │
│  │  │  Built-in Mediator Services                         │   │  │
│  │  │  - Authentication (TLS, Basic, Custom)              │   │  │
│  │  │  - Authorization (Role-based access control)        │   │  │
│  │  │  - Audit Logging (Comprehensive request/response)   │   │  │
│  │  │  - Message Queueing (Async routing & retry)         │   │  │
│  │  │  - Rate Limiting                                    │   │  │
│  │  │  - Response Caching                                 │   │  │
│  │  │  - Webhook Management (Verification callbacks)      │   │  │
│  │  │  - Data Transformation (Field mapping)              │   │  │
│  │  └──────────────────────────────────────────────────────┘   │  │
│  │                                                              │  │
│  │  ┌──────────────────────────────────────────────────────┐   │  │
│  │  │  OpenHIM Endpoints (Channels Configured)            │   │  │
│  │  │  POST   /death-notifications       (e-Notif → Queue) │   │  │
│  │  │  GET    /death-notifications/{id}  (Query endpoint)  │   │  │
│  │  │  PATCH  /death-notifications/{id}/verify (Verify)   │   │  │
│  │  │  + Additional channels for DHIS2 integration        │   │  │
│  │  └──────────────────────────────────────────────────────┘   │  │
│  │                                                              │  │
│  │  ┌──────────────────────────────────────────────────────┐   │  │
│  │  │  Data Persistence Layer                              │   │  │
│  │  │  - OpenHIM Transactions DB (MongoDB)                │   │  │
│  │  │  - Notification Status DB (PostgreSQL)              │   │  │
│  │  │  - Audit Logs (OpenHIM native)                      │   │  │
│  │  └──────────────────────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                          │                      │                   │
│                    HTTPS  │    Async Callback   │                   │
│                    (TLS)  │                     │                   │
└──────────────────────────┼─────────────────────┼──────────────────┘
                           │                      │
                           │                      │
┌──────────────────────────▼──────────────────────▼──────────────────┐
│                    DHIS2 Inpatient System                          │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │                DHIS2 Tracker                               │   │
│  │  - Tracked Entities (Patients/Persons)                     │   │
│  │  - Program: Death Notification Tracking                    │   │
│  │  - Program Stage: Death Event Recording                    │   │
│  │  - Data Elements (20 fields): Death Details                │   │
│  │  - Option Sets (5): Categorical Values                     │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │              DHIS2 Backend Services                         │   │
│  │  - API: /api/metadata (import data elements)                │   │
│  │  - API: /api/programs (death notification program)          │   │
│  │  - API: /api/events (create death events)                   │   │
│  │  - API: /api/tracked-entities (link patients)               │   │
│  │  - Authentication: User/Password or OAuth 2.0               │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │              DHIS2 Database                                 │   │
│  │  - PostgreSQL/MySQL: Patient Records, Death Events          │   │
│  │  - Audit Log: Track Events, Verification Actions            │   │
│  └────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Detailed Component Architecture (OpenHIM-Centric)

### A. OpenHIM Mediator Request Processing

```
Request Flow: e-Notification → OpenHIM Listeners → Transformation → DHIS2

┌─────────────────────────────────────────────────────┐
│           REST API Request                          │
│  (Death Notification Payload - JSON)                │
│  From e-Notification system with TLS cert           │
└─────────────────────┬───────────────────────────────┘
                      │
        ┌─────────────▼──────────────────────┐
        │   OpenHIM Secure Listener           │
        │   (Port 5001 - TLS, mTLS capable)  │
        │  - Client cert authentication       │
        │  - TLS encryption (in-transit)     │
        │  - Request buffering                │
        │  - Accept JSON payload              │
        └─────────────┬──────────────────────┘
                      │
        ┌─────────────▼──────────────────────┐
        │   OpenHIM Channel Router            │
        │  - Match to configured channel     │
        │  - Route: "DeathNotificationIn"    │
        │  - Check rate limiting rules       │
        │  - Verify RBAC permissions         │
        └─────────────┬──────────────────────┘
                      │
        ┌─────────────▼──────────────────────┐
        │   OpenHIM Transformation Script    │
        │   (JavaScript or Groovy)           │
        │  - Validate input structure        │
        │  - Map e-Notification → DHIS2      │
        │  - Call custom service endpoint    │
        │    (ICD-10 lookup)                 │
        │  - Call custom service endpoint    │
        │    (Duplicate detection)           │
        │  - Transform to DHIS2 format       │
        │  - Prepare request body            │
        └─────────────┬──────────────────────┘
                      │
        ┌─────────────▼──────────────────────┐
        │   OpenHIM Primary Route             │
        │   OR Queue for Async              │
        │  - Synchronous: Direct DHIS2 call │
        │  - Asynchronous: Queue for later   │
        │  - Automatic retry (exponential)   │
        │  - Dead letter queue on max retry  │
        └─────────────┬──────────────────────┘
                      │
        ┌─────────────▼──────────────────────┐
        │   OpenHIM Transaction Log          │
        │   (Saved to MongoDB)               │
        │  - Request received                │
        │  - Response from DHIS2             │
        │  - Transformation details          │
        │  - Timestamps & identities         │
        │  - Status codes                    │
        └─────────────┬──────────────────────┘
                      │
    ┌─────────────────▼─────────────────────┐
    │  Response to e-Notification           │
    │  - HTTP 201 Created (synchronous)    │
    │  - Include OpenHIM transaction ID     │
    │  - Include DHIS2 event ID             │
    │  - Status: ACCEPTED/QUEUED            │
    └───────────────────────────────────────┘
```

### B. OpenHIM Async Queue & Webhook Processing

```
┌──────────────────────────────────────┐
│   OpenHIM Queue (Built-in)           │
│   - Message: Queued notification     │
│   - Status: PENDING processing       │
│   - Retryable: Yes (exponential)     │
│   - Max retries: 5 (configurable)    │
└────────────┬─────────────────────────┘
             │
    ┌────────▼──────────────────────────┐
    │  OpenHIM Channel Execution         │
    │  (Async routine)                  │
    │  - Dequeue message                │
    │  - Run transformation script       │
    │  - Cache responses if configured   │
    └────────┬──────────────────────────┘
             │
    ┌────────▼──────────────────────────┐
    │  Call Custom Service               │
    │  - ICD-10 validation API           │
    │  - Duplicate detection API         │
    │  - Patient lookup/matching         │
    └────────┬──────────────────────────┘
             │
    ┌────────▼──────────────────────────┐
    │  Call DHIS2 API                    │
    │  - Endpoint: /api/events           │
    │  - Method: POST                    │
    │  - Auth: Basic Auth (configured)   │
    │  - Body: Transformed data          │
    └────────┬──────────────────────────┘
             │
         ┌───▼──────────────┐
         │ Success?         │
         └───┬──────────┬───┘
             │          │
        Yes  │          │  No (DHIS2 error)
             │          │
    ┌────────▼──┐    ┌──▼──────────────┐
    │ Mark in   │    │  OpenHIM Retry  │
    │ MongoDB:  │    │  - Exponential  │
    │ SUCCESS   │    │    backoff      │
    │ Record    │    │  - Re-queue msg │
    │ ext_id    │    │  - Log attempt  │
    │ & DHIS2   │    │  - If max retry │
    │ event_id  │    │    → Dead Letter│
    └────────┬──┘    └─┬────────────────┘
             │         │
    ┌────────▼─────────▼────────────────┐
    │  OpenHIM Webhook Channel           │
    │  (Configured callback)             │
    │  - Endpoint: e-Notification system │
    │  - Method: POST                    │
    │  - Auth: Signature (HMAC-SHA256)   │
    │  - Body: Status update             │
    └────────┬─────────────────────────┘
             │
    ┌────────▼─────────────────────────┐
    │ e-Notification receives callback   │
    │ Updates its UI/status              │
    │ Marks notification as SYNCED       │
    └────────────────────────────────────┘
```

### C. Data Flow - Death Notification Entry to DHIS2

```
Step 1: e-Notification System
┌─────────────────────────────────────────────────────────┐
│ User enters death notification:                         │
│ - Patient ID: 123456                                   │
│ - Patient Name: John Doe                               │
│ - Date of Death: 2024-02-15                            │
│ - Place of Death: Health Facility                       │
│ - Primary Cause: Malaria (B54)                          │
│ - Secondary Cause: Pneumonia (J18.9)                    │
│ - Age at Death: 45                                      │
│ ... (15 total fields)                                   │
└─────────────────────────────────────────────────────────┘
                     │
                     │ POST /death-notifications
                     ▼
Step 2: Integration Layer - Validation
┌─────────────────────────────────────────────────────────┐
│ Request Validation:                                     │
│ ✓ All required fields present                           │
│ ✓ Data types correct                                    │
│ ✓ Date validation (not future, not before birth)        │
│ ✓ ICD-10 code format valid                              │
│ ✓ Phone number format (if provided)                     │
│ ✓ Unique patient identification possible                │
└─────────────────────────────────────────────────────────┘
                     │
                     ▼
Step 3: Duplicate Detection
┌─────────────────────────────────────────────────────────┐
│ Check for duplicates:                                   │
│ Query: National ID + DOB + first name                   │
│ Result: No existing death record found                  │
└─────────────────────────────────────────────────────────┘
                     │
                     ▼
Step 4: Patient Matching
┌─────────────────────────────────────────────────────────┐
│ Find patient in DHIS2:                                  │
│ - Search by National ID                                 │
│ - Match: Found existing tracked entity                  │
│ - DHIS2 Tracked Entity ID: a1b2c3d4e5f6                 │
│ - Existing enrollments: 1 (cervical cancer program)     │
└─────────────────────────────────────────────────────────┘
                     │
                     ▼
Step 5: Data Mapping
┌──────────────────────────────────────────────────────────┐
│ e-Notification Field → DHIS2 Data Element               │
│ ─────────────────────────────────────────────────────    │
│ patient_id → deathNotifPatID                            │
│ patient_name → deathNotifPatName                        │
│ dob → deathNotifPatDOB                                  │
│ sex → deathNotifPatSex (option set)                     │
│ death_date → deathNotifDeathDate                        │
│ death_place → deathNotifPlaceOfDeath (option set)       │
│ death_facility → deathNotifFacility                     │
│ primary_cause → deathNotifPrimaryCause (ICD-10)        │
│ secondary_cause → deathNotifSecondaryCause             │
│ age_at_death → deathNotifAgeAtDeath                     │
│ admission_date → deathNotifAdmissionDate                │
│ length_of_stay → deathNotifLengthOfStay                 │
│ ward → deathNotifWard                                   │
│ reporting_date → deathNotifReportingDate                │
│ comments → deathNotifComments                           │
└──────────────────────────────────────────────────────────┘
                     │
                     ▼
Step 6: Database Storage
┌──────────────────────────────────────────────────────────┐
│ Save to Notification Database:                           │
│                                                          │
│ Table: death_notifications                              │
│ - id: UUID (auto-generated)                             │
│ - patient_id: 123456                                    │
│ - death_date: 2024-02-15                                │
│ - status: PENDING                                       │
│ - created_at: 2024-02-18 09:45:23                       │
│ - created_by: user@enotif.org                           │
│ - external_event_id: (null - to be filled)              │
│                                                          │
│ Table: audit_logs                                       │
│ - Log entry: Created death notification                 │
│ - User: integration_user                                │
│ - Timestamp: 2024-02-18 09:45:23                        │
└──────────────────────────────────────────────────────────┘
                     │
                     ▼
Step 7: Queue for Async Processing
┌──────────────────────────────────────────────────────────┐
│ Add to Message Queue:                                    │
│ {                                                        │
│   "event_id": "a1b2c3d4",                               │
│   "patient_uuid": "a1b2c3d4e5f6",                       │
│   "data_elements": {...},                               │
│   "timestamp": "2024-02-18T09:45:23Z"                   │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
                     │
                     ▼
Step 8: DHIS2 API Call (Async Worker)
┌──────────────────────────────────────────────────────────┐
│ POST /api/events                                         │
│ Body: {                                                  │
│   "program": "deathNotifProgram",                        │
│   "programStage": "deathEventStage",                     │
│   "trackedEntityInstance": "a1b2c3d4e5f6",             │
│   "dataValues": [                                        │
│     {"dataElement": "deathNotifDeathDate",               │
│      "value": "2024-02-15"},                             │
│     {"dataElement": "deathNotifPlaceOfDeath",            │
│      "value": "HEALTH_FACILITY"},                        │
│     ...                                                  │
│   ]                                                      │
│ }                                                        │
│                                                          │
│ Response: {                                              │
│   "status": "OK",                                        │
│   "response": {                                          │
│     "uid": "xyz789abc",                                 │
│     "status": "persisted"                                │
│   }                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
                     │
                     ▼
Step 9: Update Tracking Database
┌──────────────────────────────────────────────────────────┐
│ Update death_notifications:                              │
│ - external_event_id: xyz789abc                           │
│ - status: SYNCED_TO_DHIS2                               │
│ - synced_at: 2024-02-18 09:46:15                        │
└──────────────────────────────────────────────────────────┘
                     │
                     ▼
Step 10: Send Webhook Callback
┌──────────────────────────────────────────────────────────┐
│ POST https://enotif.example.com/webhooks/death-sync     │
│ Signature: HMAC-SHA256                                   │
│ Body: {                                                  │
│   "event_type": "death_notification.synced",             │
│   "data": {                                              │
│     "notification_id": "a1b2c3d4",                      │
│     "dhis2_event_id": "xyz789abc",                       │
│     "status": "SYNCED_TO_DHIS2",                         │
│     "timestamp": "2024-02-18T09:46:15Z"                  │
│   }                                                      │
│ }                                                        │
└──────────────────────────────────────────────────────────┘
                     │
                     ▼
Step 11: DHIS2 Inpatient Workflow
┌──────────────────────────────────────────────────────────┐
│ DHIS2 System:                                            │
│ - Death event created in patient's record               │
│ - Visible in dashboards and reports                      │
│ - Can be verified by supervisors                         │
│ - Status: PENDING_VERIFICATION → VERIFIED               │
│ - Workflow: Additional actions triggered                 │
│   (e.g., mark patient as deceased in cervical cancer    │
│    program, create mortality report)                     │
└──────────────────────────────────────────────────────────┘
```

---

## 3. Security Architecture (OpenHIM-Provided)

```
┌─────────────────────────────────────────────────────────────┐
│         Security Layers (OpenHIM Built-In Features)         │
└─────────────────────────────────────────────────────────────┘

Layer 1: Network Security (Your Infrastructure)
┌─────────────────────────────────────────────────────────┐
│ - Firewall: Restrict traffic to OpenHIM ports (5001)    │
│ - WAF: Optional Web Application Firewall                │
│ - VPN: Encrypted tunnel between e-Notification & OpenHIM│
│ - DDoS Protection: Handled by OpenHIM rate limiting     │
└─────────────────────────────────────────────────────────┘
                         │
Layer 2: Transport Security (OpenHIM Built-In)
┌─────────────────────────────────────────────────────────┐
│ - HTTPS/TLS 1.2+: All communications encrypted          │
│ - mTLS (Mutual TLS): Client cert authentication         │
│   ✓ e-Notification provides client certificate         │
│   ✓ OpenHIM verifies certificate chain                 │
│ - Certificate Management: Configure in OpenHIM          │
│ - Forward Secrecy: TLS session settings                 │
└─────────────────────────────────────────────────────────┘
                         │
Layer 3: Authentication (OpenHIM Built-In)
┌─────────────────────────────────────────────────────────┐
│ - mTLS Certificates: Primary auth method (recommended)  │
│ - API Keys: Configure in OpenHIM Console                │
│   ✓ Each client gets unique API key                     │
│   ✓ No need to implement OAuth 2.0                      │
│ - Basic Auth: Username/password option (fallback)       │
│ - No custom JWT implementation needed                   │
└─────────────────────────────────────────────────────────┘
                         │
Layer 4: Authorization (OpenHIM Built-In RBAC)
┌─────────────────────────────────────────────────────────┐
│ - RBAC: OpenHIM native role-based access control        │
│ - Roles configured in OpenHIM Console:                  │
│   ✓ admin: Full system access                           │
│   ✓ channel-admin: Manage channels                      │
│   ✓ viewer: Read-only audit logs                        │
│ - Custom Roles: Add facility-based or user-based roles  │
│ - Permission Matrix: Granular channel-level controls    │
└─────────────────────────────────────────────────────────┘
                         │
Layer 5: Data Protection (OpenHIM + Your Database)
┌─────────────────────────────────────────────────────────┐
│ - Encryption in Transit: TLS 1.2+ (OpenHIM listener)    │
│ - Encryption at Rest: 
│   ✓ OpenHIM MongoDB: Enable encryption at rest          │
│   ✓ PostgreSQL: Use native encryption (PgCrypto)        │
│ - Field-level Encryption: Apply to PII in transformation│
│ - Key Management: Use PostgreSQL native keys            │
│ - Secrets Management: OpenHIM console for API keys      │
└─────────────────────────────────────────────────────────┘
                         │
Layer 6: Auditing (OpenHIM Built-In Transaction Logging)
┌─────────────────────────────────────────────────────────┐
│ - Comprehensive Logging: ALL requests automatically     │
│   ✓ Who: Client certificate or API key                 │
│   ✓ What: Full request + response body                 │
│   ✓ When: Timestamp for every transaction               │
│   ✓ Where: Which channel processed the request          │
│ - Audit Trail: Stored in MongoDB (queryable dashboard   │
│ - Log Retention: Configure in OpenHIM (7 years default) │
│ - Immutable Logs: Write-once MongoDB design             │
│ - Access Control: RBAC prevents unauthorized viewing    │
│ - Compliance: Satisfies healthcare audit requirements   │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Database Schema Overview

```
┌──────────────────────────────────────────────────────────┐
│         Notification Database (PostgreSQL)                │
└──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────┐
│       death_notifications           │
├─────────────────────────────────────┤
│ id (UUID, PK)                       │
│ patient_id (VARCHAR)                │
│ patient_name_encrypted (VARCHAR)    │
│ date_of_birth (DATE)                │
│ sex (VARCHAR, FK→deathSexOptions)   │
│ death_date (DATE)                   │
│ death_place (VARCHAR, FK→places)    │
│ death_facility (VARCHAR, FK→facilities) │
│ primary_cause (VARCHAR)             │
│ secondary_cause (VARCHAR)           │
│ age_at_death (INTEGER)              │
│ admission_date (DATE)               │
│ length_of_stay (INTEGER)            │
│ ward (VARCHAR)                      │
│ reporting_date (DATE)               │
│ reporter_id (VARCHAR)               │
│ reporter_comments (TEXT)            │
│ status (VARCHAR: PENDING/VERIFIED)  │
│ external_event_id (VARCHAR)         │ ◄─ DHIS2 event ID
│ external_enrollment_id (VARCHAR)    │ ◄─ DHIS2 enrollment ID
│ external_tei_id (VARCHAR)           │ ◄─ DHIS2 patient ID
│ created_at (TIMESTAMP)              │
│ created_by (VARCHAR)                │
│ updated_at (TIMESTAMP)              │
│ updated_by (VARCHAR)                │
│ verified_at (TIMESTAMP, NULL)       │
│ verified_by (VARCHAR, NULL)         │
│ verification_notes (TEXT, NULL)     │
└─────────────────────────────────────┘
         │
         │ Foreign Keys
         ├──────────────────────────────────────────┐
         │                                          │
    ┌────▼──────────────────┐    ┌────────────────▼──────┐
    │    option_sets        │    │   audit_logs          │
    ├───────────────────────┤    ├──────────────────────┤
    │ id (VARCHAR, PK)      │    │ id (UUID, PK)        │
    │ name (VARCHAR)        │    │ entity_type (VARCHAR)│
    │ values (JSONB)        │    │ entity_id (VARCHAR)  │
    │                       │    │ action (VARCHAR)     │
    │ Examples:             │    │ user_id (VARCHAR)    │
    │ - deathSexOptions     │    │ timestamp (TIMESTAMP)│
    │ - deathPlaceOptions   │    │ old_values (JSONB)   │
    │ - verificationStatus  │    │ new_values (JSONB)   │
    │ - icd10Causes         │    │ ip_address (VARCHAR) │
    │                       │    │ user_agent (VARCHAR) │
    └───────────────────────┘    └──────────────────────┘
```

---

## 5. Integration Timeline

```
Development Phase (Months 1-5)
│
├─ Month 1: Setup & Planning
│  ├─ Infrastructure setup
│  ├─ Database design & creation
│  └─ Team training on DHIS2 APIs
│
├─ Month 2-3: API Development
│  ├─ Implement 6 REST endpoints
│  ├─ Authentication & authorization
│  ├─ Input validation & error handling
│  └─ Unit & integration tests
│
├─ Month 4: DHIS2 Integration
│  ├─ Create data elements in DHIS2
│  ├─ Create program & program stage
│  ├─ Test data transformation logic
│  └─ DHIS2 integration testing
│
└─ Month 5: Security & Performance
   ├─ Security testing & fixes
   ├─ Performance optimization
   ├─ Load testing
   └─ Documentation

Testing Phase (Months 6-7)
│
├─ Unit & Integration Tests
├─ System Integration Tests
├─ Performance Tests
├─ Security Tests
└─ User Acceptance Testing (UAT)

Deployment Phase (Month 8)
│
├─ Production infrastructure setup
├─ Data migration (if applicable)
├─ User training
├─ Go-live support
└─ Post-go-live monitoring
```

---

## 6. Key Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Core Mediator** | OpenHIM | Request/response routing, auth, audit, webhooks |
| **Custom API** | Node.js/Express (minimal) | Data transformation & validation logic only |
| **Mediator Database** | MongoDB | OpenHIM transaction logs & audit trail |
| **Primary Database** | PostgreSQL | Death notification data persistence |
| **Authentication** | OpenHIM mTLS/API Keys | Replace OAuth 2.0 - built into OpenHIM |
| **Message Queue** | OpenHIM built-in | No external RabbitMQ/Kafka needed |
| **Monitoring** | OpenHIM Console + Prometheus | Mediator metrics + application health |
| **Logging** | OpenHIM built-in + ELK | Audit logs generated automatically |
| **API Documentation** | Swagger/OpenAPI | Generated from OpenHIM channels |
| **Version Control** | Git (GitHub/GitLab) | Code management |
| **CI/CD** | Jenkins/GitLab CI | Automated testing & deployment |
| **Container** | Docker | OpenHIM + custom API containerized |
| **Orchestration** | Kubernetes (optional) | Container management for scaling |

---

## 7. Deployment Topology

```
Production Environment with OpenHIM (Recommended)

┌─────────────────────────────────────────────────────────┐
│                   Data Center                            │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │           Internet (HTTPS/TLS)                     │ │
│  └────────────┬─────────────────────────────────────┘ │
│               │                                        │
│    ┌──────────▼────────────┐                          │
│    │   OpenHIM Listeners   │                          │
│    │   (TLS/mTLS Certs)    │                          │
│    │   - e-Notification    │                          │
│    │   - Admin Console     │                          │
│    └──────────┬────────────┘                          │
│               │                                        │
│    ┌──────────▼────────────────────────┐              │
│    │   OpenHIM Core (2-3 instances)    │              │
│    │   - Channel Routes                │              │
│    │   - Request Transformation        │              │
│    │   - Queue Management              │              │
│    │   - Webhook Callbacks             │              │
│    │   - RBAC & Authentication         │              │
│    └────┬─────────────────────────┬────┘              │
│         │                         │                   │
│   ┌─────▼──────┐        ┌────────▼──────┐            │
│   │  MongoDB   │        │  PostgreSQL   │            │
│   │ (Mediator) │        │ (Notifications)│            │
│   │ - OpenHIM  │        │ - Death data   │            │
│   │   txns     │        │ - Status       │            │
│   │ - Audit    │        │ - History      │            │
│   │   logs     │        └────────────────┘            │
│   └────────────┘                                      │
│         │                                             │
│    ┌────▼──────────────────────────────┐             │
│    │ Custom Transformation Service      │             │
│    │ (Node.js/Express - minimal)        │             │
│    │ - Data validation                 │             │
│    │ - ICD-10 lookup                   │             │
│    │ - Duplicate detection             │             │
│    │ (Called by OpenHIM via script)    │             │
│    └────┬───────────────────────────────┘             │
│         │                                             │
│    ┌────▼───────────────────┐                        │
│    │ Async Route to DHIS2    │                        │
│    │ (OpenHIM Built-in Queue)│                        │
│    │ - Retry logic           │                        │
│    │ - Error handling        │                        │
│    │ - Timeout: 30 seconds   │                        │
│    └────┬────────────────────┘                        │
│         │                                             │
│    ┌────▼──────────────────┐                          │
│    │ DHIS2 Inpatient       │                          │
│    │ (External system)     │                          │
│    │ - Event creation      │                          │
│    │ - Patient tracking    │                          │
│    └───────────────────────┘                          │
│                                                      │
└──────────────────────────────────────────────────────┘

Backup & Monitoring (OpenHIM Enhanced):
┌─────────────────────────────────────────────────────────┐
│ - OpenHIM Transaction Backup (MongoDB)                 │
│ - PostgreSQL backup (daily full, hourly incremental)   │
│ - Offsite backup (cloud storage)                       │
│ - OpenHIM Console: Real-time monitoring               │
│ - Prometheus: Infrastructure metrics                   │
│ - ELK Stack: Application logs + OpenHIM logs          │
│ - Alerting: Email, Slack for critical issues         │
└─────────────────────────────────────────────────────────┘
```

---

## Summary

This architecture provides:
- **Scalability**: Load balancing, auto-scaling capability
- **Reliability**: Redundancy, failover, backup/recovery
- **Security**: Multi-layer protection, encryption, audit trail
- **Performance**: Caching, async processing, indexed database
- **Maintainability**: Separated concerns, documented APIs, monitoring
- **Compliance**: Audit logging, encryption, data governance

