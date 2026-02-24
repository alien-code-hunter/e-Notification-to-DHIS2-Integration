# Authentication Instructions

## OAuth 2.0 (Recommended)

- **Token Endpoint:** 
**All these details will be provided in due corse**
  https://sandbox-dhis2.example.com/api/oauth2/token
- **Grant Type:** `client_credentials`
- **Required Parameters:**
  - `client_id`: [provided by DHIS2 team]
  - `client_secret`: [provided by DHIS2 team]
  - `grant_type`: `client_credentials`
  - `scope`: (optional, e.g., `death-notification.write`)

**Example Token Request:**
```bash
curl -X POST https://sandbox-dhis2.example.com/api/oauth2/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials&client_id=YOUR_CLIENT_ID&client_secret=YOUR_CLIENT_SECRET"
```

**Example Authorization Header:**
```http
Authorization: Bearer [access_token]
```

---

## Basic Authentication (Testing Only)

- **Username/Password:** [provided by DHIS2 team]
- **Header Example:**
  ```http
  Authorization: Basic base64(username:password)
  ```
- **Example using curl:**
  ```bash
  curl -u testuser:testpass https://sandbox-dhis2.example.com/api/v1/death-notifications
  ```
