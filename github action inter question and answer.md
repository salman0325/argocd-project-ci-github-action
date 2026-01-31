

## Part 1: Top 20 GitHub Actions Interview Questions (Easy Answers)

### 1. What is GitHub Actions?

GitHub Actions is a **CI/CD tool** provided by GitHub that helps you **build, test, and deploy** code automatically when an event happens (like push or pull request).

**Example:**
When you push code to GitHub → pipeline runs automatically.

---

### 2. What is CI/CD?

* **CI (Continuous Integration):** Automatically build and test code
* **CD (Continuous Deployment/Delivery):** Automatically deploy code

**Example:**
Push code → build → test → deploy

---

### 3. What are GitHub Actions workflows?

A workflow is an **automation file** written in **YAML** and stored in:

```
.github/workflows/
```

---

### 4. What is a workflow file?

A YAML file that defines:

* When pipeline runs
* What steps to execute

**Example:**

```yaml
on: push
jobs:
  build:
    runs-on: ubuntu-latest
```

---

### 5. What are events in GitHub Actions?

Events trigger workflows.

**Common events:**

* push
* pull_request
* workflow_dispatch

---

### 6. What is a job?

A job is a **set of steps** that run on the same runner.

---

### 7. What is a step?

A step is a **single task** inside a job.

**Example:**

* Checkout code
* Build app

---

### 8. What is a runner?

A runner is a **machine** where the workflow runs.

**Example:**

```yaml
runs-on: ubuntu-latest
```

---

### 9. What are GitHub-hosted runners?

GitHub provides free runners like:

* ubuntu-latest
* windows-latest

---

### 10. What are self-hosted runners?

Your **own server/VM** used as a runner.

---

### 11. What are GitHub Actions?

Reusable **pre-built tasks** used in workflows.

**Example:**

```yaml
uses: actions/checkout@v4
```

---

### 12. What are secrets in GitHub Actions?

Secrets store **sensitive data** securely.

**Example:**

* AWS_ACCESS_KEY_ID
* AWS_SECRET_ACCESS_KEY

Used as:

```yaml
${{ secrets.AWS_ACCESS_KEY_ID }}
```

---

### 13. Difference between environment variables and secrets?

* **Env vars:** normal values
* **Secrets:** encrypted sensitive values

---

### 14. How do you pass data between steps?

Using **outputs** or environment variables.

---

### 15. What is matrix strategy?

Run jobs for **multiple versions**.

**Example:**
Test on Node 16 and 18

---

### 16. How do you cache dependencies?

Using cache action to speed up builds.

---

### 17. How do you deploy using GitHub Actions?

By running:

* Docker commands
* kubectl
* Helm
* Argo CD sync

---

### 18. What is AWS ECR?

ECR is **Docker image registry** in AWS.

---

### 19. What is EKS?

EKS is **managed Kubernetes** by AWS.

---

### 20. What is Argo CD?

Argo CD is a **GitOps tool** that deploys apps to Kubernetes using Git repository.

---

## Part 2: Complete CI/CD Pipeline (GitHub Actions → ECR → EKS → Argo CD)

### Flow (Interview Friendly)

```
Developer Push Code
        ↓
GitHub Actions
        ↓
Build Docker Image
        ↓
Push Image to AWS ECR
        ↓
Update Image Tag in Git Repo
        ↓
Argo CD Detects Change
        ↓
Deploys App to EKS
```

---

## 1. Prerequisites

* AWS Account
* ECR Repository
* EKS Cluster
* Argo CD Installed on EKS
* GitHub Repository

---

## 2. GitHub Secrets Required

Add these in **GitHub → Settings → Secrets**

* AWS_ACCESS_KEY_ID
* AWS_SECRET_ACCESS_KEY
* AWS_REGION
* ECR_REPOSITORY
* ACCOUNT_ID

---

## 3. GitHub Actions Workflow (CI)

Create file:

```
.github/workflows/ci-cd.yml
```

```yaml
name: CI-CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Login to Amazon ECR
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build Docker image
      run: |
        docker build -t my-app .
        docker tag my-app:latest ${{ secrets.ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/${{ secrets.ECR_REPOSITORY }}:latest

    - name: Push Docker image to ECR
      run: |
        docker push ${{ secrets.ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/${{ secrets.ECR_REPOSITORY }}:latest
```

---

## 4. Kubernetes Deployment Manifest (GitOps Repo)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/my-app:latest
        ports:
        - containerPort: 80
```

---

## 5. Argo CD Application YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/gitops-repo.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```
```yml
apiVersion: argoproj.io/v1alpha1        # Argo CD ka API version
kind: Application                      # Resource type: Argo CD Application

metadata:
  name: prod-my-app                    # Production application ka naam
  namespace: argocd                    # Argo CD ka namespace (important)

spec:
  project: default                     # Argo CD project (production mein alag bhi ho sakta)

  source:
    repoURL: https://github.com/org/gitops-repo.git   # Git repo jahan k8s configs hain
    targetRevision: main               # Stable branch (prod mein main / release)
    path: k8s/prod                     # Production specific folder
    directory:
      recurse: true                    # Sub-folders ke YAML bhi read karo

  destination:
    server: https://kubernetes.default.svc  # Same cluster jahan Argo CD install hai
    namespace: prod                    # Production namespace

  syncPolicy:
    automated:
      prune: true                      # Git se delete ho → cluster se bhi delete
      selfHeal: true                   # Manual change ho → wapas Git wali state
    syncOptions:
      - CreateNamespace=true           # Namespace exist na ho to auto create
      - PruneLast=true                 # Pehle deploy, baad mein delete (safe)
      - ApplyOutOfSyncOnly=true        # Sirf changed resources apply karo

  revisionHistoryLimit: 10             # Last 10 deployments ka record rakho

  ignoreDifferences:                   # Kuch fields ko ignore karo (prod safety)
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas               # Replicas manual scale ho sakte hain

  info:
    - name: environment
      value: production                # Just info ke liye (UI mein dikhta hai)
```

---yml
```yml
name: SonarQube Scan

on:
  push:
    branches:
      - main

jobs:
  sonar:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v2
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

          ```
```
HOW TO USE TRIVY With github action

```yml
name: Trivy Scan

on:
  push:
    branches: [ main ]

jobs:
  trivy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker Image
        run: docker build -t my-app .

      - name: Trivy Image Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: my-app
          severity: HIGH,CRITICAL
```


and how to use owasp 
```yml
name: OWASP Dependency Scan

on:
  push:
    branches: [ main ]

jobs:
  owasp:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: my-app
          path: .
          format: HTML
```


## Final Interview One-Liner (Very Important)

> **"GitHub Actions handles CI (build & push image to ECR), and Argo CD handles CD (deploy image to EKS using GitOps)."**

---

## You Can Say This in Interview

> "When I push code to GitHub, GitHub Actions builds and pushes Docker image to AWS ECR. Then Argo CD automatically pulls the updated configuration from Git and deploys the application to EKS cluster."

---

