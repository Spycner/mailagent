# Google Cloud Setup for Mailagent

This document outlines the Google Cloud Platform setup required for the mailagent project, including what can be automated with Terraform and what requires manual configuration.

## Overview

The mailagent project requires several Google Cloud services:
- **Gmail API** for fetching emails
- **Cloud Storage** for file storage and backups
- **Service Accounts** for authentication
- **OAuth2 configuration** for secure API access
- **Optional**: Cloud Run for deployment, Cloud SQL for database

## Architecture Decision: Terraform vs Manual

| Component | Automation Level | Notes |
|-----------|------------------|-------|
| GCP Project Creation | ✅ Terraform | Fully automated |
| API Enablement | ✅ Terraform | Gmail API, Storage API, etc. |
| Service Accounts | ✅ Terraform | Creation and IAM roles |
| OAuth2 Consent Screen | ⚠️ Partial | Basic setup only |
| OAuth2 Client Credentials | ❌ Manual | Security requirement |
| Domain Verification | ❌ Manual | If using custom domain |
| Initial Token Generation | ❌ Manual | First-time auth flow |

## Terraform Configuration

### Project Structure
```
terraform/
├── main.tf              # Main infrastructure
├── variables.tf         # Input variables
├── outputs.tf          # Output values
├── terraform.tfvars    # Variable values (gitignored)
└── versions.tf         # Provider versions
```

### Key Resources Managed by Terraform

1. **Google Cloud Project**
   ```hcl
   resource "google_project" "mailagent" {
     name       = "mailagent-${random_id.project_suffix.hex}"
     project_id = "mailagent-${random_id.project_suffix.hex}"
   }
   ```

2. **API Services**
   ```hcl
   resource "google_project_service" "apis" {
     for_each = toset([
       "gmail.googleapis.com",
       "storage.googleapis.com",
       "secretmanager.googleapis.com",
       "run.googleapis.com"
     ])
     service = each.value
   }
   ```

3. **Service Account**
   ```hcl
   resource "google_service_account" "mailagent" {
     account_id   = "mailagent-service"
     display_name = "Mailagent Service Account"
   }
   ```

4. **Storage Resources**
   ```hcl
   resource "google_storage_bucket" "data" {
     name     = "${var.project_id}-mailagent-data"
     location = var.region
   }
   ```

## Manual Configuration Requirements

### 1. OAuth2 Consent Screen
**Why Manual**: Google requires human verification for security-sensitive configurations.

**Steps**:
1. Navigate to Google Cloud Console → APIs & Services → OAuth consent screen
2. Choose "External" user type (for Gmail access)
3. Fill in application information:
   - App name: "Mailagent"
   - User support email: Your email
   - Developer contact: Your email
4. Add scopes:
   - `https://www.googleapis.com/auth/gmail.readonly`
   - `https://www.googleapis.com/auth/gmail.modify` (if needed)
5. Add test users (your Gmail account)

### 2. OAuth2 Client Credentials
**Why Manual**: Credential creation requires interactive consent.

**Steps**:
1. Navigate to APIs & Services → Credentials
2. Click "Create Credentials" → "OAuth 2.0 Client IDs"
3. Choose "Desktop application" type
4. Name: "Mailagent Desktop Client"
5. Download the JSON file
6. Store securely (never commit to git)

### 3. Initial Authentication Flow
**Why Manual**: First-time user consent required.

**Process**:
1. Run authentication script with downloaded credentials
2. Browser opens for Google login
3. Grant permissions to your application
4. Receive and store refresh token
5. Use refresh token for automated access

## Security Considerations

### Credential Management
- **Service Account Keys**: Store in Google Secret Manager
- **OAuth2 Credentials**: Never commit to version control
- **Refresh Tokens**: Encrypt and store securely
- **Environment Variables**: Use for local development only

### IAM Best Practices
- **Principle of Least Privilege**: Grant minimal required permissions
- **Service Account Rotation**: Regular key rotation
- **Audit Logging**: Enable Cloud Audit Logs
- **Resource-level Permissions**: Scope access to specific resources

## Development vs Production

### Development Setup
- Use personal Google account for testing
- Store credentials locally in `.env` file
- Use SQLite for simplicity
- Single service account for all operations

### Production Setup
- Dedicated Google Workspace or Cloud Identity
- Credentials in Secret Manager
- Cloud SQL for database
- Separate service accounts per function
- Cloud Run for hosting

## Cost Considerations

### Free Tier Resources
- Gmail API: 1 billion quota units/day (generous for most use cases)
- Cloud Storage: 5GB free
- Cloud Run: 2 million requests/month
- Secret Manager: 6 active secrets free

### Estimated Monthly Costs (Low Usage)
- Gmail API: $0 (within free tier)
- Cloud Storage: $0.02-0.05
- Cloud Run: $0 (within free tier)
- **Total**: < $1/month for development

## Troubleshooting Common Issues

### OAuth2 Setup
- **Error**: "Access blocked" → Add your email as test user
- **Error**: "Invalid client" → Check client ID configuration
- **Error**: "Scope not authorized" → Verify consent screen scopes

### API Quotas
- **Gmail API limits**: 250 quota units/user/second
- **Daily limits**: Monitor in Cloud Console
- **Rate limiting**: Implement exponential backoff

### Service Account Issues
- **Permission denied**: Check IAM roles
- **Key not found**: Verify service account key path
- **Token expired**: Implement automatic refresh

## Next Steps After Setup

1. **Test Gmail API Access**: Verify you can fetch emails
2. **Implement Error Handling**: Robust retry logic
3. **Set Up Monitoring**: Cloud Logging and Monitoring
4. **Configure Alerts**: Quota usage and error rates
5. **Plan Scaling**: Multi-user support considerations

## References

- [Gmail API Documentation](https://developers.google.com/gmail/api)
- [Google Cloud IAM Best Practices](https://cloud.google.com/iam/docs/using-iam-securely)
- [OAuth 2.0 for Desktop Apps](https://developers.google.com/identity/protocols/oauth2/native-app)
- [Terraform Google Provider](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
