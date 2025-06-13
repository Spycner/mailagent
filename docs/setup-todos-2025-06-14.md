# Setup Todos - June 14, 2025

**Project**: Mailagent Google Cloud Setup
**Date**: Friday, June 14, 2025
**Estimated Time**: 4-6 hours

## Pre-Setup Checklist ✅

- [ ] Ensure you have a Google account with billing enabled
- [ ] Install Terraform locally (`terraform --version`)
- [ ] Install Google Cloud CLI (`gcloud --version`)
- [ ] Have your domain ready (if using custom domain)
- [ ] Review the `docs/google-cloud-setup.md` documentation

## Phase 1: Initial Google Cloud Setup (30-45 minutes)

### 1.1 Google Cloud Console Setup
- [ ] **Login to Google Cloud Console** (console.cloud.google.com)
- [ ] **Accept Terms of Service** if first time
- [ ] **Enable billing** on your Google account
  - Navigate to Billing → Link a billing account
  - Add payment method (required even for free tier)
- [ ] **Note your billing account ID** for Terraform

### 1.2 Local Environment Setup
- [ ] **Authenticate gcloud CLI**:
  ```bash
  gcloud auth login
  gcloud auth application-default login
  ```
- [ ] **Set default project** (will be created by Terraform):
  ```bash
  gcloud config set project mailagent-dev-temp
  ```

## Phase 2: Terraform Infrastructure (45-60 minutes)

### 2.1 Create Terraform Configuration
- [ ] **Create terraform directory**:
  ```bash
  mkdir terraform
  cd terraform
  ```

- [ ] **Create `versions.tf`**:
  ```hcl
  terraform {
    required_version = ">= 1.0"
    required_providers {
      google = {
        source  = "hashicorp/google"
        version = "~> 5.0"
      }
      random = {
        source  = "hashicorp/random"
        version = "~> 3.1"
      }
    }
  }
  ```

- [ ] **Create `variables.tf`**:
  ```hcl
  variable "billing_account" {
    description = "The billing account ID"
    type        = string
  }

  variable "region" {
    description = "The default region"
    type        = string
    default     = "us-central1"
  }

  variable "support_email" {
    description = "Support email for OAuth consent screen"
    type        = string
  }
  ```

- [ ] **Create `main.tf`** (see detailed config below)
- [ ] **Create `outputs.tf`** (see detailed config below)
- [ ] **Create `terraform.tfvars`** (gitignored):
  ```hcl
  billing_account = "YOUR_BILLING_ACCOUNT_ID"
  support_email   = "your-email@gmail.com"
  region         = "us-central1"
  ```

### 2.2 Deploy Infrastructure
- [ ] **Initialize Terraform**:
  ```bash
  terraform init
  ```
- [ ] **Plan deployment**:
  ```bash
  terraform plan
  ```
- [ ] **Apply configuration**:
  ```bash
  terraform apply
  ```
- [ ] **Save outputs** (project ID, service account email, etc.)

## Phase 3: Manual OAuth2 Configuration (60-90 minutes)

### 3.1 OAuth Consent Screen Setup
- [ ] **Navigate to Google Cloud Console**
- [ ] **Go to**: APIs & Services → OAuth consent screen
- [ ] **Choose "External" user type**
- [ ] **Fill App Information**:
  - App name: `Mailagent`
  - User support email: `your-email@gmail.com`
  - App logo: (optional, skip for now)
  - App domain: (leave blank for development)
  - Developer contact: `your-email@gmail.com`
- [ ] **Click "Save and Continue"**

### 3.2 Configure Scopes
- [ ] **Click "Add or Remove Scopes"**
- [ ] **Add Gmail scopes**:
  - `https://www.googleapis.com/auth/gmail.readonly`
  - `https://www.googleapis.com/auth/gmail.modify` (if needed for marking as read)
- [ ] **Save scopes**
- [ ] **Click "Save and Continue"**

### 3.3 Add Test Users
- [ ] **Add your Gmail address** as test user
- [ ] **Add any other emails** you want to test with
- [ ] **Click "Save and Continue"**
- [ ] **Review summary** and go back to dashboard

### 3.4 Create OAuth2 Client Credentials
- [ ] **Navigate to**: APIs & Services → Credentials
- [ ] **Click "Create Credentials"** → "OAuth 2.0 Client IDs"
- [ ] **Choose "Desktop application"**
- [ ] **Name**: `Mailagent Desktop Client`
- [ ] **Click "Create"**
- [ ] **Download JSON file** (save as `client_credentials.json`)
- [ ] **Store securely** (DO NOT commit to git)

## Phase 4: Local Development Setup (30-45 minutes)

### 4.1 Project Structure Setup
- [ ] **Create project directories**:
  ```bash
  mkdir -p src/mailagent/{auth,gmail,storage}
  mkdir -p config
  mkdir -p logs
  ```

### 4.2 Environment Configuration
- [ ] **Create `.env.example`**:
  ```env
  GOOGLE_CLOUD_PROJECT=your-project-id
  GOOGLE_APPLICATION_CREDENTIALS=path/to/service-account-key.json
  GMAIL_CREDENTIALS_PATH=path/to/client_credentials.json
  GMAIL_TOKEN_PATH=path/to/token.json
  DATABASE_URL=sqlite:///./mailagent.db
  LOG_LEVEL=INFO
  ```
- [ ] **Copy to `.env`** and fill in actual values
- [ ] **Add `.env` to `.gitignore`**

### 4.3 Authentication Test Script
- [ ] **Create `scripts/test_auth.py`**:
  ```python
  #!/usr/bin/env python3
  """Test Gmail API authentication"""

  import os
  from google.auth.transport.requests import Request
  from google.oauth2.credentials import Credentials
  from google_auth_oauthlib.flow import InstalledAppFlow
  from googleapiclient.discovery import build

  SCOPES = ['https://www.googleapis.com/auth/gmail.readonly']

  def main():
      creds = None
      token_path = os.getenv('GMAIL_TOKEN_PATH', 'token.json')
      credentials_path = os.getenv('GMAIL_CREDENTIALS_PATH', 'client_credentials.json')

      # Load existing token
      if os.path.exists(token_path):
          creds = Credentials.from_authorized_user_file(token_path, SCOPES)

      # If no valid credentials, run OAuth flow
      if not creds or not creds.valid:
          if creds and creds.expired and creds.refresh_token:
              creds.refresh(Request())
          else:
              flow = InstalledAppFlow.from_client_secrets_file(credentials_path, SCOPES)
              creds = flow.run_local_server(port=0)

          # Save credentials for next run
          with open(token_path, 'w') as token:
              token.write(creds.to_json())

      # Test API access
      service = build('gmail', 'v1', credentials=creds)
      results = service.users().labels().list(userId='me').execute()
      labels = results.get('labels', [])

      print(f"✅ Authentication successful!")
      print(f"📧 Found {len(labels)} Gmail labels")
      print("🎉 Ready to start development!")

  if __name__ == '__main__':
      main()
  ```

## Phase 5: Initial Testing & Validation (30 minutes)

### 5.1 Test Authentication
- [ ] **Run authentication test**:
  ```bash
  cd scripts
  python test_auth.py
  ```
- [ ] **Complete OAuth flow** in browser
- [ ] **Verify token.json** is created
- [ ] **Confirm Gmail API access** works

### 5.2 Test Infrastructure
- [ ] **Verify project exists** in Google Cloud Console
- [ ] **Check enabled APIs**:
  - Gmail API
  - Cloud Storage API
  - Secret Manager API
- [ ] **Verify service account** exists
- [ ] **Test storage bucket** access (if created)

### 5.3 Documentation Updates
- [ ] **Update README.md** with setup instructions
- [ ] **Document any issues** encountered
- [ ] **Note project ID and key paths** for team

## Phase 6: Security & Cleanup (15-30 minutes)

### 6.1 Secure Credential Storage
- [ ] **Move credentials** out of Downloads folder
- [ ] **Set proper file permissions**:
  ```bash
  chmod 600 client_credentials.json
  chmod 600 token.json
  chmod 600 service-account-key.json
  ```
- [ ] **Verify `.gitignore`** excludes all credential files

### 6.2 Initial Security Review
- [ ] **Review IAM permissions** in Cloud Console
- [ ] **Check OAuth consent screen** settings
- [ ] **Verify test user list** is correct
- [ ] **Enable audit logging** (optional but recommended)

## Troubleshooting Checklist 🔧

### Common Issues & Solutions

**Terraform Issues:**
- [ ] `billing_account` not found → Check billing account ID format
- [ ] `project_id` already exists → Add random suffix or change name
- [ ] API not enabled → Wait 2-3 minutes after terraform apply

**OAuth Issues:**
- [ ] "Access blocked" → Add email to test users list
- [ ] "Invalid client" → Re-download client credentials
- [ ] "Scope not authorized" → Check consent screen scope configuration

**Authentication Issues:**
- [ ] `credentials not found` → Check file paths in `.env`
- [ ] `token expired` → Delete `token.json` and re-authenticate
- [ ] `permission denied` → Verify service account IAM roles

## Success Criteria ✨

By end of day, you should have:
- [ ] ✅ **Working Google Cloud project** with all required APIs
- [ ] ✅ **Functional OAuth2 authentication** for Gmail API
- [ ] ✅ **Service account** with proper permissions
- [ ] ✅ **Local development environment** configured
- [ ] ✅ **Successful Gmail API test** (can list labels/emails)
- [ ] ✅ **Secure credential storage** setup
- [ ] ✅ **Documentation** updated with your specific setup

## Next Steps (June 15+)

- [ ] Implement email fetching logic
- [ ] Set up SQLite database schema
- [ ] Create embedding service integration
- [ ] Build digest generation pipeline
- [ ] Set up scheduling with APScheduler

## Emergency Contacts & Resources

- **Google Cloud Support**: Available if billing issues
- **Gmail API Docs**: https://developers.google.com/gmail/api
- **Terraform Google Provider**: https://registry.terraform.io/providers/hashicorp/google
- **OAuth2 Troubleshooting**: https://developers.google.com/identity/protocols/oauth2/troubleshooting

---

**Remember**: Take breaks, document issues as you go, and don't hesitate to refer back to `docs/google-cloud-setup.md` for detailed explanations! 🚀
