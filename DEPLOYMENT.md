# Deployment Instructions

## Docker Deployment
1. Build Docker image:
   ```
   docker build -t dhis2-api-server .
   ```
2. Run container:
   ```
   docker run -d -p 443:443 --name dhis2-api-server dhis2-api-server
   ```

## VM/Server Deployment
1. Set up Linux VM (Ubuntu recommended)
2. Install required software (Node.js, Python, etc.)
3. Clone repository and install dependencies
4. Configure environment variables (see ENVIRONMENT.md)
5. Start API server

## Security & Backup
- Enable HTTPS/TLS
- Restrict access to trusted IPs
- Schedule regular backups of database and logs
