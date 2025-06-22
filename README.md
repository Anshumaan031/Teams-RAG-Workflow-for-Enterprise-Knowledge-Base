# Teams RAG Workflow for Enterprise Knowledge Base

This project implements a Microsoft Teams RAG workflow that processes user queries from Teams messages, sends them to a FastAPI RAG backend, and returns responses sourced from an enterprise knowledge base stored in a PostgreSQL table.

![alt text](https://github.com/Anshumaan031/Teams-RAG-Workflow-for-Enterprise-Knowledge-Base/blob/main/photo.png)

## Prerequisites

- n8n instance with public URL access
- PostgreSQL database
- Microsoft Teams admin access
- Azure Active Directory admin access
- Running FastAPI RAG service

## Required Credentials

### 1. PostgreSQL Database Credentials
**Used for:** Storing conversation history and user data

**Required Information:**
- Host/Server address
- Database name
- Username
- Password
- Port (usually 5432)

**How to Acquire:**
- **Cloud Options:**
  - AWS RDS PostgreSQL
  - Google Cloud SQL PostgreSQL
  - Azure Database for PostgreSQL
  - PostgreSQL (free tier available)
  - ElephantSQL (free tier available)
- **Self-hosted:** Install PostgreSQL on your server

**Database Schema Required:**
```sql
CREATE TABLE user_conversations (
    id SERIAL PRIMARY KEY,
    user_id VARCHAR(255) NOT NULL,
    channel_id VARCHAR(255) NOT NULL,
    conversation_history JSONB,
    last_message_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(user_id, channel_id)
);
```

### 2. Microsoft Teams Credentials
**Used for:** Receiving webhooks from Teams and sending responses back

**Required Information:**
- App ID (Client ID)
- App Secret (Client Secret)
- Access Token (automatically managed by n8n)

## Installation Steps

### Step 1: Database Setup

1. **Create PostgreSQL Database**
   ```sql
   CREATE DATABASE teams_rag_bot;
   ```

2. **Create Required Table**
   ```sql
   CREATE TABLE user_conversations (
       id SERIAL PRIMARY KEY,
       user_id VARCHAR(255) NOT NULL,
       channel_id VARCHAR(255) NOT NULL,
       conversation_history JSONB,
       last_message_at TIMESTAMP,
       created_at TIMESTAMP DEFAULT NOW(),
       updated_at TIMESTAMP DEFAULT NOW(),
       UNIQUE(user_id, channel_id)
   );
   ```

### Step 2: Azure AD App Registration
1. **Register Application**
   - Go to [Azure Portal](https://portal.azure.com)
   - Navigate to **Azure Active Directory** > **App registrations**
   - Click **New registration**
   - Fill in:
     - **Name:** Your bot name (e.g., "RAG Teams Bot")
     - **Supported account types:** Accounts in this organizational directory only
     - **Redirect URI:** Leave blank for now
   - Click **Register**
   - **Save the Application (client) ID**

2. **Configure API Permissions**
   - In your registered app, go to **API permissions**
   - Click **Add a permission**
   - Select **Microsoft Graph**
   - Choose **Application permissions**
   - Add these permissions:
     - `Chat.Read.All`
     - `ChatMessage.Send`
     - `Team.ReadBasic.All`
     - `Channel.ReadBasic.All`
   - Click **Grant admin consent**

3. **Create Client Secret**
   - Go to **Certificates & secrets**
   - Click **New client secret**
   - Add description and choose expiration
   - **Copy the Value (this is your Client Secret)**
   - Store securely - you won't see it again

### Step 3: Teams App Configuration
1. **Create App Manifest**
   - Create a `manifest.json` file with the following content:
   ```json
   {
       "$schema": "https://developer.microsoft.com/en-us/json-schemas/teams/v1.16/MicrosoftTeams.schema.json",
       "manifestVersion": "1.16",
       "version": "1.0.0",
       "id": "YOUR_APP_ID_HERE",
       "packageName": "com.yourcompany.ragbot",
       "developer": {
           "name": "Your Company",
           "websiteUrl": "https://yourwebsite.com",
           "privacyUrl": "https://yourwebsite.com/privacy",
           "termsOfUseUrl": "https://yourwebsite.com/terms"
       },
       "icons": {
           "color": "color.png",
           "outline": "outline.png"
       },
       "name": {
           "short": "RAG Bot",
           "full": "RAG Knowledge Bot"
       },
       "description": {
           "short": "AI-powered knowledge assistant",
           "full": "An intelligent bot that answers questions using RAG technology"
       },
       "accentColor": "#FFFFFF",
       "bots": [
           {
               "botId": "YOUR_APP_ID_HERE",
               "scopes": [
                   "personal",
                   "team",
                   "groupchat"
               ],
               "supportsFiles": false,
               "isNotificationOnly": false
           }
       ],
       "permissions": [
           "identity",
           "messageTeamMembers"
       ],
       "validDomains": [
           "yourn8ndomain.com"
       ]
   }
   ```
   - Replace `YOUR_APP_ID_HERE` with your Azure app's Application ID
   - Replace `yourn8ndomain.com` with your n8n instance domain

2. **Create App Icons**
   - Create `color.png` (192x192px) and `outline.png` (32x32px) icons
   - Place in same directory as manifest.json

3. **Upload to Teams**
   - Zip the manifest.json and icon files
   - Go to [Microsoft Teams Admin Center](https://admin.teams.microsoft.com/)
   - Navigate to **Teams apps** > **Manage apps**
   - Click **Upload** > **Upload a custom app**
   - Upload your zip file

### Step 4: n8n Configuration

### 3. FastAPI RAG Service
**Your workflow calls:** `http://localhost:8000/query`

**Requirements:**
- A running FastAPI service on port 8000
- The service should accept POST requests with this structure:
```json
{
    "query": "user question",
    "user_id": "teams_user_id",
    "conversation_history": [],
    "metadata": {
        "channel_id": "teams_channel_id",
        "message_id": "teams_message_id",
        "timestamp": "2024-01-01T00:00:00Z",
        "user_name": "User Name"
    }
}
```

- The service should respond with:
```json
{
    "response": "AI response text",
    "sources": ["source1", "source2"]
}
```

### 4. n8n Webhook Configuration
**Required:**
- Public URL for your n8n instance
- Webhook endpoint: `https://your-n8n-domain.com/webhook/teams-webhook`

**Setup Steps:**
1. In n8n, configure the webhook node
2. Copy the webhook URL
3. In your Teams app configuration, set the messaging endpoint to your webhook URL

1. **Import Workflow**
   - Import the `teams_rag_workflow.json` file into your n8n instance

2. **Configure PostgreSQL Credentials**
   - Click on any PostgreSQL node in the workflow
   - Click **Add new credential**
   - Enter your database connection details:
     - Host: Your PostgreSQL server address
     - Database: `teams_rag_bot` (or your database name)
     - User: Your database username
     - Password: Your database password
     - Port: 5432 (default)
   - Test the connection
   - Save as "PostgreSQL Credentials"

3. **Configure Microsoft Teams Credentials**
   - Click on the Teams response node
   - Click **Add new credential**
   - Select **OAuth2**
   - Enter:
     - **Client ID:** Your Azure app's Application ID
     - **Client Secret:** The secret you created in Azure
     - **Auth URL:** `https://login.microsoftonline.com/common/oauth2/v2.0/authorize`
     - **Token URL:** `https://login.microsoftonline.com/common/oauth2/v2.0/token`
     - **Scope:** `https://graph.microsoft.com/.default`
   - Complete OAuth flow
   - Save as "Microsoft Teams Credentials"

4. **Configure Webhook**
   - Activate the workflow
   - Copy the webhook URL from the Teams Webhook node
   - Format: `https://your-n8n-domain.com/webhook/teams-webhook`

### Step 5: Bot Endpoint Configuration

1. **Set Messaging Endpoint**
   - Go back to Azure Portal
   - Navigate to your app registration
   - Go to **Bot management** > **Configuration**
   - Set **Messaging endpoint** to your n8n webhook URL
   - Save changes

### Step 6: FastAPI Service Setup

### Pre-deployment Checklist

- [ ] PostgreSQL database is running and accessible
- [ ] Database table `user_conversations` is created
- [ ] Azure AD app is registered with correct permissions
- [ ] Client secret is generated and stored securely
- [ ] Teams app manifest is created and uploaded
- [ ] n8n workflow is imported and configured
- [ ] PostgreSQL credentials are configured in n8n
- [ ] Teams credentials are configured in n8n
- [ ] Webhook URL is set in Azure bot configuration
- [ ] FastAPI RAG service is running on port 8000
- [ ] n8n instance has public URL access

### Testing Steps

1. **Test Database Connection**
   - Execute workflow manually
   - Check PostgreSQL logs for connection attempts

2. **Test Teams Integration**
   - Add the bot to a Teams channel
   - Send a test message mentioning the bot
   - Check n8n execution logs

3. **Test FastAPI Service**
   - Send direct HTTP request to `http://localhost:8000/query`
   - Verify response format matches expected structure

4. **End-to-End Test**
   - Send message in Teams
   - Verify workflow execution in n8n
   - Confirm response appears in Teams
   - Check conversation history is stored in database

## Environment Variables

Create a `.env` file for sensitive information:

```env
# Database
DB_HOST=your-postgres-host
DB_NAME=teams_rag_bot
DB_USER=your-db-user
DB_PASSWORD=your-db-password
DB_PORT=5432

# Azure/Teams
AZURE_CLIENT_ID=your-azure-app-id
AZURE_CLIENT_SECRET=your-azure-client-secret
TEAMS_APP_ID=your-teams-app-id

# n8n
N8N_WEBHOOK_URL=https://your-n8n-domain.com/webhook/teams-webhook

# FastAPI
FASTAPI_URL=http://localhost:8000
```

## Common Issues & Solutions

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Webhook not receiving messages | Incorrect messaging endpoint | Verify webhook URL in Azure bot configuration |
| Database connection failed | Wrong credentials/host | Check PostgreSQL connection details |
| Teams auth failures | Missing permissions | Verify app permissions and admin consent |
| FastAPI connection timeout | Service not running | Ensure FastAPI service is running on port 8000 |
| Bot not responding | Workflow not activated | Activate workflow in n8n |
| "Bot not found" error | Teams app not installed | Install Teams app in target channels |

## Security Best Practices

- Store all credentials securely using environment variables
- Use HTTPS for all webhook endpoints
- Implement proper error handling and logging
- Regularly rotate client secrets (Azure recommends every 6 months)
- Monitor webhook endpoints for unusual activity
- Implement rate limiting on FastAPI endpoints
- Use Azure Key Vault for production credential storage

## Monitoring & Maintenance

### Logs to Monitor
- n8n workflow execution logs
- PostgreSQL query logs
- FastAPI service logs
- Azure AD authentication logs

### Regular Maintenance
- Monitor database size and clean old conversations
- Update client secrets before expiration
- Review and update API permissions as needed
- Monitor webhook endpoint performance
- Update Teams app manifest when needed

## Support & Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Microsoft Teams Developer Documentation](https://docs.microsoft.com/en-us/microsoftteams/platform/)
- [Azure Active Directory Documentation](https://docs.microsoft.com/en-us/azure/active-directory/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

---

**Note:** This setup requires administrative privileges in both Azure AD and Microsoft Teams. Ensure you have the necessary permissions before starting the configuration process.
