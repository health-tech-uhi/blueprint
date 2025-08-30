# **Health Tech Platform Deployment Strategy**

## **Table of Contents**
1. [Overview](#overview)
2. [Architecture Summary](#architecture-summary)
3. [Database Deployment](#database-deployment)
4. [Microservices Deployment](#microservices-deployment)
5. [Frontend Deployment](#frontend-deployment)
6. [Mobile & Desktop Clients](#mobile--desktop-clients)
7. [Networking & Security](#networking--security)
8. [Monitoring & Observability](#monitoring--observability)
9. [CI/CD Pipeline](#cicd-pipeline)
10. [Backup & Disaster Recovery](#backup--disaster-recovery)
11. [Deployment Workflows](#deployment-workflows)
12. [Resource Planning](#resource-planning)
13. [Security Considerations](#security-considerations)
14. [Troubleshooting & Maintenance](#troubleshooting--maintenance)

---

## **Overview**

This document outlines the comprehensive deployment strategy for the health tech platform, focusing on a Kubernetes-based microservices architecture with self-hosted Supabase and CDN-based frontend delivery.

### **Key Architectural Decisions**
- **Backend Communication:** Pure gRPC using Tonic framework for all microservice-to-microservice communication
- **Frontend Communication:** REST APIs from all clients (web, mobile, desktop) to Backend-for-Frontend (BFF)
- **Database:** Self-hosted Supabase within the Kubernetes cluster
- **Web Frontend:** Static assets deployed via CDN for optimal global performance
- **Mobile/Desktop:** Native apps distributed through standard channels

---

## **Architecture Summary**

```mermaid
graph TB
    subgraph "CDN Layer"
        CDN[CDN<br/>React Web App<br/>Static Assets]
    end
    
    subgraph "Client Applications"
        IOS[iOS App]
        ANDROID[Android App]
        DESKTOP[Desktop App<br/>Electron/Tauri]
    end
    
    subgraph "Kubernetes Cluster"
        subgraph "Ingress Layer"
            INGRESS[HTTPS Ingress<br/>Load Balancer]
        end
        
        subgraph "Backend Services"
            BFF[BFF Service<br/>REST API Gateway]
            IAM[IAM Service<br/>gRPC]
            ORG[Org Management<br/>gRPC]
            APT[Appointment Service<br/>gRPC]
            PAY[Payment Service<br/>gRPC]
            NOT[Notification Service<br/>gRPC]
            DISC[Discussion Forum<br/>gRPC]
        end
        
        subgraph "Data Layer"
            SUPA[Self-Hosted Supabase<br/>PostgreSQL + Auth + Storage]
            REDIS[Redis Cache]
        end
        
        subgraph "Observability"
            PROM[Prometheus]
            GRAF[Grafana]
            JAEGER[Jaeger]
            LOKI[Loki]
        end
    end
    
    subgraph "External Services"
        UHI[UHI Gateway]
        PAYMENT_GW[Payment Gateways]
        SMS[SMS/Email Providers]
    end
    
    CDN -->|HTTPS REST| INGRESS
    IOS -->|HTTPS REST| INGRESS
    ANDROID -->|HTTPS REST| INGRESS
    DESKTOP -->|HTTPS REST| INGRESS
    
    INGRESS --> BFF
    
    BFF -.->|gRPC| IAM
    BFF -.->|gRPC| ORG
    BFF -.->|gRPC| APT
    BFF -.->|gRPC| PAY
    BFF -.->|gRPC| NOT
    BFF -.->|gRPC| DISC
    
    IAM --> SUPA
    ORG --> SUPA
    APT --> SUPA
    PAY --> SUPA
    NOT --> REDIS
    DISC --> SUPA
    
    BFF --> REDIS
    
    ORG --> UHI
    PAY --> PAYMENT_GW
    NOT --> SMS
    
    PROM -.-> GRAF
    JAEGER -.-> GRAF
    LOKI -.-> GRAF
```

---

## **Database Deployment**

### **Self-Hosted Supabase Setup**

#### **Why Self-Host Supabase?**
- **Data Sovereignty:** Full control over healthcare data and compliance
- **Performance:** Co-located with microservices for low latency
- **Customization:** Ability to tune PostgreSQL and enable custom extensions
- **Cost Control:** Predictable costs and no vendor lock-in
- **Security:** Private network access and custom security policies

#### **Deployment Strategy**
```yaml
# Supabase Helm Chart Configuration
apiVersion: v1
kind: Namespace
metadata:
  name: supabase
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: supabase-postgres
  namespace: supabase
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: supabase/postgres:15.1.0.117
        env:
        - name: POSTGRES_DB
          value: "postgres"
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: username
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Gi
```

#### **Supabase Components Deployment**
1. **PostgreSQL:** Primary database with row-level security enabled
2. **Auth Service:** JWT-based authentication service
3. **Storage Service:** File upload and management
4. **Studio:** Admin interface (optional, for development)
5. **PostgREST:** Auto-generated REST API (if needed for admin tools)

#### **Security Configuration**
- Enable SSL/TLS for all connections
- Configure row-level security policies
- Set up database user roles with minimal privileges
- Enable audit logging
- Configure backup encryption

---

## **Microservices Deployment**

### **Service Architecture**
Each microservice follows the same deployment pattern with the following characteristics:

#### **Common Service Template**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {service-name}
  namespace: microservices
spec:
  replicas: 3
  selector:
    matchLabels:
      app: {service-name}
  template:
    metadata:
      labels:
        app: {service-name}
    spec:
      containers:
      - name: {service-name}
        image: healthtech/{service-name}:latest
        ports:
        - containerPort: 50051  # gRPC port
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: database-secret
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: url
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: jwt-secret
              key: secret
        livenessProbe:
          grpc:
            port: 50051
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          grpc:
            port: 50051
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: {service-name}
  namespace: microservices
spec:
  selector:
    app: {service-name}
  ports:
  - port: 50051
    targetPort: 50051
    protocol: TCP
  type: ClusterIP
```

### **Individual Services**

#### **1. Backend-for-Frontend (BFF)**
- **Purpose:** REST API gateway for frontend clients
- **Protocols:** Exposes REST endpoints, communicates via gRPC internally
- **Special Configuration:** 
  - HTTP/HTTPS ports (80/443) in addition to gRPC
  - CORS configuration for web clients
  - Rate limiting and request validation

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bff-service
  namespace: microservices
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bff-service
  template:
    metadata:
      labels:
        app: bff-service
    spec:
      containers:
      - name: bff-service
        image: healthtech/bff-service:latest
        ports:
        - containerPort: 8080  # HTTP REST API
        - containerPort: 50051 # gRPC (for internal health checks)
        env:
        - name: IAM_SERVICE_URL
          value: "iam-service:50051"
        - name: ORG_SERVICE_URL
          value: "org-service:50051"
        - name: APPOINTMENT_SERVICE_URL
          value: "appointment-service:50051"
        - name: PAYMENT_SERVICE_URL
          value: "payment-service:50051"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

#### **2. Identity & Access Management (IAM)**
- **Purpose:** User authentication, authorization, and JWT management
- **Key Features:** User registration, login, password reset, MFA

#### **3. Organization Management**
- **Purpose:** Healthcare provider and facility management
- **Key Features:** Provider registration, verification, staff management

#### **4. Appointment Service**
- **Purpose:** Appointment booking, scheduling, and management
- **Key Features:** Availability management, booking workflow, status tracking

#### **5. Payment Service**
- **Purpose:** Payment processing and billing
- **Key Features:** Payment gateway integration, invoice generation, refunds

#### **6. Notification Service**
- **Purpose:** Multi-channel notifications (email, SMS, push)
- **Key Features:** Real-time notifications, preferences management

#### **7. Discussion Forum**
- **Purpose:** Community features and patient-provider communication
- **Key Features:** Threaded discussions, moderation, search

### **Service Mesh Configuration**
Deploy Linkerd or Istio for:
- **mTLS:** Automatic mutual TLS between services
- **Traffic Management:** Load balancing, retries, circuit breakers
- **Observability:** Metrics, traces, and logs
- **Security:** Zero-trust networking

```yaml
# Linkerd Service Profile Example
apiVersion: linkerd.io/v1alpha2
kind: ServiceProfile
metadata:
  name: iam-service
  namespace: microservices
spec:
  routes:
  - name: create_user
    condition:
      method: POST
      pathRegex: /users
    responseClasses:
    - condition:
        status:
          min: 200
          max: 299
      isFailure: false
    - condition:
        status:
          min: 500
          max: 599
      isFailure: true
```

---

## **Frontend Deployment**

### **Web Application (React)**

#### **Build Process**
```bash
# Build optimized production bundle
npm run build

# Output: Static assets in build/ directory
# - HTML, CSS, JavaScript files
# - Images, fonts, and other assets
# - Service worker for PWA features
```

#### **CDN Deployment Options**

##### **Option 1: Cloudflare (Recommended)**
```bash
# Install Wrangler CLI
npm install -g wrangler

# Deploy to Cloudflare Pages
wrangler pages publish build/

# Configure custom domain
wrangler pages domain add healthtech.com
```

**Benefits:**
- Global edge locations
- Built-in DDoS protection
- Free SSL certificates
- Advanced caching rules
- Edge computing capabilities

##### **Option 2: AWS CloudFront + S3**
```bash
# Upload to S3
aws s3 sync build/ s3://healthtech-frontend-bucket/ --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation --distribution-id E1234567890 --paths "/*"
```

##### **Option 3: Vercel**
```bash
# Deploy with Vercel CLI
vercel --prod

# Or configure automatic deployments via Git integration
```

#### **Environment Configuration**
```javascript
// Environment-specific API endpoints
const config = {
  production: {
    apiUrl: 'https://api.healthtech.com',
    wsUrl: 'wss://api.healthtech.com/ws'
  },
  staging: {
    apiUrl: 'https://staging-api.healthtech.com',
    wsUrl: 'wss://staging-api.healthtech.com/ws'
  },
  development: {
    apiUrl: 'http://localhost:8080',
    wsUrl: 'ws://localhost:8080/ws'
  }
};
```

#### **Security Headers Configuration**
```yaml
# Cloudflare Workers or CDN configuration
security_headers:
  Content-Security-Policy: "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
  Strict-Transport-Security: "max-age=31536000; includeSubDomains"
  X-Frame-Options: "DENY"
  X-Content-Type-Options: "nosniff"
  Referrer-Policy: "strict-origin-when-cross-origin"
```

---

## **Mobile & Desktop Clients**

### **Mobile Applications**

#### **iOS App Deployment**
```yaml
# CI/CD Pipeline for iOS
ios_deployment:
  build:
    - name: Install dependencies
      run: pod install
    - name: Build app
      run: xcodebuild -workspace HealthTech.xcworkspace -scheme HealthTech -configuration Release
    - name: Archive
      run: xcodebuild -archivePath HealthTech.xcarchive archive
    - name: Export IPA
      run: xcodebuild -exportArchive -archivePath HealthTech.xcarchive -exportPath . -exportOptionsPlist ExportOptions.plist
  
  distribute:
    - name: Upload to App Store Connect
      run: xcrun altool --upload-app --file HealthTech.ipa --username $APPLE_ID --password $APP_PASSWORD
```

#### **Android App Deployment**
```yaml
# CI/CD Pipeline for Android
android_deployment:
  build:
    - name: Build APK/AAB
      run: ./gradlew assembleRelease bundleRelease
    - name: Sign APK/AAB
      run: jarsigner -verbose -sigalg SHA256withRSA -digestalg SHA-256 -keystore release.keystore app-release.aab alias_name
  
  distribute:
    - name: Upload to Google Play Console
      run: fastlane android deploy
```

### **Desktop Applications**

#### **Electron App**
```json
{
  "build": {
    "appId": "com.healthtech.desktop",
    "productName": "HealthTech Desktop",
    "directories": {
      "output": "dist"
    },
    "files": [
      "build/**/*",
      "node_modules/**/*",
      "main.js"
    ],
    "mac": {
      "category": "public.app-category.medical"
    },
    "win": {
      "target": "nsis"
    },
    "linux": {
      "target": "AppImage"
    }
  }
}
```

#### **Tauri App (Alternative)**
```toml
[build]
beforeBuildCommand = "npm run build"
beforeDevCommand = "npm run dev"
devPath = "http://localhost:3000"
distDir = "../build"

[tauri.bundle]
identifier = "com.healthtech.app"
```

---

## **Networking & Security**

### **Network Architecture**

#### **Ingress Configuration**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: healthtech-ingress
  namespace: microservices
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
  - hosts:
    - api.healthtech.com
    secretName: healthtech-tls
  rules:
  - host: api.healthtech.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: bff-service
            port:
              number: 8080
```

#### **Network Policies**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: microservices
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-bff-to-services
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: bff-service
    ports:
    - protocol: TCP
      port: 50051
```

### **Security Configuration**

#### **RBAC (Role-Based Access Control)**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: microservices
  name: healthtech-service-role
rules:
- apiGroups: [""]
  resources: ["secrets", "configmaps"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: healthtech-service-binding
  namespace: microservices
subjects:
- kind: ServiceAccount
  name: healthtech-service-account
  namespace: microservices
roleRef:
  kind: Role
  name: healthtech-service-role
  apiGroup: rbac.authorization.k8s.io
```

#### **Pod Security Standards**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

#### **Secrets Management**
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: microservices
spec:
  provider:
    vault:
      server: "https://vault.internal"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "healthtech-role"
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-secret
  namespace: microservices
spec:
  refreshInterval: 60s
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: database-secret
    creationPolicy: Owner
  data:
  - secretKey: url
    remoteRef:
      key: database
      property: url
```

---

## **Monitoring & Observability**

### **Metrics Collection**

#### **Prometheus Configuration**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
      - "/etc/prometheus/rules/*.yml"
    
    scrape_configs:
    - job_name: 'kubernetes-apiservers'
      kubernetes_sd_configs:
      - role: endpoints
      scheme: https
      tls_config:
        ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
      relabel_configs:
      - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: default;kubernetes;https
    
    - job_name: 'kubernetes-nodes'
      kubernetes_sd_configs:
      - role: node
      relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    
    - job_name: 'kubernetes-pods'
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
    
    - job_name: 'healthtech-services'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - microservices
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

#### **Service Monitoring**
```rust
// Rust service metrics example
use prometheus::{Counter, Histogram, Gauge, register_counter, register_histogram, register_gauge};

lazy_static! {
    static ref GRPC_REQUESTS_TOTAL: Counter = register_counter!(
        "grpc_requests_total", "Total number of gRPC requests"
    ).unwrap();
    
    static ref GRPC_REQUEST_DURATION: Histogram = register_histogram!(
        "grpc_request_duration_seconds", "gRPC request duration in seconds"
    ).unwrap();
    
    static ref ACTIVE_CONNECTIONS: Gauge = register_gauge!(
        "active_database_connections", "Number of active database connections"
    ).unwrap();
}

// Instrumentation in service methods
#[tonic::async_trait]
impl PatientService for PatientServiceImpl {
    async fn create_patient(
        &self,
        request: Request<CreatePatientRequest>,
    ) -> Result<Response<PatientResponse>, Status> {
        let _timer = GRPC_REQUEST_DURATION.start_timer();
        GRPC_REQUESTS_TOTAL.inc();
        
        // Service logic here...
        
        Ok(Response::new(response))
    }
}
```

### **Distributed Tracing**

#### **Jaeger Configuration**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:1.35
        ports:
        - containerPort: 16686  # UI
        - containerPort: 14268  # HTTP collector
        - containerPort: 14250  # gRPC collector
        env:
        - name: COLLECTOR_ZIPKIN_HOST_PORT
          value: ":9411"
        - name: SPAN_STORAGE_TYPE
          value: "elasticsearch"
        - name: ES_SERVER_URLS
          value: "http://elasticsearch:9200"
```

### **Logging**

#### **Loki Configuration**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: loki-config
  namespace: monitoring
data:
  loki.yaml: |
    auth_enabled: false
    
    server:
      http_listen_port: 3100
      grpc_listen_port: 9095
    
    ingester:
      lifecycler:
        address: 127.0.0.1
        ring:
          kvstore:
            store: inmemory
          replication_factor: 1
        final_sleep: 0s
      chunk_idle_period: 5m
      chunk_retain_period: 30s
    
    schema_config:
      configs:
      - from: 2020-05-15
        store: boltdb-shipper
        object_store: filesystem
        schema: v11
        index:
          prefix: index_
          period: 24h
    
    storage_config:
      boltdb_shipper:
        active_index_directory: /loki/boltdb-shipper-active
        cache_location: /loki/boltdb-shipper-cache
        shared_store: filesystem
      filesystem:
        directory: /loki/chunks
    
    limits_config:
      enforce_metric_name: false
      reject_old_samples: true
      reject_old_samples_max_age: 168h
```

### **Alerting Rules**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-rules
  namespace: monitoring
data:
  healthtech.yml: |
    groups:
    - name: healthtech.rules
      rules:
      - alert: HighErrorRate
        expr: rate(grpc_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Service {{ $labels.service }} has error rate > 10%"
      
      - alert: DatabaseConnectionsHigh
        expr: active_database_connections > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High database connection usage"
          description: "Database connections usage is above 80%"
      
      - alert: ServiceDown
        expr: up{job="healthtech-services"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "Service {{ $labels.service }} is down"
```

---

## **CI/CD Pipeline**

### **GitHub Actions Workflow**

#### **Backend Services Pipeline**
```yaml
name: Backend CI/CD

on:
  push:
    branches: [main, develop]
    paths: ['backend/**']
  pull_request:
    branches: [main]
    paths: ['backend/**']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: healthtech

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: password
          POSTGRES_DB: healthtech_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
    - uses: actions/checkout@v4
    
    - name: Install Rust
      uses: dtolnay/rust-toolchain@stable
      with:
        components: rustfmt, clippy
    
    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.cargo/registry
          ~/.cargo/git
          target
        key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}
    
    - name: Run tests
      run: |
        cd backend
        cargo test --all-features
      env:
        DATABASE_URL: postgres://postgres:password@localhost:5432/healthtech_test
        REDIS_URL: redis://localhost:6379
    
    - name: Run clippy
      run: |
        cd backend
        cargo clippy -- -D warnings
    
    - name: Check formatting
      run: |
        cd backend
        cargo fmt -- --check

  security-audit:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: dtolnay/rust-toolchain@stable
    - name: Install cargo-audit
      run: cargo install cargo-audit
    - name: Run security audit
      run: |
        cd backend
        cargo audit

  build-and-push:
    needs: [test, security-audit]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    strategy:
      matrix:
        service: [iam, bff, org-management, appointment, payment, notification, discussion-forum]
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Log in to Container Registry
      uses: docker/login-action@v2
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/${{ matrix.service }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v4
      with:
        context: ./backend/${{ matrix.service }}
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}

  deploy:
    needs: [build-and-push]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Install kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'v1.28.0'
    
    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBECONFIG }}" | base64 -d > $HOME/.kube/config
    
    - name: Install Helm
      uses: azure/setup-helm@v3
      with:
        version: '3.12.0'
    
    - name: Deploy services
      run: |
        helm upgrade --install healthtech-backend ./helm/healthtech-backend \
          --namespace microservices \
          --create-namespace \
          --set image.tag=${{ github.sha }} \
          --wait
```

#### **Frontend Pipeline**
```yaml
name: Frontend CI/CD

on:
  push:
    branches: [main, develop]
    paths: ['frontend/**']
  pull_request:
    branches: [main]
    paths: ['frontend/**']

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: frontend/package-lock.json
    
    - name: Install dependencies
      run: |
        cd frontend
        npm ci
    
    - name: Run tests
      run: |
        cd frontend
        npm run test:ci
    
    - name: Run linting
      run: |
        cd frontend
        npm run lint
    
    - name: Type checking
      run: |
        cd frontend
        npm run type-check

  build-and-deploy:
    needs: [test]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: frontend/package-lock.json
    
    - name: Install dependencies
      run: |
        cd frontend
        npm ci
    
    - name: Build application
      run: |
        cd frontend
        npm run build
      env:
        REACT_APP_API_URL: https://api.healthtech.com
        REACT_APP_WS_URL: wss://api.healthtech.com/ws
    
    - name: Deploy to Cloudflare Pages
      uses: cloudflare/pages-action@v1
      with:
        apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
        accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
        projectName: healthtech-frontend
        directory: frontend/build
        gitHubToken: ${{ secrets.GITHUB_TOKEN }}
```

### **Helm Charts**

#### **Backend Service Helm Chart**
```yaml
# helm/healthtech-backend/Chart.yaml
apiVersion: v2
name: healthtech-backend
description: HealthTech Backend Microservices
version: 0.1.0
appVersion: "1.0.0"

# helm/healthtech-backend/values.yaml
image:
  registry: ghcr.io
  repository: healthtech
  tag: "latest"
  pullPolicy: Always

services:
  iam:
    enabled: true
    replicas: 3
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"
  
  bff:
    enabled: true
    replicas: 3
    resources:
      requests:
        memory: "512Mi"
        cpu: "200m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
  
  org-management:
    enabled: true
    replicas: 2
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "500m"

database:
  host: "supabase-postgres.supabase.svc.cluster.local"
  port: 5432
  name: "healthtech"
  existingSecret: "database-secret"

redis:
  host: "redis.microservices.svc.cluster.local"
  port: 6379
  existingSecret: "redis-secret"

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: api.healthtech.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: healthtech-tls
      hosts:
        - api.healthtech.com

monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
    interval: 30s
```

---

## **Backup & Disaster Recovery**

### **Database Backup Strategy**

#### **Automated PostgreSQL Backups**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: supabase
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: postgres-backup
            image: postgres:15
            command:
            - /bin/bash
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              pg_dump -h $POSTGRES_HOST -U $POSTGRES_USER -d $POSTGRES_DB > /backup/backup_$TIMESTAMP.sql
              
              # Upload to S3 or other storage
              aws s3 cp /backup/backup_$TIMESTAMP.sql s3://healthtech-backups/postgres/
              
              # Clean up old local backups
              find /backup -name "backup_*.sql" -mtime +7 -delete
            env:
            - name: POSTGRES_HOST
              value: "supabase-postgres"
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: POSTGRES_DB
              value: "postgres"
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

#### **Point-in-Time Recovery Setup**
```yaml
# Enable WAL archiving for PITR
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  namespace: supabase
data:
  postgresql.conf: |
    # WAL settings for PITR
    wal_level = replica
    archive_mode = on
    archive_command = 'aws s3 cp %p s3://healthtech-backups/wal/%f'
    max_wal_senders = 3
    wal_keep_segments = 32
    
    # Performance settings
    shared_buffers = 256MB
    effective_cache_size = 1GB
    maintenance_work_mem = 64MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
```

### **Application State Backup**

#### **Redis Backup**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: redis-backup
  namespace: microservices
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: redis-backup
            image: redis:7
            command:
            - /bin/sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              redis-cli -h $REDIS_HOST --rdb /backup/redis_backup_$TIMESTAMP.rdb
              
              # Upload to S3
              aws s3 cp /backup/redis_backup_$TIMESTAMP.rdb s3://healthtech-backups/redis/
              
              # Clean up old backups
              find /backup -name "redis_backup_*.rdb" -mtime +3 -delete
            env:
            - name: REDIS_HOST
              value: "redis"
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          restartPolicy: OnFailure
```

### **Disaster Recovery Procedures**

#### **Database Recovery**
```bash
#!/bin/bash
# restore_database.sh

# Stop all services that use the database
kubectl scale deployment --replicas=0 -n microservices --all

# Restore from backup
BACKUP_FILE="backup_20231201_020000.sql"
aws s3 cp s3://healthtech-backups/postgres/$BACKUP_FILE /tmp/

# Drop and recreate database
kubectl exec -it supabase-postgres-0 -n supabase -- psql -U postgres -c "DROP DATABASE IF EXISTS healthtech;"
kubectl exec -it supabase-postgres-0 -n supabase -- psql -U postgres -c "CREATE DATABASE healthtech;"

# Restore data
kubectl exec -i supabase-postgres-0 -n supabase -- psql -U postgres -d healthtech < /tmp/$BACKUP_FILE

# Restart services
kubectl scale deployment --replicas=3 -n microservices --all
```

#### **Full Cluster Recovery**
```bash
#!/bin/bash
# cluster_recovery.sh

# 1. Restore cluster from backup
# 2. Apply base configurations
kubectl apply -f k8s/namespaces/
kubectl apply -f k8s/secrets/
kubectl apply -f k8s/configmaps/

# 3. Deploy Supabase
helm install supabase ./helm/supabase -n supabase

# 4. Restore database
./restore_database.sh

# 5. Deploy backend services
helm install healthtech-backend ./helm/healthtech-backend -n microservices

# 6. Verify all services are healthy
kubectl get pods -n microservices
kubectl get pods -n supabase
```

---

## **Resource Planning**

### **Cluster Sizing**

#### **Production Cluster Specifications**
```yaml
cluster_sizing:
  node_pools:
    system:
      count: 3
      instance_type: "n1-standard-2"  # 2 vCPU, 7.5GB RAM
      disk_size: "100GB"
      purpose: "System components, monitoring"
    
    application:
      count: 5
      instance_type: "n1-standard-4"  # 4 vCPU, 15GB RAM
      disk_size: "200GB"
      purpose: "Microservices, BFF"
    
    database:
      count: 3
      instance_type: "n1-highmem-2"   # 2 vCPU, 13GB RAM
      disk_size: "500GB SSD"
      purpose: "Supabase, Redis"
```

### **Resource Allocation**

#### **Per-Service Resource Requirements**
```yaml
resource_allocation:
  bff_service:
    requests:
      cpu: "200m"
      memory: "512Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
    replicas: 3
    
  iam_service:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
    replicas: 3
    
  org_management:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
    replicas: 2
    
  appointment_service:
    requests:
      cpu: "150m"
      memory: "384Mi"
    limits:
      cpu: "750m"
      memory: "768Mi"
    replicas: 3
    
  payment_service:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
    replicas: 2
    
  notification_service:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
    replicas: 2
    
  discussion_forum:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
    replicas: 2
    
  supabase_postgres:
    requests:
      cpu: "1000m"
      memory: "2Gi"
    limits:
      cpu: "2000m"
      memory: "4Gi"
    storage: "100Gi"
    
  redis:
    requests:
      cpu: "100m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
    storage: "10Gi"
```

### **Scaling Strategies**

#### **Horizontal Pod Autoscaler**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: bff-service-hpa
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: bff-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 2
        periodSeconds: 60
```

#### **Vertical Pod Autoscaler**
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: supabase-postgres-vpa
  namespace: supabase
spec:
  targetRef:
    apiVersion: apps/v1
    kind: StatefulSet
    name: supabase-postgres
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: postgres
      minAllowed:
        cpu: 500m
        memory: 1Gi
      maxAllowed:
        cpu: 4
        memory: 8Gi
      controlledResources: ["cpu", "memory"]
```

---

## **Security Considerations**

### **Container Security**

#### **Security Contexts**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-service
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: service
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: var-logs
          mountPath: /var/log
      volumes:
      - name: tmp
        emptyDir: {}
      - name: var-logs
        emptyDir: {}
```

#### **Image Scanning**
```yaml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    - name: Build Docker image
      run: docker build -t test-image .
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'test-image'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
```

### **Network Security**

#### **Service Mesh Security**
```yaml
# Linkerd authorization policy
apiVersion: policy.linkerd.io/v1beta1
kind: Server
metadata:
  name: iam-service-server
  namespace: microservices
spec:
  podSelector:
    matchLabels:
      app: iam-service
  port: 50051
  proxyProtocol: gRPC
---
apiVersion: policy.linkerd.io/v1beta1
kind: ServerAuthorization
metadata:
  name: iam-service-policy
  namespace: microservices
spec:
  server:
    name: iam-service-server
  client:
    meshTLS:
      serviceAccounts:
      - name: bff-service
      - name: org-service
```

#### **Egress Control**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: egress-control
  namespace: microservices
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to: []
    ports:
    - protocol: UDP
      port: 53
  # Allow access to Supabase
  - to:
    - namespaceSelector:
        matchLabels:
          name: supabase
    ports:
    - protocol: TCP
      port: 5432
  # Allow HTTPS to external services
  - to: []
    ports:
    - protocol: TCP
      port: 443
```

### **Data Protection**

#### **Encryption at Rest**
```yaml
apiVersion: v1
kind: StorageClass
metadata:
  name: encrypted-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  encrypted: "true"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

#### **Encryption in Transit**
All communication enforces TLS:
- Frontend to BFF: HTTPS
- BFF to microservices: gRPC with TLS
- Microservice to database: TLS
- Service mesh: mTLS

---

## **Deployment Workflows**

### **Environment Promotion**

#### **Development → Staging → Production**
```bash
#!/bin/bash
# deploy.sh

ENVIRONMENT=$1
VERSION=$2

case $ENVIRONMENT in
  "development")
    NAMESPACE="dev"
    REPLICAS=1
    RESOURCES="minimal"
    ;;
  "staging")
    NAMESPACE="staging"
    REPLICAS=2
    RESOURCES="medium"
    ;;
  "production")
    NAMESPACE="microservices"
    REPLICAS=3
    RESOURCES="production"
    ;;
  *)
    echo "Invalid environment: $ENVIRONMENT"
    exit 1
    ;;
esac

# Deploy backend services
helm upgrade --install healthtech-backend ./helm/healthtech-backend \
  --namespace $NAMESPACE \
  --create-namespace \
  --set image.tag=$VERSION \
  --set replicaCount=$REPLICAS \
  --set resources.profile=$RESOURCES \
  --wait

# Run health checks
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=healthtech-backend -n $NAMESPACE --timeout=300s

# Run smoke tests
./scripts/smoke-tests.sh $ENVIRONMENT
```

### **Canary Deployments**

#### **Argo Rollouts Configuration**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: bff-service-rollout
  namespace: microservices
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 2m}
      - setWeight: 40
      - pause: {duration: 2m}
      - setWeight: 60
      - pause: {duration: 2m}
      - setWeight: 80
      - pause: {duration: 2m}
      canaryService: bff-service-canary
      stableService: bff-service-stable
      trafficRouting:
        nginx:
          stableIngress: bff-service-ingress
          annotationPrefix: nginx.ingress.kubernetes.io
      analysis:
        templates:
        - templateName: success-rate
        startingStep: 1
        args:
        - name: service-name
          value: bff-service
  selector:
    matchLabels:
      app: bff-service
  template:
    metadata:
      labels:
        app: bff-service
    spec:
      containers:
      - name: bff-service
        image: healthtech/bff-service:latest
        ports:
        - containerPort: 8080
```

### **Blue-Green Deployments**

#### **Blue-Green Strategy**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-service-rollout
  namespace: microservices
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: payment-service-active
      previewService: payment-service-preview
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: payment-service-analysis
        args:
        - name: service-name
          value: payment-service-preview
      postPromotionAnalysis:
        templates:
        - templateName: payment-service-analysis
        args:
        - name: service-name
          value: payment-service-active
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      containers:
      - name: payment-service
        image: healthtech/payment-service:latest
        ports:
        - containerPort: 50051
```

---

## **Troubleshooting & Maintenance**

### **Common Issues & Solutions**

#### **Database Connection Issues**
```bash
# Check database connectivity
kubectl exec -it supabase-postgres-0 -n supabase -- pg_isready

# Check connection pool status
kubectl logs -f deployment/bff-service -n microservices | grep "connection"

# Reset connection pool
kubectl rollout restart deployment/bff-service -n microservices
```

#### **Service Discovery Issues**
```bash
# Check service endpoints
kubectl get endpoints -n microservices

# Test service connectivity
kubectl run test-pod --image=busybox -it --rm -- nslookup iam-service.microservices.svc.cluster.local

# Check gRPC health
kubectl exec -it deployment/bff-service -n microservices -- grpc_health_probe -addr=iam-service:50051
```

#### **Performance Issues**
```bash
# Check resource usage
kubectl top pods -n microservices
kubectl top nodes

# Check HPA status
kubectl get hpa -n microservices

# Check for OOM kills
kubectl describe pod -n microservices | grep -i "oom"
```

### **Maintenance Procedures**

#### **Database Maintenance**
```bash
#!/bin/bash
# db_maintenance.sh

# Weekly vacuum and analyze
kubectl exec -it supabase-postgres-0 -n supabase -- psql -U postgres -d healthtech -c "VACUUM ANALYZE;"

# Check database size
kubectl exec -it supabase-postgres-0 -n supabase -- psql -U postgres -d healthtech -c "
SELECT 
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables 
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;"

# Check slow queries
kubectl exec -it supabase-postgres-0 -n supabase -- psql -U postgres -d healthtech -c "
SELECT query, mean_time, calls, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;"
```

#### **Log Rotation and Cleanup**
```bash
#!/bin/bash
# log_cleanup.sh

# Clean up old logs (older than 30 days)
kubectl delete events --field-selector involvedObject.kind!=Pod -n microservices
kubectl delete events --field-selector involvedObject.kind!=Pod -n supabase

# Clean up completed jobs
kubectl delete jobs --field-selector status.successful=1 -n microservices

# Clean up old replica sets
kubectl delete replicasets --field-selector status.replicas=0 -n microservices
```

#### **Certificate Renewal**
```bash
#!/bin/bash
# cert_renewal.sh

# Check certificate expiration
kubectl get certificates -n microservices

# Force renewal if needed
kubectl delete secret healthtech-tls -n microservices
kubectl annotate certificate healthtech-cert -n microservices cert-manager.io/issue-temporary-certificate="true"
```

### **Monitoring and Alerting Setup**

#### **Critical Alerts**
```yaml
# Slack notification for critical alerts
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-slack
  namespace: monitoring
data:
  slack_api_url: <base64-encoded-slack-webhook-url>
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      slack_api_url_file: '/etc/secrets/slack_api_url'
    
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'web.hook'
      routes:
      - match:
          severity: critical
        receiver: 'slack-critical'
    
    receivers:
    - name: 'web.hook'
    - name: 'slack-critical'
      slack_configs:
      - channel: '#alerts'
        title: 'Critical Alert: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
```

---

## **Summary**

This deployment strategy provides a comprehensive, production-ready approach for the health tech platform with the following key characteristics:

### **Architecture Highlights**
- **Pure gRPC backend** using Tonic framework for efficient service communication
- **REST API gateway** (BFF) for frontend clients
- **Self-hosted Supabase** for data sovereignty and performance
- **CDN-delivered frontend** for global performance
- **Kubernetes-native** deployment with best practices

### **Security & Compliance**
- End-to-end encryption (TLS/mTLS)
- RBAC and network policies
- Automated security scanning
- Healthcare data protection

### **Operational Excellence**
- Comprehensive monitoring and observability
- Automated backups and disaster recovery
- CI/CD pipelines with security checks
- Scalable infrastructure with auto-scaling

### **Performance & Reliability**
- Horizontal and vertical auto-scaling
- Service mesh for traffic management
- Multiple deployment strategies (canary, blue-green)
- Health checks and circuit breakers

This strategy ensures the platform can scale from initial deployment to serving millions of users while maintaining security, compliance, and operational efficiency.
