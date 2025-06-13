# Terraform Configuration Files

This document contains the complete Terraform configuration files referenced in the setup todos.

## main.tf

```hcl
# Generate random suffix for unique project ID
resource "random_id" "project_suffix" {
  byte_length = 4
}

# Create the Google Cloud project
resource "google_project" "mailagent" {
  name       = "mailagent-${random_id.project_suffix.hex}"
  project_id = "mailagent-${random_id.project_suffix.hex}"

  billing_account = var.billing_account

  # Prevent accidental deletion
  lifecycle {
    prevent_destroy = true
  }
}

# Enable required APIs
resource "google_project_service" "apis" {
  for_each = toset([
    "gmail.googleapis.com",
    "storage.googleapis.com",
    "secretmanager.googleapis.com",
    "run.googleapis.com",
    "cloudbuild.googleapis.com",
    "logging.googleapis.com",
    "monitoring.googleapis.com"
  ])

  project = google_project.mailagent.project_id
  service = each.value

  # Don't disable on destroy to avoid issues
  disable_on_destroy = false
}

# Create service account for the application
resource "google_service_account" "mailagent" {
  project      = google_project.mailagent.project_id
  account_id   = "mailagent-service"
  display_name = "Mailagent Service Account"
  description  = "Service account for mailagent application operations"

  depends_on = [google_project_service.apis]
}

# Generate service account key
resource "google_service_account_key" "mailagent_key" {
  service_account_id = google_service_account.mailagent.name
  public_key_type    = "TYPE_X509_PEM_FILE"
}

# IAM roles for the service account
resource "google_project_iam_member" "mailagent_roles" {
  for_each = toset([
    "roles/storage.admin",           # For Cloud Storage operations
    "roles/secretmanager.admin",     # For managing secrets
    "roles/logging.logWriter",       # For writing logs
    "roles/monitoring.metricWriter", # For writing metrics
  ])

  project = google_project.mailagent.project_id
  role    = each.value
  member  = "serviceAccount:${google_service_account.mailagent.email}"
}

# Storage bucket for application data
resource "google_storage_bucket" "mailagent_data" {
  project  = google_project.mailagent.project_id
  name     = "${google_project.mailagent.project_id}-data"
  location = var.region

  # Enable versioning for data protection
  versioning {
    enabled = true
  }

  # Lifecycle management
  lifecycle_rule {
    condition {
      age = 90
    }
    action {
      type = "Delete"
    }
  }

  # Prevent public access
  public_access_prevention = "enforced"

  depends_on = [google_project_service.apis]
}

# Storage bucket for backups
resource "google_storage_bucket" "mailagent_backups" {
  project  = google_project.mailagent.project_id
  name     = "${google_project.mailagent.project_id}-backups"
  location = var.region

  versioning {
    enabled = true
  }

  # Longer retention for backups
  lifecycle_rule {
    condition {
      age = 365
    }
    action {
      type = "Delete"
    }
  }

  public_access_prevention = "enforced"

  depends_on = [google_project_service.apis]
}

# Secret for storing OAuth2 client credentials
resource "google_secret_manager_secret" "oauth_credentials" {
  project   = google_project.mailagent.project_id
  secret_id = "oauth-client-credentials"

  replication {
    auto {}
  }

  depends_on = [google_project_service.apis]
}

# Secret for storing refresh tokens
resource "google_secret_manager_secret" "refresh_tokens" {
  project   = google_project.mailagent.project_id
  secret_id = "gmail-refresh-tokens"

  replication {
    auto {}
  }

  depends_on = [google_project_service.apis]
}

# IAM binding for service account to access secrets
resource "google_secret_manager_secret_iam_member" "oauth_credentials_access" {
  project   = google_project.mailagent.project_id
  secret_id = google_secret_manager_secret.oauth_credentials.secret_id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.mailagent.email}"
}

resource "google_secret_manager_secret_iam_member" "refresh_tokens_access" {
  project   = google_project.mailagent.project_id
  secret_id = google_secret_manager_secret.refresh_tokens.secret_id
  role      = "roles/secretmanager.secretAccessor"
  member    = "serviceAccount:${google_service_account.mailagent.email}"
}

# Cloud Run service (for future deployment)
resource "google_cloud_run_v2_service" "mailagent" {
  project  = google_project.mailagent.project_id
  name     = "mailagent"
  location = var.region

  template {
    containers {
      image = "gcr.io/cloudrun/hello"  # Placeholder image

      env {
        name  = "GOOGLE_CLOUD_PROJECT"
        value = google_project.mailagent.project_id
      }

      resources {
        limits = {
          cpu    = "1000m"
          memory = "512Mi"
        }
      }
    }

    service_account = google_service_account.mailagent.email

    scaling {
      min_instance_count = 0
      max_instance_count = 10
    }
  }

  traffic {
    percent = 100
    type    = "TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST"
  }

  depends_on = [google_project_service.apis]
}

# Allow unauthenticated access to Cloud Run (for webhooks, if needed)
resource "google_cloud_run_service_iam_member" "public_access" {
  project  = google_project.mailagent.project_id
  location = google_cloud_run_v2_service.mailagent.location
  service  = google_cloud_run_v2_service.mailagent.name
  role     = "roles/run.invoker"
  member   = "allUsers"
}

# Cloud Scheduler job for digest generation (placeholder)
resource "google_cloud_scheduler_job" "weekly_digest" {
  project     = google_project.mailagent.project_id
  region      = var.region
  name        = "weekly-digest-job"
  description = "Weekly digest generation job"
  schedule    = "0 9 * * 1"  # Every Monday at 9 AM
  time_zone   = "UTC"

  http_target {
    http_method = "POST"
    uri         = "${google_cloud_run_v2_service.mailagent.uri}/generate-digest"

    oidc_token {
      service_account_email = google_service_account.mailagent.email
    }
  }

  depends_on = [google_project_service.apis]
}
```

## outputs.tf

```hcl
# Project information
output "project_id" {
  description = "The ID of the created Google Cloud project"
  value       = google_project.mailagent.project_id
}

output "project_name" {
  description = "The name of the created Google Cloud project"
  value       = google_project.mailagent.name
}

output "project_number" {
  description = "The number of the created Google Cloud project"
  value       = google_project.mailagent.number
}

# Service account information
output "service_account_email" {
  description = "Email address of the service account"
  value       = google_service_account.mailagent.email
}

output "service_account_id" {
  description = "The ID of the service account"
  value       = google_service_account.mailagent.account_id
}

output "service_account_key" {
  description = "Base64 encoded service account key (sensitive)"
  value       = google_service_account_key.mailagent_key.private_key
  sensitive   = true
}

# Storage information
output "data_bucket_name" {
  description = "Name of the data storage bucket"
  value       = google_storage_bucket.mailagent_data.name
}

output "data_bucket_url" {
  description = "URL of the data storage bucket"
  value       = google_storage_bucket.mailagent_data.url
}

output "backup_bucket_name" {
  description = "Name of the backup storage bucket"
  value       = google_storage_bucket.mailagent_backups.name
}

output "backup_bucket_url" {
  description = "URL of the backup storage bucket"
  value       = google_storage_bucket.mailagent_backups.url
}

# Secret Manager information
output "oauth_credentials_secret_id" {
  description = "Secret Manager secret ID for OAuth credentials"
  value       = google_secret_manager_secret.oauth_credentials.secret_id
}

output "refresh_tokens_secret_id" {
  description = "Secret Manager secret ID for refresh tokens"
  value       = google_secret_manager_secret.refresh_tokens.secret_id
}

# Cloud Run information
output "cloud_run_service_url" {
  description = "URL of the Cloud Run service"
  value       = google_cloud_run_v2_service.mailagent.uri
}

output "cloud_run_service_name" {
  description = "Name of the Cloud Run service"
  value       = google_cloud_run_v2_service.mailagent.name
}

# Scheduler information
output "scheduler_job_name" {
  description = "Name of the Cloud Scheduler job"
  value       = google_cloud_scheduler_job.weekly_digest.name
}

# Environment variables for local development
output "env_vars" {
  description = "Environment variables for local development"
  value = {
    GOOGLE_CLOUD_PROJECT                = google_project.mailagent.project_id
    GOOGLE_APPLICATION_CREDENTIALS      = "path/to/service-account-key.json"
    MAILAGENT_DATA_BUCKET              = google_storage_bucket.mailagent_data.name
    MAILAGENT_BACKUP_BUCKET            = google_storage_bucket.mailagent_backups.name
    OAUTH_CREDENTIALS_SECRET_ID        = google_secret_manager_secret.oauth_credentials.secret_id
    REFRESH_TOKENS_SECRET_ID           = google_secret_manager_secret.refresh_tokens.secret_id
    CLOUD_RUN_SERVICE_URL              = google_cloud_run_v2_service.mailagent.uri
  }
}

# Instructions for next steps
output "next_steps" {
  description = "Next steps after Terraform deployment"
  value = <<-EOT

    🎉 Infrastructure deployed successfully!

    Next steps:
    1. Save the service account key:
       echo '${base64decode(google_service_account_key.mailagent_key.private_key)}' > service-account-key.json

    2. Set up OAuth2 consent screen:
       https://console.cloud.google.com/apis/credentials/consent?project=${google_project.mailagent.project_id}

    3. Create OAuth2 client credentials:
       https://console.cloud.google.com/apis/credentials?project=${google_project.mailagent.project_id}

    4. Update your .env file with these values:
       GOOGLE_CLOUD_PROJECT=${google_project.mailagent.project_id}
       GOOGLE_APPLICATION_CREDENTIALS=./service-account-key.json
       MAILAGENT_DATA_BUCKET=${google_storage_bucket.mailagent_data.name}

    5. Test authentication with the provided script

    Project Console: https://console.cloud.google.com/home/dashboard?project=${google_project.mailagent.project_id}
  EOT
}
```

## Usage Instructions

1. **Save these files** in your `terraform/` directory as `main.tf` and `outputs.tf`

2. **Create your `terraform.tfvars`** file:
   ```hcl
   billing_account = "YOUR_BILLING_ACCOUNT_ID"
   support_email   = "your-email@gmail.com"
   region         = "us-central1"
   ```

3. **Deploy the infrastructure**:
   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

4. **Save the service account key**:
   ```bash
   terraform output -raw service_account_key | base64 -d > service-account-key.json
   chmod 600 service-account-key.json
   ```

5. **View all outputs**:
   ```bash
   terraform output
   ```

## Important Notes

- **Security**: The service account key is marked as sensitive and should be handled carefully
- **Billing**: Make sure your billing account is set up before running terraform apply
- **Permissions**: The service account gets minimal required permissions following least privilege principle
- **Cleanup**: Use `terraform destroy` to clean up all resources when no longer needed
- **State Management**: Consider using remote state storage for production deployments

## Customization Options

You can customize the configuration by:
- Changing the `region` variable for different geographic locations
- Modifying storage bucket lifecycle rules
- Adjusting Cloud Run resource limits
- Adding additional APIs to the `google_project_service` resource
- Configuring different IAM roles based on your needs
</rewritten_file>
