# Death Notification REST API Specification
## e-Notification to DHIS2 Integration

### API Overview
This API allows the e-Notification system to push death notification data to the DHIS2 Inpatient system. The API follows RESTful principles and uses JSON for data exchange.

---

## 1. Authentication

### Method: OAuth 2.0 / Basic Authentication

**Recommended: OAuth 2.0**
- Endpoint: `https://[dhis2-domain]/api/oauth2/token`
- Grant Type: `client_credentials`
- Headers:
  ```
  Authorization: Bearer [access_token]
  Content-Type: application/json
  ```

**Alternative: Basic Authentication**
```
Authorization: Basic base64(username:password)
```

---

## 2. Base URL
```
https://[dhis2-domain]/api/v1/death-notifications
```

---

## 3. API Endpoints

### 3.1 POST - Create Death Notification
**Endpoint:** `POST /death-notifications`

**Description:** Submit a new death notification from e-Notification system

**Request Headers:**
```json
{
  "Content-Type": "application/json",
  "Authorization": "Bearer [access_token]"
}
```

**Request Body:**
```json
{
  "externalId": "EN-2024-001-12345",
  "patient": {
    "firstName": "John",
    "lastName": "Doe",
    "nationalId": "1234567890",
    "dhis2PatientId": "ABC123DEF456",
    "dateOfBirth": "1980-05-15",
    "sex": "M"
  },
  "deathDetails": {
    "dateOfDeath": "2024-02-10",
    "ageAtDeath": 44,
    "placeOfDeath": "Health Facility",
    "healthFacility": {
      "facilityCode": "HF001",
      "facilityName": "Central Hospital",
      "county": "Nairobi"
    }
  },
  "medicalInfo": {
    "primaryCauseOfDeath": {
      "icd10Code": "I21.9",
      "description": "ST elevation (STEMI) myocardial infarction of unspecified site"
    },
    "secondaryCausesOfDeath": [
      {
        "icd10Code": "E14.9",
        "description": "Type 2 diabetes mellitus without complications"
      },
      {
        "icd10Code": "I10",
        "description": "Essential (primary) hypertension"
      }
    ],
    "clinicalDiagnosis": "Acute myocardial infarction with hypertension and diabetes",
    "autopsyPerformed": false
  },
  "admissionDetails": {
    "dateOfAdmission": "2024-02-08",
    "lengthOfStayDays": 2,
    "wardDepartment": "Cardiac Unit"
  },
  "reportingDetails": {
    "reportingDate": "2024-02-11",
    "reportedBy": "Dr. Jane Smith",
    "reportingFacility": "Central Hospital",
    "verificationStatus": "pending"
  },
  "additionalInfo": {
    "comments": "Patient admitted with chest pain, underwent ECG and troponin tests. STEMI confirmed. Patient died 2 days after admission.",
    "phoneNumber": "+254711234567",
    "nextOfKin": "Mary Doe"
  }
}
```

**Response (Success - 201 Created):**
```json
{
  "status": "success",
  "message": "Death notification created successfully",
  "data": {
    "id": "deathNotif_abc123xyz789",
    "externalId": "EN-2024-001-12345",
    "eventId": "event_xyz123",
    "enrollmentId": "enrollment_xyz123",
    "trackedEntityId": "te_xyz123",
    "trackedEntityType": "Person",
    "createdAt": "2024-02-11T10:30:45Z",
    "status": "pending_verification",
    "link": "https://[dhis2-domain]/api/v1/death-notifications/deathNotif_abc123xyz789"
  }
}
```

**Response (Error - 400 Bad Request):**
```json
{
  "status": "error",
  "statusCode": 400,
  "message": "Validation failed",
  "errors": [
    {
      "field": "patient.dateOfBirth",
      "message": "Date format invalid. Use 'YYYY-MM-DD'"
    },
    {
      "field": "deathDetails.dateOfDeath",
      "message": "Date of death cannot be in the future"
    }
  ]
}
```

---

### 3.2 GET - Retrieve Death Notification
**Endpoint:** `GET /death-notifications/{id}`

**Description:** Retrieve a specific death notification record

**Response (Success - 200 OK):**
```json
{
  "status": "success",
  "data": {
    "id": "deathNotif_abc123xyz789",
    "externalId": "EN-2024-001-12345",
    "patient": {
      "firstName": "John",
      "lastName": "Doe",
      "nationalId": "1234567890",
      "dhis2PatientId": "ABC123DEF456",
      "dateOfBirth": "1980-05-15",
      "sex": "M"
    },
    "deathDetails": {
      "dateOfDeath": "2024-02-10",
      "ageAtDeath": 44,
      "placeOfDeath": "Health Facility",
      "healthFacility": {
        "facilityCode": "HF001",
        "facilityName": "Central Hospital",
        "county": "Nairobi"
      }
    },
    "medicalInfo": {
      "primaryCauseOfDeath": {
        "icd10Code": "I21.9",
        "description": "ST elevation (STEMI) myocardial infarction of unspecified site"
      },
      "secondaryCausesOfDeath": [
        {
          "icd10Code": "E14.9",
          "description": "Type 2 diabetes mellitus without complications"
        }
      ],
      "clinicalDiagnosis": "Acute myocardial infarction with hypertension and diabetes",
      "autopsyPerformed": false
    },
    "reportingDetails": {
      "reportingDate": "2024-02-11",
      "reportedBy": "Dr. Jane Smith",
      "verificationStatus": "verified",
      "verifiedBy": "Dr. Admin User",
      "verificationDate": "2024-02-12"
    },
    "dhis2TrackerData": {
      "eventId": "event_xyz123",
      "enrollmentId": "enrollment_xyz123",
      "trackedEntityId": "te_xyz123"
    },
    "createdAt": "2024-02-11T10:30:45Z",
    "lastModifiedAt": "2024-02-12T14:22:10Z"
  }
}
```

---

### 3.3 PUT - Update Death Notification
**Endpoint:** `PUT /death-notifications/{id}`

**Description:** Update an existing death notification (only pending verifications)

**Request Body:** (Same structure as POST)

**Response (Success - 200 OK):**
```json
{
  "status": "success",
  "message": "Death notification updated successfully",
  "data": {
    "id": "deathNotif_abc123xyz789",
    "lastModifiedAt": "2024-02-12T10:30:45Z"
  }
}
```

---

### 3.4 GET - List Death Notifications (with Filtering)
**Endpoint:** `GET /death-notifications?[filters]`

**Query Parameters:**
- `startDate` (YYYY-MM-DD) - Filter by death date range start
- `endDate` (YYYY-MM-DD) - Filter by death date range end
- `facilityCod` (string) - Filter by health facility code
- `cause` (string) - Filter by primary cause of death (ICD-10)
- `verificationStatus` (pending|verified|rejected)
- `limit` (integer, default: 50) - Number of records to return
- `offset` (integer, default: 0) - Pagination offset

**Example:**
```
GET /death-notifications?startDate=2024-02-01&endDate=2024-02-28&verificationStatus=pending&limit=20
```

**Response (Success - 200 OK):**
```json
{
  "status": "success",
  "data": {
    "total": 150,
    "returned": 20,
    "offset": 0,
    "notifications": [
      {
        "id": "deathNotif_abc123xyz789",
        "externalId": "EN-2024-001-12345",
        "patientName": "John Doe",
        "dateOfDeath": "2024-02-10",
        "primaryCause": "I21.9",
        "facilitieCode": "HF001",
        "verificationStatus": "pending",
        "createdAt": "2024-02-11T10:30:45Z"
      }
    ]
  }
}
```

---

### 3.5 PATCH - Verify Death Notification
**Endpoint:** `PATCH /death-notifications/{id}/verify`

**Description:** Verify a death notification (admin/supervisor only)

**Request Body:**
```json
{
  "verificationStatus": "verified",
  "verifiedBy": "Dr. Admin User",
  "verificationDate": "2024-02-12",
  "verificationNotes": "All details verified and confirmed"
}
```

**Response (Success - 200 OK):**
```json
{
  "status": "success",
  "message": "Death notification verified successfully",
  "data": {
    "id": "deathNotif_abc123xyz789",
    "verificationStatus": "verified",
    "verifiedAt": "2024-02-12T14:22:10Z"
  }
}
```

---

### 3.6 DELETE - Reject/Delete Death Notification
**Endpoint:** `DELETE /death-notifications/{id}`

**Description:** Reject or delete a death notification (only if not verified)

**Request Body:**
```json
{
  "reason": "Duplicate entry",
  "rejectionNotes": "This death was already reported on 2024-02-11"
}
```

**Response (Success - 200 OK):**
```json
{
  "status": "success",
  "message": "Death notification deleted/rejected successfully",
  "data": {
    "id": "deathNotif_abc123xyz789",
    "status": "rejected",
    "deletedAt": "2024-02-12T15:30:00Z"
  }
}
```

---

## 4. Data Validation Rules

### Patient Information
- `firstName` and `lastName` - Required, max 100 characters each
- `nationalId` - Format validation (must match country format)
- `dateOfBirth` - Required, must be in YYYY-MM-DD format, not future date
- `sex` - Required, must be one of: M, F, U
- `dhis2PatientId` - Optional, if provided must match existing patient in DHIS2

### Death Details
- `dateOfDeath` - Required, YYYY-MM-DD format, cannot be future date
- `ageAtDeath` - Optional but recommended, must be 0-150
- `placeOfDeath` - Required, must be one of predefined options
- `healthFacility.facilityCode` - Must match existing facility code in DHIS2

### Medical Information
- `primaryCauseOfDeath` - Required, must be valid ICD-10 code
- `secondaryCausesOfDeath` - Optional array of ICD-10 codes
- `clinicalDiagnosis` - Recommended, max 500 characters
- `autopsyPerformed` - Boolean, optional

### Reporting Details
- `reportingDate` - Required, YYYY-MM-DD format
- `reportedBy` - Required, max 100 characters
- `verificationStatus` - Must be one of: pending, verified, rejected

---

## 5. Error Handling

### Standard Error Response Format
```json
{
  "status": "error",
  "statusCode": 400,
  "message": "Human-readable error message",
  "timestamp": "2024-02-11T10:30:45Z",
  "path": "/api/v1/death-notifications",
  "errors": [
    {
      "field": "fieldName",
      "message": "Specific validation error"
    }
  ]
}
```

### Common HTTP Status Codes
| Status | Meaning | Example |
|--------|---------|---------|
| 200 | OK | Successful GET/PUT |
| 201 | Created | Successful POST |
| 400 | Bad Request | Validation error |
| 401 | Unauthorized | Missing/invalid authentication |
| 403 | Forbidden | Permission denied |
| 404 | Not Found | Record not found |
| 409 | Conflict | Duplicate external ID |
| 500 | Server Error | Internal DHIS2 error |

---

## 6. Webhook/Callback (Optional)

The e-Notification system can register for callbacks to receive verification status updates.

**Register Webhook:**
```
POST /death-notifications/webhooks
```

**Payload:**
```json
{
  "callbackUrl": "https://[e-notification-domain]/api/webhooks/death-verification",
  "events": ["verified", "rejected"],
  "secret": "[webhook-secret-key]"
}
```

**Webhook Event Payload:**
```json
{
  "event": "death_verified",
  "deathNotificationId": "deathNotif_abc123xyz789",
  "externalId": "EN-2024-001-12345",
  "verificationStatus": "verified",
  "verificationDate": "2024-02-12T14:22:10Z",
  "dhis2EventId": "event_xyz123",
  "timestamp": "2024-02-12T14:22:10Z"
}
```

---

## 7. Rate Limiting

- **Rate Limit:** 1000 requests per hour per API key
- **Headers:**
  ```
  X-RateLimit-Limit: 1000
  X-RateLimit-Remaining: 999
  X-RateLimit-Reset: [unix-timestamp]
  ```

---

## 8. Data Mapping to DHIS2

The API will automatically map incoming e-Notification data to DHIS2 Tracker entities:

| e-Notification Field | DHIS2 Data Element | Notes |
|---------------------|-------------------|-------|
| patient.firstName + lastName | Tracked Entity Attribute (Name) | Creates/updates patient record |
| patient.dateOfBirth | Tracked Entity Attribute (DOB) | |
| patient.sex | Tracked Entity Attribute (Sex) | |
| patient.nationalId | Tracked Entity Attribute (ID) | Unique identifier |
| deathDetails.dateOfDeath | deathNotifDateDeath | Event date in DHIS2 |
| deathDetails.ageAtDeath | deathNotifAge | |
| deathDetails.placeOfDeath | deathNotifPlaceDeath | |
| medicalInfo.primaryCauseOfDeath | deathNotifPrimaryCOD | |
| medicalInfo.secondaryCausesOfDeath | deathNotifSecondaryCOD | |
| admissionDetails.dateOfAdmission | deathNotifAdmitDate | |
| reportingDetails.reportedBy | deathNotifReportedBy | |
| reportingDetails.verificationStatus | deathNotifVerfStatus | |

---

## 9. Implementation Example (cURL)

### Creating a Death Notification
```bash
curl -X POST https://dhis2-domain/api/v1/death-notifications \
  -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "externalId": "EN-2024-001-12345",
    "patient": {
      "firstName": "John",
      "lastName": "Doe",
      "dateOfBirth": "1980-05-15",
      "sex": "M"
    },
    "deathDetails": {
      "dateOfDeath": "2024-02-10",
      "ageAtDeath": 44,
      "placeOfDeath": "Health Facility"
    },
    "medicalInfo": {
      "primaryCauseOfDeath": {
        "icd10Code": "I21.9",
        "description": "Myocardial infarction"
      }
    },
    "reportingDetails": {
      "reportingDate": "2024-02-11",
      "reportedBy": "Dr. Jane Smith",
      "verificationStatus": "pending"
    }
  }'
```

---

## 10. Security Considerations

1. **HTTPS Only** - All API calls must use HTTPS/TLS encryption
2. **Authentication** - Use OAuth 2.0 with certificate-based validation
3. **Data Validation** - All inputs are validated server-side
4. **Access Control** - Role-based permissions for verification
5. **Audit Logging** - All API calls logged with timestamp and user ID
6. **PII Protection** - Death notifications encrypted at rest
7. **API Key Rotation** - Implement key rotation policies
8. **IP Whitelisting** - Restrict API access to known e-Notification system IPs

---

## 11. Testing Endpoints

### Sandbox Environment
```
https://sandbox-dhis2.example.com/api/v1/death-notifications
```

### Testing with Mock Data
All endpoints support a `test=true` query parameter to simulate responses without persisting data.

```
POST /death-notifications?test=true
```

---

## 12. Documentation and Support

- **API Documentation:** https://dhis2-domain/api/docs
- **OpenAPI Specification:** https://dhis2-domain/api/v1/death-notifications/openapi.json
- **Support Email:** dhis2-admin@example.com
- **Webhook Testing Tool:** https://webhook.site (for development)

