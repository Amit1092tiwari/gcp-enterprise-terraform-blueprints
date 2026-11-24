# ☁️ Google Cloud Platform Enterprise Infrastructure-as-Code (Terraform Blueprints)

[![License: Apache-2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Terraform](https://img.shields.io/badge/Terraform-%3E%3D_1.0.0-623CE4?logo=terraform)](https://www.terraform.io/)
[![Google Cloud Provider](https://img.shields.io/badge/GCP_Provider-%3E%3D_4.0-4285F4?logo=googlecloud)](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
[![CI Validation](https://img.shields.io/badge/CI-Automated_Syntax_%26_Lint-success.svg)](#-continuous-integration--validation)
[![Architecture: Well-Architected](https://img.shields.io/badge/Google_Cloud-Well--Architected-009688)](#-google-cloud-well-architected-framework-alignment)

Production-grade, modular Terraform blueprints demonstrating secure, multi-region serverless compute, event-driven microservices, private VPC networking, Cloud SQL integration, and automated Cloud Build release pipelines on Google Cloud Platform.

---

## 🏛️ Enterprise Cloud Architecture & Service Fabric

The following architecture illustrates the interconnected service topology provisioned across these modules:

```mermaid
flowchart TD
    Client["🌐 Client / API Gateway / CDN"] -->|HTTPS (Custom Domain)| CloudArmor["🛡️ Cloud Armor & Cloud Load Balancer"]
    
    subgraph EdgeServices ["Edge & Ingress Routing"]
        CloudArmor -->|Direct Ingress| CR_Public["Cloud Run (Public Entrypoint)"]
        CloudArmor -->|Traffic Split (90/10 Canary)| CR_Revision["Cloud Run (Canary Revision)"]
    end

    subgraph VPC_Fabric ["Enterprise VPC & Private Service Networking"]
        Connector["VPC Access Connector (Serverless VPC)"]
        PrivateSubnet["Private IP Subnet (No Public IP)"]
        CloudSQL["🗄️ Cloud SQL (Postgres / MySQL / MSSQL)"]
        
        CR_Public -->|Egress via Connector| Connector
        Connector --> PrivateSubnet
        PrivateSubnet -->|Private Service Access RFC 1918| CloudSQL
    end

    subgraph EventFabric ["Event-Driven Ingestion & Task Orchestration"]
        PubSub["📬 Cloud Pub/Sub Topic"]
        Tasks["⏱️ Cloud Tasks Queue"]
        Scheduler["⏰ Cloud Scheduler (Cron)"]
        
        CR_Public -->|Publish Event| PubSub
        PubSub -->|Push Subscription| CF_Worker["⚡ Cloud Functions v2 (Event Worker)"]
        Scheduler -->|Trigger Ingestion| CR_Scheduled["Cloud Run (Scheduled Batch)"]
        Tasks -->|Buffered Dispatch| CR_Tasks["Cloud Run (Task Consumer)"]
    end

    subgraph StorageFabric ["Storage & Audit Layer"]
        GCS["🪣 Cloud Storage (Object Store)"]
        AuditLogs["📑 Cloud Audit Logs & Logging Sink"]
        
        CF_Worker -->|Process & Store| GCS
        CR_Public -.->|Log Traces| AuditLogs
    end

    subgraph CI_CD ["Automated Delivery & Build Engine"]
        CloudBuild["🚀 Cloud Build Pipeline"]
        ArtifactReg["📦 Artifact Registry"]
        
        CloudBuild -->|Build & Tag Image| ArtifactReg
        ArtifactReg -->|Deploy Revision| CR_Public
    end
```

---

## 📋 Comprehensive Module Catalog

Over 160 Terraform configurations structured into specialized enterprise workload domains:

### 1. Serverless Compute & Traffic Engineering (Cloud Run)
| Module Path | Primary GCP Resources | Architecture Pattern |
| :--- | :--- | :--- |
| [`cloud_run_service_custom_domain_mapping`](cloud_run_service_custom_domain_mapping/) | `google_cloud_run_domain_mapping` | Custom domain DNS verification and automated SSL/TLS management. |
| [`cloud_run_service_traffic_split`](cloud_run_service_traffic_split/) | `google_cloud_run_service` | Deterministic percentage-based canary traffic routing between revisions. |
| [`cloud_run_service_traffic_gradual_rollout`](cloud_run_service_traffic_gradual_rollout/) | `google_cloud_run_service` | Zero-downtime blue/green deployment orchestration. |
| [`cloud_run_service_multiple_regions`](cloud_run_service_multiple_regions/) | `google_cloud_run_service` | Active-Active multi-region deployment for latency optimization and disaster recovery. |
| [`cloud_run_service_ingress`](cloud_run_service_ingress/) | `google_cloud_run_service` | Strict ingress enforcement (`internal-and-cloud-load-balancing`). |
| [`cloud_run_service_interservice`](cloud_run_service_interservice/) | `google_cloud_run_service_iam_member` | Service-to-service zero-trust IAM authentication via OIDC identity tokens. |

### 2. Private Networking & Database Interconnects
| Module Path | Primary GCP Resources | Architecture Pattern |
| :--- | :--- | :--- |
| [`cloud_run_service_connect_cloud_sql`](cloud_run_service_connect_cloud_sql/) | Cloud Run + Cloud SQL instance connection | High-performance private Unix socket and Cloud SQL Auth Proxy injection. |
| [`cloudrun_vpc_access_connector`](cloudrun_vpc_access_connector/) | `google_vpc_access_connector` | Serverless VPC Access bridging serverless containers to RFC 1918 private subnets. |
| [`cloud_run_service_static_outbound`](cloud_run_service_static_outbound/) | Cloud Run + Cloud NAT + Static External IP | Deterministic static egress IP routing for enterprise firewall whitelisting. |

### 3. Event-Driven Workflows & Asynchronous Processing
| Module Path | Primary GCP Resources | Architecture Pattern |
| :--- | :--- | :--- |
| [`cloud_run_service_pubsub`](cloud_run_service_pubsub/) | Cloud Pub/Sub + Cloud Run push subscription | Scalable, at-least-once message consumption with dead-letter queue routing. |
| [`cloud_run_service_scheduled`](cloud_run_service_scheduled/) | Cloud Scheduler + Cloud Run service | Cron-driven automated batch ingestion and maintenance invocations. |
| [`cloud_run_service_tasks`](cloud_run_service_tasks/) | Cloud Tasks queue + Cloud Run consumer | Rate-limited, buffered background task processing with exponential backoff. |
| [`cloudfunctions2_basic`](cloudfunctions2_basic/) | `google_cloudfunctions2_function` | 2nd-gen Cloud Functions powered by Cloud Run concurrency and Eventarc. |
| [`cloudfunctions2_basic_gcs`](cloudfunctions2_basic_gcs/) | Cloud Functions v2 + Cloud Storage trigger | Eventarc object finalization triggers for real-time file processing pipelines. |

### 4. Enterprise Compute & Auto-Scaling
| Module Path | Primary GCP Resources | Architecture Pattern |
| :--- | :--- | :--- |
| [`autoscaler_instance_group`](autoscaler_instance_group/) | `google_compute_autoscaler`, MIG | Horizontal autoscaling managed instance groups based on CPU and custom Cloud Monitoring metrics. |
| [`build`](build/) | Google Cloud Build YAML configurations | Automated container builds, linting stages, and integration test executions. |

---

## 🛡️ Google Cloud Well-Architected Framework Alignment

Every Terraform module is designed in compliance with Google Cloud architecture pillars:

1. **Security & Identity:**
   - Principle of Least Privilege: Dedicated, distinct Service Accounts for each service boundary.
   - Private IP Everywhere: Database engines and internal APIs are provisioned without public IP endpoints.
   - Secret Management: Secrets and credentials injected securely via Secret Manager references rather than plaintext state.
2. **Reliability & Resilience:**
   - Multi-Region Failover: Cross-region active-active deployments resilient against single-zone or regional outages.
   - Canary Rollouts: Percentage-based revision splitting prevents faulty updates from impacting all live users.
3. **Performance Efficiency:**
   - Serverless VPC Connectors tuned with appropriate `/28` CIDR blocks avoiding IP exhaustion.
   - Concurrency optimization leveraging Cloud Run multi-threading (up to 80 concurrent requests per container).
4. **Cost Optimization:**
   - Min-instances configured to `0` for non-critical workloads to achieve zero idle compute spend.
   - Autoscaler scaling schedules and predictive autoscaling to minimize over-provisioning.

---

## 🚀 Quickstart & Deployment Guide

### Prerequisites
* [Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/install-cli) ($\ge$ 1.0.0)
* [Google Cloud SDK (`gcloud`)](https://cloud.google.com/sdk/docs/install)
* Active GCP Project with billing enabled

### 1. Authenticate with Google Cloud
```bash
gcloud auth login
gcloud auth application-default login
gcloud config set project YOUR_PROJECT_ID
```

### 2. Deploy a Selected Blueprint (e.g., Cloud Run + Cloud SQL Connector)
```bash
cd cloud_run_service_connect_cloud_sql

# Initialize provider plugins and backend state
terraform init

# Validate configuration syntax
terraform validate

# Inspect execution plan
terraform plan -var="project_id=YOUR_PROJECT_ID"

# Apply infrastructure deployment
terraform apply -var="project_id=YOUR_PROJECT_ID" -auto-approve
```

---

## 🧪 Continuous Integration & Validation

Automated CI pipelines run on every commit and pull request via [`.github/workflows/terraform-ci.yml`](.github/workflows/terraform-ci.yml):

```bash
# Verify formatting standard across all modules
terraform fmt -check -recursive

# Run static security and best-practice linting
tflint --recursive
```

---

## 📜 Governance & Licensing
* **License:** Licensed under the [Apache License, Version 2.0](LICENSE).
* **Security Reporting:** See [SECURITY.md](SECURITY.md) for disclosure guidelines.
* **Contributing:** Contribution protocols detailed in [CONTRIBUTING.md](CONTRIBUTING.md).
* **Changelog:** Major revisions documented in [CHANGELOG.md](CHANGELOG.md).
