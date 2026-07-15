# Creative Studio Infrastructure

This repository contains the Terraform configuration for deploying the Creative Studio application platform (frontend and backend) to Google Cloud.

## 🚀 Overview

This infrastructure is managed using a modular, environment-based approach with Terraform. The key principles are:
* **Don't Repeat Yourself (DRY):** All the logic for creating a service is defined once in a reusable **module**.
* **Strong Isolation:** Each environment (`dev`, `prod`, etc.) is managed in its own directory, with its own state file, to prevent accidental changes to production.

## Attach this machine to an existing deployment

If you originally deployed from another computer and need Terraform here (e.g. project `bbva-genai-veo`):

1. Copy `infra/environments/dev-infra-example/` to a new folder such as `infra/environments/bbva-genai-veo/`.
2. Set `backend.tf` to the **same GCS bucket/prefix** used when the project was first deployed (ask the teammate / check the original machine).
3. Fill `*.tfvars` with the real project id, region, GitHub connection, and audiences.
4. Authenticate and init:

```bash
gcloud auth application-default login
gcloud config set project bbva-genai-veo
cd infra/environments/bbva-genai-veo
terraform init
terraform plan
```

5. Before applying Email/Password auth on an already-live Identity Platform project, **import** the singleton config so Terraform does not try to recreate it:

```bash
terraform import 'module.creative_studio_platform.google_identity_platform_config.auth' projects/bbva-genai-veo/config
```

6. Run `terraform apply` and create client users manually in Firebase Console → Authentication → Users.

## 📁 Directory Structure

The project is organized into `modules` and `environments`.

```
infrastructure/
│
├── modules/                # Reusable "Blueprints"
│   ├── cloud-run-service/  # Defines how to build ONE service
│   └── platform/           # Defines the ENTIRE application platform
│
└── environments/
    ├── dev/                # Configuration for the 'dev' environment
    │   ├── main.tf         # Calls the platform module with dev values
    │   ├── backend.tf      # Defines where to store the dev state file
    │   └── dev.tfvars      # Contains all variables for dev
    │
    └── prod/               # Configuration for the 'prod' environment
        └── ...
```
* **`/modules`**: Contains reusable building blocks. The `platform` module is the main entry point, which in turn uses the `cloud-run-service` module.
* **`/environments`**: Contains a directory for each distinct deployment environment. These directories call the `platform` module with the correct set of variables.

---

## Deploy in 20min!!
Just run this script which has a step by step approach for you to deploy the infrastructure and start the app, just follow the instructions
```
curl https://raw.githubusercontent.com/GoogleCloudPlatform/gcc-creative-studio/refs/heads/main/bootstrap.sh | bash
```

For better guidance, [we recorded a video](./screenshots/how_to_deploy_creative_studio.mp4) to showcase how to deploy Creative Studio in a completely new and fresh GCP Account.

<video controls autoplay loop width="100%" style="max-width: 1200px;">
  <source src="./screenshots/how_to_deploy_creative_studio.mp4" type="video/mp4">
  Your browser does not support the video tag. You can <a href="./screenshots/how_to_deploy_creative_studio.mp4">download the video here</a>.
</video>

## System Architecture & Dependencies

For a detailed overview of the **System Architecture**, **Technology Stack**, and **Dependencies**, please refer to the [Root README](../README.md#system-architecture).



