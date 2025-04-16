# Java App on GKE with CI/CD via GitHub Actions

This guide walks you through deploying a Spring Boot Java application on Google Kubernetes Engine (GKE) using Docker, GitHub Actions, and securing it with HTTPS via Ingress.

---

## 🚀 Prerequisites
- Google Cloud Project
- GKE cluster created and `kubectl` configured
- Docker installed
- Maven installed
- GitHub repository for your app

---

## 🧱 Step-by-Step Setup

### 1. Clone the Repository
```bash
git clone https://github.com/your-username/gke-java-app.git
cd gke-java-app
```

### 2. Build Java App
```bash
mvn clean install
```
This creates a `.jar` in the `target/` folder.

### 3. Dockerfile (already present)
Ensure you have:
```Dockerfile
FROM registry.access.redhat.com/ubi8/ubi-minimal:8.5
RUN microdnf install --nodocs java-11-openjdk-headless && microdnf clean all
WORKDIR /work/
COPY target/*.jar /work/application.jar
EXPOSE 8080
CMD ["java", "-jar", "application.jar"]
```

### 4. Build and Push Docker Image
```bash
docker build -t us-east1-docker.pkg.dev/PROJECT_ID/REPO_NAME/java-app .
docker push us-east1-docker.pkg.dev/PROJECT_ID/REPO_NAME/java-app
```

### 5. IAM Permissions for GKE to Pull Image
```bash
gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"
```

### 6. Deploy to GKE
#### `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-gke-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: java-gke
  template:
    metadata:
      labels:
        app: java-gke
    spec:
      containers:
      - name: app
        image: us-east1-docker.pkg.dev/PROJECT_ID/REPO_NAME/java-app
        ports:
        - containerPort: 8080
```

#### `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: java-gke-service
spec:
  selector:
    app: java-gke
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: NodePort
```

### 7. Create TLS Secret
Use your domain's certificate and key:
```bash
kubectl create secret tls java-tls-secret \
  --cert=fullchain.pem \
  --key=privkey.pem
```

### 8. Ingress with HTTPS
#### `ingress.yaml`
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: java-app-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"  # for GKE Ingress controller
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-gke-app
            port:
              number: 8080
```

---

## 🔐 GitHub Actions Secrets
Set these in your repo under `Settings > Secrets and variables > Actions`:

| Name                    | Description                                 |
|-------------------------|---------------------------------------------|
| `GCP_SA_KEY`            | Base64 encoded service account JSON key    |

---

## 🧚‍♂️ GitHub Actions CI/CD Workflow
`.github/workflows/ci-cd.yaml`
```yaml
name: Build and Deploy to GKE

on:
  push:
    branches:
      - main

env:
  PROJECT_ID: splendid-map-423700-r8
  REGION: us-east1
  REPO_NAME: obiwan
  IMAGE_NAME: java-app
  CLUSTER_NAME: autopilot-cluster-1
  CLUSTER_ZONE: us-east1

jobs:
  build-deploy:
    name: Build & Deploy to GKE
    runs-on: ubuntu-latest

    steps:
    - name: Checkout source code
      uses: actions/checkout@v3

    - name: Set up JDK 11
      uses: actions/setup-java@v3
      with:
        java-version: '11'
        distribution: 'temurin'

    - name: Set up Maven cache
      uses: actions/cache@v3
      with:
        path: ~/.m2
        key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
        restore-keys: ${{ runner.os }}-maven

    - name: Build the app with Maven
      run: mvn clean package --no-transfer-progress

    - name: Authenticate with Google Cloud
      uses: google-github-actions/auth@v2
      with:
        credentials_json: '${{ secrets.GCP_SA_KEY }}'

    - name: Set up gcloud SDK
      uses: google-github-actions/setup-gcloud@v1
      with:
        project_id: ${{ env.PROJECT_ID }}

    - name: Configure Docker for Artifact Registry
      run: gcloud auth configure-docker ${{ env.REGION }}-docker.pkg.dev --quiet

    - name: Build Docker image
      run: |
        docker build -t ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPO_NAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }} .

    - name: Push Docker image
      run: |
        docker push ${{ env.REGION }}-docker.pkg.dev/${{ env.PROJECT_ID }}/${{ env.REPO_NAME }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

    - name: Install gke-gcloud-auth-plugin
      run: gcloud components install gke-gcloud-auth-plugin --quiet

    - name: Get GKE credentials
      run: |
        gcloud container clusters get-credentials ${{ env.CLUSTER_NAME }} --zone ${{ env.CLUSTER_ZONE }}

    - name: Deploy to GKE
      run: |
        kubectl apply -f manifest/deployment.yaml
        kubectl apply -f manifest/service.yaml
        kubectl apply -f manifest/ingress.yaml
```

---

## ✅ Validation
1. Check pod status:
```bash
kubectl get pods
```

2. Get external IP:
```bash
kubectl get ingress
```
Visit `https://external-ip`.

3. Logs:
```bash
kubectl logs -l app=java-gke
```

---

## 📦 Cleanup
```bash
gcloud container clusters delete GKE_CLUSTER_NAME --zone GKE_ZONE
```

---

Feel free to fork and customize for your use case!

