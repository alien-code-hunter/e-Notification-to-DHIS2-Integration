# Environment Variables

List and describe all required environment variables for running the API server:

- `DB_CONNECTION_STRING`: Database connection string for storing notifications.
- `OAUTH_CLIENT_ID`: OAuth 2.0 client ID for authentication.
- `OAUTH_CLIENT_SECRET`: OAuth 2.0 client secret for authentication.
- `OAUTH_TOKEN_URL`: OAuth 2.0 token endpoint URL.
- `API_KEY`: API key for rate limiting and access control.
- `WEBHOOK_SECRET`: Secret key for webhook validation.
- `LOG_LEVEL`: Logging verbosity (info, debug, error).

Create a `.env.example` file for reference:

```
DB_CONNECTION_STRING=your-db-connection-string
OAUTH_CLIENT_ID=your-client-id
OAUTH_CLIENT_SECRET=your-client-secret
OAUTH_TOKEN_URL=https://sandbox-dhis2.example.com/api/oauth2/token
API_KEY=your-api-key
WEBHOOK_SECRET=your-webhook-secret
LOG_LEVEL=info
```
