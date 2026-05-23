# Invision — DevSecOps CI/CD Pipeline

> A production-grade DevSecOps project: every Git push triggers a fully automated pipeline that builds, scans for vulnerabilities, runs security analysis, and deploys a containerized Flask application to AWS — zero manual steps, zero compromises on security.

---

## Pipeline Overview

```
Developer Push
      │
      ▼
┌─────────────────┐
│  GitHub Webhook │  ──── triggers on every push to main
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Jenkins Server │  ──── orchestrates all stages
└────────┬────────┘
         │
         ├──► Stage 1 │ Clone Code        → pulls latest source from GitHub
         │
         ├──► Stage 2 │ Build Docker Image → builds containerized Flask app
         │
         ├──► Stage 3 │ Trivy Scan         → scans image for CVEs (blocks on CRITICAL)
         │
         ├──► Stage 4 │ SonarQube SAST     → static code analysis, quality gate
         │
         ├──► Stage 5 │ Tag & Push to ECR  → pushes verified image to AWS ECR
         │
         └──► Stage 6 │ Deploy to EC2      → SSH pull + zero-downtime container swap
                               │
                               ▼
                     ┌──────────────────┐
                     │   AWS EC2        │  Flask app running on port 5000
                     │   + RDS Postgres │  persistent data layer
                     └──────────────────┘
```

**Security gates are non-negotiable:** if Trivy finds a CRITICAL vulnerability or SonarQube fails the quality gate, the pipeline halts. No broken or insecure image ever reaches ECR or production.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Application | Python (Flask), SQLAlchemy, Jinja2, Vanilla JS |
| Containerization | Docker, Docker Compose |
| CI/CD Orchestration | Jenkins (declarative pipeline, GitHub webhooks) |
| Container Registry | AWS ECR (Elastic Container Registry) |
| Security Scanning | Trivy (CVE scan), SonarQube (SAST / code quality) |
| Compute | AWS EC2 (Ubuntu) |
| Database | AWS RDS PostgreSQL |
| Authentication | Flask-Login, session management |
| SSH Automation | Jenkins → EC2 via SSH (no manual deployment) |

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────┐
│                        AWS Cloud                          │
│                                                          │
│   ┌─────────────┐    ┌──────────────┐   ┌────────────┐  │
│   │  Jenkins    │───►│   AWS ECR    │   │  AWS RDS   │  │
│   │  Server     │    │  (Image      │   │ PostgreSQL  │  │
│   │  (EC2)      │    │   Registry)  │   │  (db layer) │  │
│   └──────┬──────┘    └──────┬───────┘   └─────┬──────┘  │
│          │                  │                 │          │
│          │ SSH deploy        │ docker pull     │ connects │
│          ▼                  ▼                 ▼          │
│   ┌─────────────────────────────────────────────────┐    │
│   │               AWS EC2 (App Server)              │    │
│   │   Docker Container → Flask App (port 5000)      │    │
│   └─────────────────────────────────────────────────┘    │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ▲
         │  GitHub Webhook (push event)
         │
┌────────────────┐
│  Developer     │
│  Workstation   │
└────────────────┘
```

---

## Jenkins Pipeline — Stage by Stage

### Stage 1 — Clone Code
Pulls the latest code from the `main` branch on every push. No manual checkout required.

### Stage 2 — Build Docker Image
Builds the containerized Flask application using the `Dockerfile`. The image is tagged with the build version for traceability.

```groovy
stage('Build Docker Image') {
    steps {
        sh 'docker build -t $IMAGE_NAME .'
    }
}
```

### Stage 3 — Trivy Container Vulnerability Scan
Scans the Docker image for known CVEs before it is tagged or pushed anywhere. The pipeline is configured to **fail and halt** if any `CRITICAL` severity vulnerability is found — no insecure image ever reaches ECR.

```groovy
stage('Trivy Security Scan') {
    steps {
        sh '''
            trivy image --exit-code 1 \
                        --severity CRITICAL \
                        --no-progress \
                        $IMAGE_NAME:latest
        '''
    }
}
```

> `--exit-code 1` causes Jenkins to mark the build as FAILED on any CRITICAL CVE, blocking all downstream stages automatically.

### Stage 4 — SonarQube Static Code Analysis (SAST)
Runs SonarQube static analysis on the Python codebase. Enforces a **Quality Gate** — the pipeline proceeds only if code quality standards pass (no high-severity code smells, no duplications beyond threshold, no security hotspots left unreviewed).

```groovy
stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh 'sonar-scanner -Dsonar.projectKey=invision -Dsonar.sources=.'
        }
    }
}
stage('Quality Gate') {
    steps {
        timeout(time: 2, unit: 'MINUTES') {
            waitForQualityGate abortPipeline: true
        }
    }
}
```

### Stage 5 — Tag & Push to AWS ECR
After both security gates pass, the Docker image is tagged and pushed to AWS Elastic Container Registry. AWS CLI authenticates using IAM role credentials — no plaintext secrets in the pipeline.

```groovy
stage('Push to ECR') {
    steps {
        sh '''
            aws ecr get-login-password --region $REGION \
                | docker login --username AWS --password-stdin $ECR_REPO
            docker tag $IMAGE_NAME:latest $ECR_REPO:latest
            docker push $ECR_REPO:latest
        '''
    }
}
```

### Stage 6 — Deploy to EC2 via SSH
Jenkins SSHs into the production EC2 instance, pulls the new image from ECR, stops the old container, and starts the new one. `--restart always` ensures the container auto-recovers from reboots.

```groovy
stage('Deploy to APP_SERVER') {
    steps {
        sh '''
            ssh -o StrictHostKeyChecking=no $APP_SERVER << EOF
                aws ecr get-login-password --region $REGION \
                    | docker login --username AWS --password-stdin $ECR_REPO
                docker pull $ECR_REPO:latest
                docker stop $CONTAINER_NAME || true
                docker rm   $CONTAINER_NAME || true
                docker run -d -p 5000:5000 \
                    --name $CONTAINER_NAME \
                    --restart always \
                    $ECR_REPO:latest
            EOF
        '''
    }
}
```

---

## Application Features

Invision is a data intelligence platform built with Flask. It demonstrates that the DevSecOps pipeline supports a real, multi-feature production application — not just a toy app.

| Feature | Details |
|---|---|
| Authentication | Flask-Login with session management, `@login_required` on all protected routes |
| Data Visualization | CSV/Excel upload → Chart.js rendering (6 chart types) |
| Report Analysis | Document upload (PDF, DOCX, TXT) → server-side analysis → JSON response |
| Export | Download as CSV, Excel, JSON, PDF (via ReportLab) |
| Activity Log | Every user action recorded to RDS PostgreSQL via SQLAlchemy |
| Dark / Light Mode | Persisted in `localStorage`, synced across all pages |
| Responsive UI | Mobile-first, hamburger nav, sidebar collapse |

---

## Project Structure

```
invision/
├── app.py                  ← Flask app factory (create_app)
├── config.py               ← All configuration (DB, mail, secrets)
├── extensions.py           ← Shared extensions (db, login_manager)
├── models.py               ← SQLAlchemy models (User, UploadedFile, SavedChart)
├── requirements.txt
├── Dockerfile              ← Multi-stage production build
├── docker-compose.yml      ← Local dev environment
├── Jenkinsfile             ← Declarative CI/CD pipeline (6 stages)
├── test_app.py             ← Pytest unit tests (pipeline gate)
│
├── routes/                 ← Flask blueprints (one per feature)
│   ├── auth.py             ← Login / signup / logout
│   ├── dashboard.py        ← Dashboard + activity chart API
│   ├── visualize.py        ← File upload + chart save API
│   ├── report.py           ← Document analysis API
│   └── export.py           ← CSV / Excel / JSON / PDF download
│
├── templates/              ← Jinja2 HTML templates
└── static/                 ← CSS + JS (modular, one file per page)
```

---

## Security Implementation

### IAM Least-Privilege
- Jenkins EC2 instance uses an IAM role with permissions scoped to ECR (`ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload`) only
- No AWS access keys stored in Jenkins credentials — IAM role attached at the instance level

### Container Security
- Trivy scans every image build before ECR push — CRITICAL CVEs block the pipeline
- Base image pinned to a specific digest (not `:latest` tag) to prevent supply chain drift
- `--restart always` with no privileged mode, no host network binding

### Secrets Management
- Database credentials injected via environment variables at container runtime — never baked into the image
- Jenkins credentials store used for SSH private key — not committed to source code
- `SECRET_KEY` for Flask session signing injected at runtime via environment variable

### Application Security (Production Checklist)
- `DEBUG=False` in production
- `SESSION_COOKIE_SECURE=True` (requires HTTPS)
- Flask-Login enforces authentication on all data routes
- Form validation at both client side (JS) and server side (Flask)

---

## Local Setup

### Prerequisites
- Docker and Docker Compose installed
- Python 3.9+

### Run locally

```bash
# Clone the repo
git clone https://github.com/kushagrasharmavs/Invision-AWS-DOCKER-JENKINS.git
cd Invision-AWS-DOCKER-JENKINS

# Create virtual environment
python -m venv venv
source venv/bin/activate          # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Set environment variables
export SECRET_KEY=your-secret-key
export DATABASE_URL=sqlite:///invision.db   # or postgresql://...
export DEBUG=True

# Run
python app.py
```

Open `http://localhost:5000` in your browser.

### Run with Docker

```bash
docker build -t invision .
docker run -d -p 5000:5000 \
    -e SECRET_KEY=your-secret-key \
    -e DATABASE_URL=sqlite:///invision.db \
    invision
```

### Run tests

```bash
pytest test_app.py -v
```

---

## Infrastructure Prerequisites (AWS)

| Resource | Configuration |
|---|---|
| EC2 Instance | Ubuntu 22.04, t2.micro or t3.small, IAM role with ECR access |
| AWS ECR | Repository created in same region as EC2 |
| AWS RDS | PostgreSQL, private subnet, security group allows EC2 inbound on 5432 |
| Jenkins | Installed on EC2, GitHub webhook configured, SonarQube + Trivy installed |

---

## Environment Variables

| Variable | Description | Default |
|---|---|---|
| `SECRET_KEY` | Flask session signing key | Required |
| `DATABASE_URL` | SQLAlchemy DB URI | `sqlite:///invision.db` |
| `MAIL_USERNAME` | SMTP username for contact form | Optional |
| `MAIL_PASSWORD` | SMTP app password | Optional |
| `DEBUG` | Flask debug mode | `False` in production |

---

## Key Learnings

- Designing a multi-stage **DevSecOps pipeline** with security gates that halt broken builds
- Integrating **Trivy** for container CVE scanning as a mandatory CI gate, not an afterthought
- Configuring **SonarQube Quality Gates** that enforce code standards automatically
- Managing **AWS ECR image lifecycle** with IAM role-based authentication (no plaintext credentials)
- Setting up **SSH-based zero-touch deployment** from Jenkins to EC2 — any push deploys in under 3 minutes
- Separating compute (EC2) and data (RDS PostgreSQL) layers for production-grade resilience
- Building a modular **Flask blueprint architecture** that scales as the application grows

---

## Future Enhancements

- [ ] Migrate from EC2 to **AWS ECS Fargate** (serverless container execution)
- [ ] Add **Terraform** for infrastructure provisioning (IaC for EC2, RDS, VPC, ECR)
- [ ] Integrate **Prometheus + Grafana** for container metrics and pipeline health monitoring
- [ ] Implement **blue-green deployment** strategy to eliminate deployment downtime
- [ ] Add **OWASP Dependency-Check** for third-party library vulnerability scanning
- [ ] Set up **CloudWatch alarms** for EC2 CPU, memory, and application error rate

---

## Author

**Kushagra Sharma** — Cloud & DevOps Engineer

[![LinkedIn](https://img.shields.io/badge/LinkedIn-kushagrasharma11-blue?logo=linkedin)](https://linkedin.com/in/kushagrasharma11)
[![GitHub](https://img.shields.io/badge/GitHub-kushagrasharmavs-black?logo=github)](https://github.com/kushagrasharmavs)
[![Email](https://img.shields.io/badge/Email-kushagrasharmavs%40gmail.com-red?logo=gmail)](mailto:kushagrasharmavs@gmail.com)
