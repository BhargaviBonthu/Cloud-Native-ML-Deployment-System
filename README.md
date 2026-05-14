# ☁️ Cloud-Native ML Deployment System

> **Provision. Train. Deploy. Monitor. Recover — automatically.**
> A production-grade ML infrastructure built on AWS, designed for zero-touch deployments, real-time observability, and self-healing availability.


---

## 🧭 What This Is

Most ML projects stop at the model. This project starts there.

The Cloud-Native ML Deployment System is a full-stack infrastructure for taking a trained model and making it reliable, observable, and self-healing in production. It handles everything from environment provisioning to drift detection — so the model keeps being right, not just at launch, but every day after.

**Key design philosophy:**
- Infrastructure is code. Zero manual steps.
- Observability Every layer is instrumented.
- Failures are expected. Recovery is automated.

---

## ✨ Features at a Glance

| Capability | What It Does |
|---|---|
| **One-command provisioning** | Full AWS environment up in under 10 minutes via Terraform |
| **p99 latency auto-rollback** | Lambda watches latency; breaches trigger automatic rollback |
| **Model drift detection** | Daily KL-divergence check against training baseline |
| **Structured logging everywhere** | Every service layer emits structured JSON — queryable, alertable |
| **Slack alerting** | Drift, latency, and error-rate alerts routed to Slack in real time |
| **Reproducible from zero** | Idempotent Terraform — destroy and rebuild in minutes |

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                        VPC                              │
│                                                         │
│   ┌──────────┐    ┌──────────┐    ┌─────────────────┐  │
│   │   EC2    │───▶│  Model   │───▶│   S3 Artifact   │  │
│   │ Inference│    │ Service  │    │     Store        │  │
│   │  Server  │    │ (Python) │    └─────────────────┘  │
│   └────┬─────┘    └────┬─────┘                         │
│        │               │                               │
│   ┌────▼───────────────▼──────────────────────────┐    │
│   │              CloudWatch                        │    │
│   │   Metrics · Alarms · Structured Logs · Dashbd  │    │
│   └────────────────────┬───────────────────────────┘    │
│                        │  p99 latency breach            │
│                   ┌────▼────┐                           │
│                   │ Lambda  │──── auto-rollback          │
│                   │Watchdog │                           │
│                   └─────────┘                           │
└─────────────────────────────────────────────────────────┘
          │                           │
   ┌──────▼──────┐           ┌────────▼───────┐
   │  IAM Roles  │           │  Drift Detector│
   │  & Policies │           │ (Daily Lambda) │──── Slack Alert
   └─────────────┘           └────────────────┘
```

---

## 📦 Stack

### Infrastructure
- **Terraform** — provisions EC2, S3, Lambda, IAM , VPC, subnets, security groups
- **AWS EC2** — model inference servers
- **AWS S3** — model artifact storage, versioned
- **AWS Lambda** — latency watchdog + daily drift detector
- **AWS IAM** — least-privilege roles per service component
- **AWS VPC** — isolated network with public/private subnet separation

### Observability
- **AWS CloudWatch** — metrics, log groups, dashboards, alarms
- **Structured JSON logging** — every service emits machine-readable logs with trace context
- **Slack webhooks** — alert routing for drift and latency events

### ML
- **Python (scikit-learn / custom)** — model training and inference
- **KL-Divergence** — statistical measure for distribution shift detection
- **S3-versioned artifacts** — model checkpointing and rollback targets

---

## 🚀 Getting Started

### Prerequisites

```bash
# Required tools
terraform >= 1.5
python >= 3.11
aws-cli >= 2.x (configured with credentials)
```

### 1. Clone & configure

```bash
git clone https://github.com/BhargaviBonthu/cloud-native-ml-deployment.git
cd cloud-native-ml-deployment

cp terraform/terraform.tfvars.example terraform/terraform.tfvars
# Edit terraform.tfvars with your AWS region, account ID, Slack webhook URL
```

### 2. Provision the entire environment

```bash
cd terraform
terraform init
terraform plan
terraform apply  
```

This creates:
- VPC with public/private subnets
- EC2 inference server (auto-configured via user_data)
- S3 bucket for model artifacts (versioning enabled)
- IAM roles with least-privilege policies
- CloudWatch log groups and metric alarms
- Lambda functions (watchdog + drift detector)

### 3. Deploy a model

```bash
# Upload model artifact
python scripts/upload_model.py \
  --model-path ./models/my_model.pkl \
  --version v1.2.0

# Deploy to inference server
python scripts/deploy.py --version v1.2.0
```

### 4. Verify health

```bash
# Check inference endpoint
curl https://<your-ec2-endpoint>/health

# Check CloudWatch dashboard
aws cloudwatch get-dashboard --dashboard-name MLInferenceDashboard
```

---

## 📊 Observability in Depth

### Structured Logging

Every service layer emits JSON logs with consistent fields:

```json
{
  "timestamp": "2025-11-01T14:32:01Z",
  "service": "inference-server",
  "level": "INFO",
  "request_id": "req_8f3a21b",
  "model_version": "v1.2.0",
  "latency_ms": 43,
  "prediction": 0.87,
  "input_hash": "sha256:a3f9..."
}
```

Logs are shipped to CloudWatch Log Groups and queryable with CloudWatch Insights.

### Latency Watchdog (Lambda)

A Lambda function runs on a 1-minute schedule and evaluates p99 latency:

```python
# Simplified watchdog logic
def handler(event, context):
    p99 = get_cloudwatch_p99_latency(window_minutes=5)
    if p99 > LATENCY_THRESHOLD_MS:
        previous_version = get_previous_stable_version()
        trigger_rollback(previous_version)
        notify_slack(f"⚠️ Rollback triggered — p99={p99}ms exceeded threshold")
```

### CloudWatch Alarms

| Alarm | Threshold | Action |
|---|---|---|
| `HighP99Latency` | > 200ms for 3 datapoints | Trigger Lambda rollback |
| `HighErrorRate` | > 1% errors in 5 min | Alert to Slack |
| `InferenceServerDown` | 0 healthy instances | Page on-call |
| `DriftDetected` | KL-divergence > 0.1 | Alert to Slack |

---

## 🔬 Model Drift Detection

### What Is Drift?

After deployment, the real-world data distribution can shift away from what the model was trained on. This system detects that automatically  before it causes silent prediction degradation.

### How It Works

A daily Lambda batch job:
1. Pulls the last 24h of live inference inputs from S3
2. Computes the KL-divergence between the live feature distribution and the training baseline distribution
3. If KL-divergence exceeds the configured threshold → Slack alert is fired


### Drift Alert Example

```
🚨 Drift Alert — cloud-native-ml-prod
──────────────────────────────────────
Model version : v1.2.0
KL-divergence : 0.1847  (threshold: 0.10)
Feature flagged: customer_age_bucket
Window        : 2025-11-01 00:00 → 23:59 UTC
Action needed : Review input pipeline or trigger retrain
```

---

## 🔁 Automated Rollback Flow

```
Live Traffic
     │
     ▼
CloudWatch p99 Metric
     │
     │  > 200ms for 3 consecutive datapoints
     ▼
Lambda Watchdog Triggered
     │
     ├── 1. Identify last stable model version from S3 tags
     ├── 2. Swap EC2 inference server to previous artifact
     ├── 3. Validate health check passes
     └── 4. Notify Slack: "Rollback to v1.1.0 complete ✅"
```

Zero manual intervention. Mean time to recovery: under 3 minutes.

---

## 📁 Project Structure

```
cloud-native-ml-deployment/
│
├── terraform/                    # All IaC — provision with one command
│   ├── main.tf                   # Root module
│   ├── ec2.tf                    # Inference server
│   ├── s3.tf                     # Artifact store
│   ├── lambda.tf                 # Watchdog + drift detector
│   ├── iam.tf                    # Least-privilege roles
│   ├── vpc.tf                    # Network isolation
│   ├── cloudwatch.tf             # Alarms, dashboards, log groups
│   └── terraform.tfvars.example
│
├── src/
│   ├── inference/
│   │   ├── server.py             # FastAPI inference endpoint
│   │   ├── model_loader.py       # S3 artifact loader
│   │   └── logging_config.py    # Structured JSON logger
│   │
│   ├── watchdog/
│   │   └── latency_watchdog.py  # Lambda: p99 check + rollback
│   │
│   └── drift/
│       ├── detector.py           # KL-divergence computation
│       ├── baseline_loader.py    # Load training distribution from S3
│       └── alerting.py           # Slack webhook integration
│
├── scripts/
│   ├── upload_model.py           # Upload + version model artifacts
│   ├── deploy.py                 # Deploy specific version to EC2
│   └── rollback.py               # Manual rollback (emergency use)
│
├── tests/
│   ├── test_drift_detector.py
│   ├── test_watchdog.py
│   └── test_inference_server.py
│
├── docs/
│   └── architecture.md
│
└── README.md
```

---

## 🧪 Running Tests

```bash
pip install -r requirements-dev.txt
pytest tests/ -v --cov=src --cov-report=term-missing
```

---

## ⚙️ Configuration Reference

| Variable | Default | Description |

| 'AWS_REGION' | 'us-east-1' | Target AWS region |
| 'LATENCY_THRESHOLD_MS' | '200' | p99 threshold before rollback |
| 'KL_DIVERGENCE_THRESHOLD' | '0.10' | Drift alert sensitivity |
| 'DRIFT_CHECK_SCHEDULE' | 'rate(1 day)' | How often drift runs |
| 'SLACK_WEBHOOK_URL' | — | Webhook for alerts |
| 'MODEL_S3_BUCKET' | — | S3 bucket for artifact storage |

---

## 🔒 Security

- All IAM roles follow least-privilege principle — each Lambda and EC2 instance has only the permissions it needs
- VPC isolates inference servers from public internet; only load balancer is public-facing
- S3 bucket has versioning enabled and public access blocked
- No credentials in code — all secrets via AWS Secrets Manager or environment variables

---

## 📈 Production Metrics 

| Metric | Value |

| Environment provisioning time | < 10 minutes from zero |
| Deployment pipeline (CI→live) | ~12 minutes (automated) |
| Mean time to detect (MTTD) | Proactive — alarms fire before users notice |
| Mean time to recover (MTTR) | < 3 minutes (automated rollback) |
| Drift check frequency | Daily |

---

## 🗺️ Roadmap

- Migrate IaC to Azure Bicep / Terraform AzureRM (Azure parity)
- Add A/B traffic splitting for shadow deployments
- Integrate SHAP for explainable drift attribution (which features drifted?)
- Add Grafana dashboard export for CloudWatch metrics
- Support multi-model serving with routing layer

---

## 👩‍💻 Author

**Bhargavi Bonthu**
Software Engineer · ML Systems · Cloud Infrastructure


---

## 📄 License

MIT — see [LICENSE](./LICENSE) for details.

---

