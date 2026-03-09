# ⚙️ CI/CD — Jenkins & GitHub Actions

## Core Concepts

| Concept | Meaning |
|---------|---------|
| **CI** | Merge code often, run automated tests on every push |
| **CD (Delivery)** | Automatically deliver tested code to staging |
| **CD (Deployment)** | Automatically deploy to production |
| **Pipeline** | Ordered set of automated stages |
| **Artifact** | Build output (JAR, Docker image, zip) |

---

## ⚡ GitHub Actions

### Anatomy of a Workflow
```
.github/
└── workflows/
    ├── ci.yml        # Run tests on PR
    ├── cd.yml        # Deploy on merge to main
    └── release.yml   # Tag-triggered release
```

### Full CI/CD Workflow (Docker + K8s)
```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ── Job 1: Test ─────────────────────────────
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
          cache: 'pip'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest --cov=app tests/ --cov-report=xml

      - name: Upload coverage
        uses: codecov/codecov-action@v4

  # ── Job 2: Build & Push ──────────────────────
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
      - uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=ref,event=branch

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ── Job 3: Deploy ────────────────────────────
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production

    steps:
      - uses: actions/checkout@v4

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubeconfig
        run: echo "${{ secrets.KUBECONFIG }}" | base64 -d > kubeconfig.yaml

      - name: Deploy to K8s
        env:
          KUBECONFIG: kubeconfig.yaml
          IMAGE_TAG: ${{ needs.build.outputs.image-tag }}
        run: |
          kubectl set image deployment/myapp \
            myapp=$IMAGE_TAG \
            --record
          kubectl rollout status deployment/myapp --timeout=120s
```

### Reusable Workflow (DRY)
```yaml
# .github/workflows/reusable-test.yml
on:
  workflow_call:
    inputs:
      python-version:
        required: true
        type: string
    secrets:
      TEST_SECRET:
        required: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - run: pytest
```

---

## 🔴 Jenkins

### Declarative Pipeline (Jenkinsfile)
```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'registry.example.com'
        IMAGE_NAME      = 'myapp'
        KUBECONFIG      = credentials('k8s-kubeconfig')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    python -m venv venv
                    source venv/bin/activate
                    pip install -r requirements.txt
                    pytest tests/ -v --junitxml=results.xml
                '''
            }
            post {
                always {
                    junit 'results.xml'
                }
            }
        }
        
        stage('Build Image') {
            steps {
                script {
                    def imageTag = "${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                    
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-creds') {
                        sh "docker push ${imageTag}"
                    }
                }
            }
        }
        
        stage('Deploy') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    kubectl set image deployment/myapp \
                      myapp=${DOCKER_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
                    kubectl rollout status deployment/myapp
                """
            }
        }
    }
    
    post {
        failure {
            slackSend channel: '#alerts',
                      color: 'danger',
                      message: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        success {
            slackSend channel: '#deploys',
                      color: 'good',
                      message: "Deployed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

---

## 🛠️ Hands-On Project: Push-to-Deploy Pipeline

**Flow:** `git push` → GitHub Actions → Docker build → push to GHCR → K8s rolling update → Slack alert

📁 See [`examples/`](./examples/) for complete working workflow files.

---

## 🎯 Key Interview Questions

1. **What's the difference between a GitHub Actions job and a step?**  
   A job runs on a runner; steps are sequential commands within a job. Jobs run in parallel by default; use `needs` to serialize.

2. **How do you pass secrets securely in pipelines?**  
   Store in GitHub Secrets or Jenkins Credentials manager. Never echo secrets; they're masked in logs.

3. **What is a matrix strategy in GitHub Actions?**  
   Runs the same job across multiple configurations (e.g., Python 3.9/3.10/3.11) in parallel.

4. **How do you implement rollback in a Jenkins pipeline?**  
   Use `kubectl rollout undo` or redeploy the previous Docker image tag. Store the last known good tag as an artifact.

5. **Difference between CI and CD?**  
   CI = build and test on every commit. CD = automated delivery/deployment after CI passes.
