# e-Notification-to-DHIS2-Integration

## Overview
This project provides an integration layer for receiving death notification data from the national e-Notification system and storing it in DHIS2 (District Health Information System 2). The integration is one-way: e-Notification pushes data to DHIS2 via a secure REST API.

## Key Features
- **RESTful API** for receiving death notifications in JSON format
- **OAuth 2.0 / Basic Auth** authentication (recommended: OAuth 2.0)
- **Data validation** to ensure DHIS2 data integrity
- **Verification and rejection** workflow for DHIS2 admins
- **Optional webhooks** for status callbacks to e-Notification
- **Docker-ready** for on-premises or cloud deployment

## Data Flow
1. e-Notification system sends a POST request with death notification data to the DHIS2 integration API.
2. The API validates and stores the data in DHIS2.
3. DHIS2 users can view, verify, or reject notifications as needed.
4. (Optional) DHIS2 can send status updates back to e-Notification via webhooks if configured.

## Important Notes
- **No data is sent from DHIS2 to e-Notification** except for optional webhook status updates.
- **Verification and deletion actions** in DHIS2 do not affect records in the e-Notification system; they only manage DHIS2’s own copy.
- **Data validation** is enforced on all incoming records, even if the source is trusted, to maintain DHIS2 data quality.

## Documentation
- See `API_SPECIFICATION.md` for full API details, endpoints, and data formats.
- See `AUTHENTICATION.md` for authentication setup.
- See `DEPLOYMENT.md` for deployment instructions.
- See `ENVIRONMENT.md` for environment variable requirements.

## Support
For questions or support, see `SUPPORT_CONTACTS.md` or contact the DHIS2 admin team.