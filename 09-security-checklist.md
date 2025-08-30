# Section 9: Security Checklist (ENFORCED ACROSS THE STACK)

## Table of Contents
1. [Authentication & Authorization](#authentication--authorization)
2. [API Security](#api-security)
3. [Frontend Security](#frontend-security)
4. [Backend Security](#backend-security)
5. [Database Security](#database-security)
6. [Infrastructure Security](#infrastructure-security)
7. [Data Protection & Privacy](#data-protection--privacy)
8. [Monitoring & Incident Response](#monitoring--incident-response)
9. [Compliance & Auditing](#compliance--auditing)
10. [Security Testing](#security-testing)

---

## Authentication & Authorization

### 1. ✅ Use Battle-Tested Auth Library (CRITICAL)

**Rule: NEVER hand-roll authentication systems**

```typescript
// ✅ REQUIRED: Use proven auth solutions like Clerk, Auth0, or AWS Cognito
import { ClerkProvider, SignIn, SignUp, useAuth } from '@clerk/nextjs';

// Clerk integration for healthcare platform
const HealthTechApp = () => {
  return (
    <ClerkProvider
      publishableKey={process.env.NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY}
      appearance={{
        elements: {
          formButtonPrimary: 'bg-blue-600 hover:bg-blue-700',
          card: 'shadow-lg border border-gray-200',
        },
      }}
    >
      <Router>
        <Routes>
          <Route path="/sign-in" element={<SignIn />} />
          <Route path="/sign-up" element={<SignUp />} />
          <Route path="/dashboard" element={<ProtectedDashboard />} />
        </Routes>
      </Router>
    </ClerkProvider>
  );
};

// ❌ FORBIDDEN: Custom authentication implementation
// const customLogin = (username, password) => {
//   const hashedPassword = md5(password); // INSECURE!
//   // Custom JWT creation, session management, etc.
// };

// ✅ REQUIRED: Proper auth hooks usage
const ProtectedDashboard = () => {
  const { isLoaded, isSignedIn, user } = useAuth();
  
  if (!isLoaded) return <LoadingSpinner />;
  if (!isSignedIn) return <Redirect to="/sign-in" />;
  
  return <Dashboard user={user} />;
};
```

### 2. ✅ Multi-Factor Authentication (MFA)

```typescript
// ✅ REQUIRED: Enable MFA for all healthcare users
const enableMFAForUser = async (userId: string) => {
  // Clerk MFA setup
  await clerk.users.updateUser(userId, {
    totpEnabled: true,
    backupCodes: generateBackupCodes(),
  });
  
  // Send setup instructions
  await sendMFASetupEmail(userId);
};

// ✅ REQUIRED: Enforce MFA for sensitive operations
const handleSensitiveOperation = async (operation: string) => {
  const { user } = useAuth();
  
  if (!user?.totpEnabled) {
    throw new Error('MFA required for this operation');
  }
  
  // Verify MFA token before proceeding
  const mfaVerified = await verifyMFAToken(user.id, mfaToken);
  if (!mfaVerified) {
    throw new Error('Invalid MFA token');
  }
  
  // Proceed with operation
  await executeSensitiveOperation(operation);
};
```

### 3. ✅ Role-Based Access Control (RBAC)

```typescript
// ✅ REQUIRED: Define granular roles and permissions
enum UserRole {
  PATIENT = 'patient',
  PROVIDER = 'provider', 
  NURSE = 'nurse',
  ADMIN = 'admin',
  SUPER_ADMIN = 'super_admin',
  SUPPORT = 'support',
}

enum Permission {
  READ_PATIENTS = 'read:patients',
  WRITE_PATIENTS = 'write:patients',
  DELETE_PATIENTS = 'delete:patients',
  READ_APPOINTMENTS = 'read:appointments',
  WRITE_APPOINTMENTS = 'write:appointments',
  ACCESS_BILLING = 'access:billing',
  MANAGE_USERS = 'manage:users',
  VIEW_ANALYTICS = 'view:analytics',
  SYSTEM_CONFIG = 'system:config',
}

const ROLE_PERMISSIONS: Record<UserRole, Permission[]> = {
  [UserRole.PATIENT]: [
    Permission.READ_PATIENTS, // Own data only
    Permission.READ_APPOINTMENTS, // Own appointments only
  ],
  [UserRole.PROVIDER]: [
    Permission.READ_PATIENTS,
    Permission.WRITE_PATIENTS,
    Permission.READ_APPOINTMENTS,
    Permission.WRITE_APPOINTMENTS,
    Permission.VIEW_ANALYTICS,
  ],
  [UserRole.NURSE]: [
    Permission.READ_PATIENTS,
    Permission.WRITE_PATIENTS,
    Permission.READ_APPOINTMENTS,
  ],
  [UserRole.ADMIN]: [
    Permission.READ_PATIENTS,
    Permission.WRITE_PATIENTS,
    Permission.DELETE_PATIENTS,
    Permission.READ_APPOINTMENTS,
    Permission.WRITE_APPOINTMENTS,
    Permission.ACCESS_BILLING,
    Permission.VIEW_ANALYTICS,
    Permission.MANAGE_USERS,
  ],
  [UserRole.SUPER_ADMIN]: Object.values(Permission),
  [UserRole.SUPPORT]: [
    Permission.READ_PATIENTS,
    Permission.READ_APPOINTMENTS,
    Permission.VIEW_ANALYTICS,
  ],
};

// ✅ REQUIRED: Permission-based component protection
const ProtectedComponent = ({ 
  requiredPermission, 
  children 
}: { 
  requiredPermission: Permission; 
  children: React.ReactNode; 
}) => {
  const { user } = useAuth();
  
  const hasPermission = user?.publicMetadata?.permissions?.includes(requiredPermission);
  
  if (!hasPermission) {
    return <UnauthorizedMessage />;
  }
  
  return <>{children}</>;
};

// Usage
const PatientManagement = () => (
  <ProtectedComponent requiredPermission={Permission.WRITE_PATIENTS}>
    <PatientForm />
  </ProtectedComponent>
);
```

---

## API Security

### 2. ✅ Lock Down Protected Endpoints (CRITICAL)

**Rule: Every request must be identity-checked**

```rust
// ✅ REQUIRED: Rust backend middleware for authentication
use axum::{
    extract::{Request, State},
    http::{StatusCode, HeaderMap},
    middleware::Next,
    response::Response,
};
use jsonwebtoken::{decode, DecodingKey, Validation, Algorithm};

pub async fn auth_middleware(
    State(state): State<AppState>,
    headers: HeaderMap,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    // Extract Bearer token
    let auth_header = headers
        .get("Authorization")
        .and_then(|header| header.to_str().ok())
        .and_then(|header| header.strip_prefix("Bearer "));
        
    let token = auth_header.ok_or(StatusCode::UNAUTHORIZED)?;
    
    // Verify JWT token
    let token_data = decode::<Claims>(
        token,
        &DecodingKey::from_secret(state.config.jwt_secret.as_ref()),
        &Validation::new(Algorithm::HS256),
    ).map_err(|_| StatusCode::UNAUTHORIZED)?;
    
    // Check token expiration
    let now = chrono::Utc::now().timestamp();
    if token_data.claims.exp < now {
        return Err(StatusCode::UNAUTHORIZED);
    }
    
    // Check if user is active
    let user = state.user_service
        .get_user_by_id(&token_data.claims.sub)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?
        .ok_or(StatusCode::UNAUTHORIZED)?;
        
    if !user.is_active {
        return Err(StatusCode::FORBIDDEN);
    }
    
    // Add user info to request
    let mut request = request;
    request.extensions_mut().insert(user);
    
    Ok(next.run(request).await?)
}

// ✅ REQUIRED: Rate limiting per user
use governor::{Quota, RateLimiter, DefaultKeyedRateLimiter};

pub async fn rate_limit_middleware(
    headers: HeaderMap,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let user_id = get_user_id_from_token(&headers)
        .unwrap_or_else(|| "anonymous".to_string());
    
    // Different limits for different users
    let quota = match get_user_role(&user_id).await {
        UserRole::Patient => Quota::per_minute(60),   // 60 requests/min
        UserRole::Provider => Quota::per_minute(120), // 120 requests/min
        UserRole::Admin => Quota::per_minute(300),    // 300 requests/min
        _ => Quota::per_minute(30),                    // 30 requests/min for unknown
    };
    
    let limiter = RateLimiter::keyed(quota);
    
    match limiter.check_key(&user_id) {
        Ok(_) => Ok(next.run(request).await?),
        Err(_) => Err(StatusCode::TOO_MANY_REQUESTS),
    }
}

// ✅ REQUIRED: Input validation for all endpoints
use validator::Validate;

#[derive(Deserialize, Validate)]
pub struct CreatePatientRequest {
    #[validate(length(min = 2, max = 100))]
    pub name: String,
    
    #[validate(email)]
    pub email: String,
    
    #[validate(regex = "PHONE_REGEX")]
    pub phone: String,
    
    #[validate(regex = "ABHA_ID_REGEX")]
    pub abha_id: String,
}

pub async fn create_patient(
    State(state): State<AppState>,
    Json(request): Json<CreatePatientRequest>,
) -> Result<Json<PatientResponse>, ApiError> {
    // Validate input
    request.validate()
        .map_err(|e| ApiError::ValidationError(e.to_string()))?;
    
    // Sanitize input
    let sanitized_request = CreatePatientRequest {
        name: sanitize_html(&request.name),
        email: request.email.to_lowercase().trim().to_string(),
        phone: sanitize_phone(&request.phone),
        abha_id: validate_abha_id(&request.abha_id)?,
    };
    
    // Create patient
    let patient = state.patient_service
        .create_patient(sanitized_request)
        .await?;
        
    Ok(Json(patient.into()))
}
```

### 3. ✅ API Security Headers

```rust
// ✅ REQUIRED: Security headers middleware
use axum::http::HeaderValue;

pub async fn security_headers_middleware(
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let mut response = next.run(request).await?;
    
    let headers = response.headers_mut();
    
    // Prevent XSS attacks
    headers.insert(
        "X-XSS-Protection",
        HeaderValue::from_static("1; mode=block"),
    );
    
    // Prevent clickjacking
    headers.insert(
        "X-Frame-Options",
        HeaderValue::from_static("DENY"),
    );
    
    // Prevent MIME type sniffing
    headers.insert(
        "X-Content-Type-Options",
        HeaderValue::from_static("nosniff"),
    );
    
    // Force HTTPS
    headers.insert(
        "Strict-Transport-Security",
        HeaderValue::from_static("max-age=31536000; includeSubDomains; preload"),
    );
    
    // Content Security Policy
    headers.insert(
        "Content-Security-Policy",
        HeaderValue::from_static(
            "default-src 'self'; \
             script-src 'self' 'unsafe-inline' https://clerk.healthtech.com; \
             style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; \
             font-src 'self' https://fonts.gstatic.com; \
             img-src 'self' data: https:; \
             connect-src 'self' https://api.healthtech.com wss://realtime.healthtech.com;"
        ),
    );
    
    // Referrer Policy
    headers.insert(
        "Referrer-Policy",
        HeaderValue::from_static("strict-origin-when-cross-origin"),
    );
    
    Ok(response)
}
```

---

## Frontend Security

### 3. ✅ Never Expose Secrets on Frontend (CRITICAL)

**Rule: All secrets live server-side only**

```typescript
// ❌ FORBIDDEN: Exposing secrets in frontend
// const API_KEY = 'sk_live_abc123...'; // NEVER!
// const DATABASE_URL = 'postgresql://...'; // NEVER!
// const JWT_SECRET = 'secret123'; // NEVER!

// ✅ REQUIRED: Use environment variables properly
// .env.local (for Next.js)
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_... // Public keys only
NEXT_PUBLIC_APP_URL=https://healthtech.com
NEXT_PUBLIC_SENTRY_DSN=https://...

// Backend environment variables (NEVER prefix with NEXT_PUBLIC_)
DATABASE_URL=postgresql://...
JWT_SECRET=super_secure_secret
ENCRYPTION_KEY=...
STRIPE_SECRET_KEY=sk_live_...

// ✅ REQUIRED: Secure API configuration
const apiConfig = {
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
};

// ✅ REQUIRED: Token management
const useSecureAuth = () => {
  const { getToken } = useAuth();
  
  const makeAuthenticatedRequest = async (url: string, options: RequestInit = {}) => {
    const token = await getToken();
    
    return fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        'Authorization': `Bearer ${token}`,
        'X-Requested-With': 'XMLHttpRequest', // CSRF protection
      },
    });
  };
  
  return { makeAuthenticatedRequest };
};
```

### 4. ✅ Git-Ignore Sensitive Files (CRITICAL)

```gitignore
# ✅ REQUIRED: .gitignore configuration
# Environment variables
.env
.env.local
.env.development.local
.env.test.local
.env.production.local

# API keys and secrets
.secrets/
*.pem
*.key
*.p12
*.pfx

# Database
*.db
*.sqlite
*.sqlite3

# Logs
logs
*.log
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Coverage directory used by tools like istanbul
coverage/
*.lcov

# Build outputs
dist/
build/
.next/
.nuxt/
.vuepress/dist

# Cache
.cache/
.parcel-cache/
.eslintcache

# Local development
.vscode/settings.json
.idea/

# OS files
.DS_Store
Thumbs.db

# Backup files
*.backup
*.bak
*.tmp
*.swp
```

### 5. ✅ Content Security Policy (CSP)

```typescript
// ✅ REQUIRED: Strict CSP headers
const cspHeader = `
  default-src 'self';
  script-src 'self' 'unsafe-eval' 'unsafe-inline' https://clerk.com https://js.stripe.com;
  style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
  img-src 'self' blob: data: https:;
  font-src 'self' https://fonts.gstatic.com;
  object-src 'none';
  base-uri 'self';
  form-action 'self';
  frame-ancestors 'none';
  upgrade-insecure-requests;
`;

// Next.js configuration
const nextConfig = {
  async headers() {
    return [
      {
        source: '/(.*)',
        headers: [
          {
            key: 'Content-Security-Policy',
            value: cspHeader.replace(/\s{2,}/g, ' ').trim(),
          },
        ],
      },
    ];
  },
};
```

---

## Backend Security

### 5. ✅ Sanitize Error Messages (CRITICAL)

**Rule: Never reveal internal logic to clients**

```rust
// ✅ REQUIRED: Sanitized error responses
#[derive(Debug, thiserror::Error)]
pub enum InternalError {
    #[error("Database connection failed: {0}")]
    DatabaseConnection(String),
    
    #[error("Authentication service unavailable: {0}")]
    AuthService(String),
    
    #[error("Invalid SQL query: {0}")]
    SqlError(String),
    
    #[error("Configuration error: {0}")]
    Config(String),
}

#[derive(Debug, thiserror::Error)]
pub enum PublicError {
    #[error("The requested resource was not found")]
    NotFound,
    
    #[error("You are not authorized to access this resource")]
    Unauthorized,
    
    #[error("The provided data is invalid")]
    BadRequest,
    
    #[error("A temporary service issue occurred. Please try again later")]
    ServiceUnavailable,
    
    #[error("An unexpected error occurred. Please contact support")]
    InternalServerError,
}

impl From<InternalError> for PublicError {
    fn from(internal_error: InternalError) -> Self {
        // Log internal error details for debugging
        error!("Internal error: {:?}", internal_error);
        
        // Send internal error to monitoring service
        send_error_to_monitoring(internal_error);
        
        // Return generic public error
        match internal_error {
            InternalError::DatabaseConnection(_) => PublicError::ServiceUnavailable,
            InternalError::AuthService(_) => PublicError::ServiceUnavailable,
            InternalError::SqlError(_) => PublicError::InternalServerError,
            InternalError::Config(_) => PublicError::InternalServerError,
        }
    }
}

// ✅ REQUIRED: Error response structure
#[derive(Serialize)]
pub struct ErrorResponse {
    pub error: String,
    pub error_code: String,
    pub request_id: String,
    pub timestamp: DateTime<Utc>,
}

impl IntoResponse for PublicError {
    fn into_response(self) -> Response {
        let (status, error_code) = match self {
            PublicError::NotFound => (StatusCode::NOT_FOUND, "RESOURCE_NOT_FOUND"),
            PublicError::Unauthorized => (StatusCode::UNAUTHORIZED, "UNAUTHORIZED"),
            PublicError::BadRequest => (StatusCode::BAD_REQUEST, "INVALID_REQUEST"),
            PublicError::ServiceUnavailable => (StatusCode::SERVICE_UNAVAILABLE, "SERVICE_UNAVAILABLE"),
            PublicError::InternalServerError => (StatusCode::INTERNAL_SERVER_ERROR, "INTERNAL_ERROR"),
        };
        
        let error_response = ErrorResponse {
            error: self.to_string(),
            error_code: error_code.to_string(),
            request_id: get_request_id().unwrap_or_default(),
            timestamp: Utc::now(),
        };
        
        (status, Json(error_response)).into_response()
    }
}
```

### 6. ✅ Input Validation & Sanitization

```rust
// ✅ REQUIRED: Comprehensive input validation
use regex::Regex;
use lazy_static::lazy_static;

lazy_static! {
    static ref ABHA_ID_REGEX: Regex = Regex::new(r"^\d{2}-\d{4}-\d{4}-\d{4}$").unwrap();
    static ref PHONE_REGEX: Regex = Regex::new(r"^\+91-\d{10}$").unwrap();
    static ref SAFE_STRING_REGEX: Regex = Regex::new(r"^[a-zA-Z0-9\s\-_.,!?()]+$").unwrap();
}

pub fn validate_abha_id(abha_id: &str) -> Result<String, ValidationError> {
    if !ABHA_ID_REGEX.is_match(abha_id) {
        return Err(ValidationError::new("Invalid ABHA ID format"));
    }
    
    // Additional ABHA ID validation logic
    if !is_valid_abha_checksum(abha_id) {
        return Err(ValidationError::new("Invalid ABHA ID checksum"));
    }
    
    Ok(abha_id.to_string())
}

pub fn sanitize_html(input: &str) -> String {
    // Remove all HTML tags and entities
    ammonia::clean(input)
}

pub fn sanitize_sql_input(input: &str) -> String {
    // Escape SQL special characters
    input
        .replace("'", "''")
        .replace("\\", "\\\\")
        .replace("%", "\\%")
        .replace("_", "\\_")
}

pub fn validate_file_upload(
    filename: &str,
    content_type: &str,
    file_size: u64,
) -> Result<(), ValidationError> {
    // Check file extension
    let allowed_extensions = ["jpg", "jpeg", "png", "pdf", "doc", "docx"];
    let extension = filename
        .split('.')
        .last()
        .ok_or_else(|| ValidationError::new("File must have an extension"))?
        .to_lowercase();
        
    if !allowed_extensions.contains(&extension.as_str()) {
        return Err(ValidationError::new("File type not allowed"));
    }
    
    // Check MIME type
    let allowed_mime_types = [
        "image/jpeg", "image/png", "application/pdf",
        "application/msword", "application/vnd.openxmlformats-officedocument.wordprocessingml.document"
    ];
    
    if !allowed_mime_types.contains(&content_type) {
        return Err(ValidationError::new("Invalid file type"));
    }
    
    // Check file size (10MB limit)
    const MAX_FILE_SIZE: u64 = 10 * 1024 * 1024;
    if file_size > MAX_FILE_SIZE {
        return Err(ValidationError::new("File too large"));
    }
    
    Ok(())
}
```

---

## Database Security

### 8. ✅ Use Secure DB Libraries/Platforms (CRITICAL)

**Rule: Avoid raw SQL, use ORMs with built-in security**

```rust
// ✅ REQUIRED: Use SQLx with compile-time checked queries
use sqlx::{PgPool, query_as, query};

// ✅ GOOD: Parameterized queries prevent SQL injection
pub async fn get_patient_by_id(
    pool: &PgPool,
    patient_id: &str,
    requesting_user_id: &str,
) -> Result<Option<Patient>, sqlx::Error> {
    // Row-level security check
    let patient = query_as!(
        Patient,
        r#"
        SELECT p.* FROM patients p
        WHERE p.id = $1 
        AND (
            p.id = $2 OR  -- Patient can access own data
            EXISTS (
                SELECT 1 FROM healthcare_providers hp 
                WHERE hp.user_id = $2 AND hp.verified = true
            ) OR  -- Verified providers can access
            EXISTS (
                SELECT 1 FROM users u 
                WHERE u.id = $2 AND u.role IN ('admin', 'nurse')
            )  -- Admin/nurse can access
        )
        "#,
        patient_id,
        requesting_user_id
    )
    .fetch_optional(pool)
    .await?;
    
    Ok(patient)
}

// ✅ REQUIRED: Database connection with SSL
pub async fn create_secure_db_pool(database_url: &str) -> Result<PgPool, sqlx::Error> {
    let pool = PgPoolOptions::new()
        .max_connections(20)
        .acquire_timeout(Duration::from_secs(30))
        .idle_timeout(Duration::from_secs(600))
        .max_lifetime(Duration::from_secs(1800))
        .test_before_acquire(true)
        .connect_with(
            PgConnectOptions::from_str(database_url)?
                .ssl_mode(PgSslMode::Require) // Force SSL
                .statement_cache_capacity(100)
        )
        .await?;
    
    Ok(pool)
}

// ✅ REQUIRED: Database audit logging
pub async fn audit_database_operation(
    pool: &PgPool,
    operation: &str,
    table_name: &str,
    record_id: &str,
    user_id: &str,
    changes: Option<serde_json::Value>,
) -> Result<(), sqlx::Error> {
    query!(
        r#"
        INSERT INTO audit_log (
            operation, table_name, record_id, user_id, 
            changes, ip_address, user_agent, timestamp
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, NOW())
        "#,
        operation,
        table_name,
        record_id,
        user_id,
        changes,
        get_client_ip(),
        get_user_agent()
    )
    .execute(pool)
    .await?;
    
    Ok(())
}
```

### 9. ✅ Row-Level Security (RLS)

```sql
-- ✅ REQUIRED: Enable RLS on all sensitive tables
ALTER TABLE patients ENABLE ROW LEVEL SECURITY;
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;
ALTER TABLE medical_records ENABLE ROW LEVEL SECURITY;
ALTER TABLE billing_records ENABLE ROW LEVEL SECURITY;

-- ✅ REQUIRED: RLS policies for patients table
CREATE POLICY patients_own_data ON patients
    FOR ALL
    TO authenticated
    USING (id = auth.user_id());

CREATE POLICY patients_provider_access ON patients
    FOR SELECT
    TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM healthcare_providers hp
            JOIN appointments a ON a.provider_id = hp.id
            WHERE a.patient_id = patients.id
            AND hp.user_id = auth.user_id()
            AND hp.verified = true
        )
    );

CREATE POLICY patients_admin_access ON patients
    FOR ALL
    TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM users u
            WHERE u.id = auth.user_id()
            AND u.role IN ('admin', 'super_admin')
        )
    );

-- ✅ REQUIRED: RLS policies for appointments table
CREATE POLICY appointments_patient_access ON appointments
    FOR ALL
    TO authenticated
    USING (patient_id = auth.user_id());

CREATE POLICY appointments_provider_access ON appointments
    FOR ALL
    TO authenticated
    USING (
        EXISTS (
            SELECT 1 FROM healthcare_providers hp
            WHERE hp.id = appointments.provider_id
            AND hp.user_id = auth.user_id()
            AND hp.verified = true
        )
    );

-- ✅ REQUIRED: Encryption for sensitive fields
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Function to encrypt sensitive data
CREATE OR REPLACE FUNCTION encrypt_sensitive_data(data TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN encode(
        encrypt(
            data::bytea,
            current_setting('app.encryption_key')::bytea,
            'aes'
        ),
        'base64'
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Function to decrypt sensitive data
CREATE OR REPLACE FUNCTION decrypt_sensitive_data(encrypted_data TEXT)
RETURNS TEXT AS $$
BEGIN
    RETURN convert_from(
        decrypt(
            decode(encrypted_data, 'base64'),
            current_setting('app.encryption_key')::bytea,
            'aes'
        ),
        'UTF8'
    );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

---

## Infrastructure Security

### 9. ✅ Host on Secure Platform (CRITICAL)

**Rule: Use platforms with built-in security features**

```yaml
# ✅ REQUIRED: Kubernetes security configuration
apiVersion: v1
kind: Pod
metadata:
  name: healthtech-backend
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    runAsGroup: 1001
    fsGroup: 1001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: healthtech-backend
    image: healthtech/backend:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m"
      requests:
        memory: "256Mi"
        cpu: "100m"
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

---
# ✅ REQUIRED: Network policies
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: healthtech-network-policy
spec:
  podSelector:
    matchLabels:
      app: healthtech
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: nginx-ingress
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
```

### 10. ✅ Enable HTTPS Everywhere (CRITICAL)

```yaml
# ✅ REQUIRED: Ingress with TLS termination
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: healthtech-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/ssl-protocols: "TLSv1.2 TLSv1.3"
    nginx.ingress.kubernetes.io/ssl-ciphers: "ECDHE-RSA-AES128-GCM-SHA256,ECDHE-RSA-AES256-GCM-SHA384"
    nginx.ingress.kubernetes.io/hsts: "true"
    nginx.ingress.kubernetes.io/hsts-max-age: "31536000"
    nginx.ingress.kubernetes.io/hsts-include-subdomains: "true"
spec:
  tls:
  - hosts:
    - api.healthtech.com
    - app.healthtech.com
    secretName: healthtech-tls
  rules:
  - host: api.healthtech.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: healthtech-backend
            port:
              number: 8080
  - host: app.healthtech.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: healthtech-frontend
            port:
              number: 3000
```

### 11. ✅ File Upload Security (CRITICAL)

```rust
// ✅ REQUIRED: Secure file upload handling
use clamav_rs::{ClamAV, ScanResult};
use uuid::Uuid;

pub struct SecureFileUpload {
    antivirus: ClamAV,
    allowed_types: Vec<String>,
    max_file_size: u64,
}

impl SecureFileUpload {
    pub fn new() -> Result<Self, Box<dyn std::error::Error>> {
        Ok(Self {
            antivirus: ClamAV::new()?,
            allowed_types: vec![
                "image/jpeg".to_string(),
                "image/png".to_string(),
                "application/pdf".to_string(),
            ],
            max_file_size: 10 * 1024 * 1024, // 10MB
        })
    }
    
    pub async fn process_upload(
        &self,
        file_data: &[u8],
        filename: &str,
        content_type: &str,
        user_id: &str,
    ) -> Result<UploadResult, UploadError> {
        // Validate file size
        if file_data.len() as u64 > self.max_file_size {
            return Err(UploadError::FileTooLarge);
        }
        
        // Validate content type
        if !self.allowed_types.contains(&content_type.to_string()) {
            return Err(UploadError::InvalidFileType);
        }
        
        // Validate file extension
        let extension = filename
            .split('.')
            .last()
            .ok_or(UploadError::NoFileExtension)?
            .to_lowercase();
            
        let allowed_extensions = ["jpg", "jpeg", "png", "pdf"];
        if !allowed_extensions.contains(&extension.as_str()) {
            return Err(UploadError::InvalidFileExtension);
        }
        
        // Scan for malware
        match self.antivirus.scan_bytes(file_data) {
            Ok(ScanResult::Clean) => {},
            Ok(ScanResult::Virus(virus_name)) => {
                error!("Virus detected: {} in file from user {}", virus_name, user_id);
                return Err(UploadError::VirusDetected);
            },
            Err(e) => {
                error!("Antivirus scan failed: {}", e);
                return Err(UploadError::ScanFailed);
            }
        }
        
        // Generate secure filename
        let secure_filename = format!(
            "{}_{}.{}",
            Uuid::new_v4(),
            chrono::Utc::now().timestamp(),
            extension
        );
        
        // Save to secure location
        let file_path = format!("uploads/{}/{}", user_id, secure_filename);
        
        // Store file metadata in database
        let file_record = FileRecord {
            id: Uuid::new_v4(),
            original_filename: filename.to_string(),
            secure_filename: secure_filename.clone(),
            file_path: file_path.clone(),
            content_type: content_type.to_string(),
            file_size: file_data.len() as u64,
            uploaded_by: user_id.to_string(),
            upload_timestamp: Utc::now(),
            virus_scan_status: "clean".to_string(),
        };
        
        // Save file to storage
        self.save_file_securely(&file_path, file_data).await?;
        
        Ok(UploadResult {
            file_id: file_record.id,
            secure_url: format!("/api/files/{}", file_record.id),
        })
    }
    
    async fn save_file_securely(
        &self,
        file_path: &str,
        file_data: &[u8],
    ) -> Result<(), std::io::Error> {
        // Create directory if it doesn't exist
        if let Some(parent) = std::path::Path::new(file_path).parent() {
            tokio::fs::create_dir_all(parent).await?;
        }
        
        // Write file with secure permissions
        tokio::fs::write(file_path, file_data).await?;
        
        // Set file permissions (read-only for owner only)
        #[cfg(unix)]
        {
            use std::os::unix::fs::PermissionsExt;
            let mut perms = tokio::fs::metadata(file_path).await?.permissions();
            perms.set_mode(0o600); // Owner read/write only
            tokio::fs::set_permissions(file_path, perms).await?;
        }
        
        Ok(())
    }
}
```

---

## Data Protection & Privacy

### ✅ HIPAA/GDPR Compliance

```typescript
// ✅ REQUIRED: Data privacy controls
interface DataPrivacyConfig {
  dataRetentionDays: number;
  anonymizationRules: AnonymizationRule[];
  consentRequirements: ConsentRequirement[];
  accessLogRetentionDays: number;
}

const HEALTHCARE_PRIVACY_CONFIG: DataPrivacyConfig = {
  dataRetentionDays: 2555, // 7 years for medical records
  anonymizationRules: [
    {
      field: 'patientName',
      method: 'hash',
      afterDays: 365,
    },
    {
      field: 'phoneNumber',
      method: 'mask',
      afterDays: 90,
    },
    {
      field: 'email',
      method: 'hash',
      afterDays: 365,
    },
  ],
  consentRequirements: [
    {
      type: 'data_processing',
      required: true,
      renewalDays: 365,
    },
    {
      type: 'marketing',
      required: false,
      renewalDays: 180,
    },
  ],
  accessLogRetentionDays: 2555,
};

// ✅ REQUIRED: Data encryption at rest
const encryptSensitiveData = async (data: any, encryptionKey: string): Promise<string> => {
  const algorithm = 'aes-256-gcm';
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipher(algorithm, encryptionKey);
  
  let encrypted = cipher.update(JSON.stringify(data), 'utf8', 'hex');
  encrypted += cipher.final('hex');
  
  const authTag = cipher.getAuthTag();
  
  return JSON.stringify({
    encrypted,
    iv: iv.toString('hex'),
    authTag: authTag.toString('hex'),
  });
};

// ✅ REQUIRED: GDPR data subject rights
class DataSubjectRightsHandler {
  async handleDataPortabilityRequest(userId: string): Promise<DataExport> {
    // Collect all user data
    const userData = await this.collectUserData(userId);
    
    // Create structured export
    const dataExport: DataExport = {
      personalData: userData.personal,
      medicalRecords: userData.medical,
      appointments: userData.appointments,
      communications: userData.communications,
      exportDate: new Date().toISOString(),
      dataController: 'HealthTech UHI Platform',
    };
    
    // Log the request
    await this.auditService.logDataRequest({
      userId,
      requestType: 'data_portability',
      timestamp: new Date(),
      status: 'completed',
    });
    
    return dataExport;
  }
  
  async handleDataDeletionRequest(userId: string): Promise<DeletionResult> {
    // Check if user can be deleted (no active medical cases)
    const canDelete = await this.checkDeletionEligibility(userId);
    
    if (!canDelete.eligible) {
      return {
        success: false,
        reason: canDelete.reason,
        retryAfter: canDelete.retryAfter,
      };
    }
    
    // Anonymize instead of hard delete for medical data
    await this.anonymizeUserData(userId);
    
    // Delete non-medical data
    await this.deleteNonMedicalData(userId);
    
    return { success: true };
  }
}
```

---

## Monitoring & Incident Response

### ✅ Security Monitoring

```typescript
// ✅ REQUIRED: Security event monitoring
interface SecurityEvent {
  id: string;
  type: SecurityEventType;
  severity: 'low' | 'medium' | 'high' | 'critical';
  userId?: string;
  ipAddress: string;
  userAgent: string;
  timestamp: Date;
  details: Record<string, any>;
}

enum SecurityEventType {
  FAILED_LOGIN = 'failed_login',
  BRUTE_FORCE_ATTEMPT = 'brute_force_attempt',
  SUSPICIOUS_API_ACCESS = 'suspicious_api_access',
  DATA_EXPORT_REQUEST = 'data_export_request',
  ADMIN_ACTION = 'admin_action',
  UNUSUAL_ACCESS_PATTERN = 'unusual_access_pattern',
  MALWARE_DETECTED = 'malware_detected',
  SQL_INJECTION_ATTEMPT = 'sql_injection_attempt',
}

class SecurityMonitor {
  private failedAttempts = new Map<string, number>();
  private suspiciousPatterns = new Map<string, Date[]>();
  
  async logSecurityEvent(event: SecurityEvent): Promise<void> {
    // Store in security log
    await this.securityLogger.log(event);
    
    // Check for patterns
    await this.analyzeSecurityPattern(event);
    
    // Send alerts for critical events
    if (event.severity === 'critical') {
      await this.sendImmediateAlert(event);
    }
    
    // Update monitoring dashboards
    await this.updateSecurityMetrics(event);
  }
  
  async analyzeSecurityPattern(event: SecurityEvent): Promise<void> {
    switch (event.type) {
      case SecurityEventType.FAILED_LOGIN:
        await this.handleFailedLogin(event);
        break;
        
      case SecurityEventType.SUSPICIOUS_API_ACCESS:
        await this.handleSuspiciousApiAccess(event);
        break;
        
      case SecurityEventType.SQL_INJECTION_ATTEMPT:
        await this.handleSqlInjectionAttempt(event);
        break;
    }
  }
  
  private async handleFailedLogin(event: SecurityEvent): Promise<void> {
    const key = `${event.ipAddress}:${event.userId}`;
    const attempts = this.failedAttempts.get(key) || 0;
    
    if (attempts >= 5) {
      // Potential brute force attack
      await this.logSecurityEvent({
        ...event,
        type: SecurityEventType.BRUTE_FORCE_ATTEMPT,
        severity: 'high',
        details: {
          ...event.details,
          attemptCount: attempts + 1,
        },
      });
      
      // Temporarily block IP
      await this.blockIpAddress(event.ipAddress, 3600); // 1 hour
    }
    
    this.failedAttempts.set(key, attempts + 1);
    
    // Clean up old entries
    setTimeout(() => {
      this.failedAttempts.delete(key);
    }, 15 * 60 * 1000); // 15 minutes
  }
}

// ✅ REQUIRED: Automated incident response
class IncidentResponse {
  async handleSecurityIncident(incident: SecurityIncident): Promise<void> {
    // Create incident ticket
    const ticket = await this.createIncidentTicket(incident);
    
    // Notify security team
    await this.notifySecurityTeam(incident);
    
    // Execute automated response
    switch (incident.severity) {
      case 'critical':
        await this.handleCriticalIncident(incident);
        break;
      case 'high':
        await this.handleHighSeverityIncident(incident);
        break;
      default:
        await this.handleStandardIncident(incident);
        break;
    }
    
    // Update incident status
    await this.updateIncidentStatus(ticket.id, 'responding');
  }
  
  private async handleCriticalIncident(incident: SecurityIncident): Promise<void> {
    // Immediate response for critical security incidents
    
    // 1. Block malicious IPs
    if (incident.indicators?.maliciousIps) {
      for (const ip of incident.indicators.maliciousIps) {
        await this.blockIpAddress(ip, 86400); // 24 hours
      }
    }
    
    // 2. Revoke compromised tokens
    if (incident.indicators?.compromisedTokens) {
      for (const token of incident.indicators.compromisedTokens) {
        await this.revokeToken(token);
      }
    }
    
    // 3. Lock affected user accounts
    if (incident.indicators?.affectedUsers) {
      for (const userId of incident.indicators.affectedUsers) {
        await this.lockUserAccount(userId, 'security_incident');
      }
    }
    
    // 4. Page on-call security engineer
    await this.pageSecurityEngineer(incident);
    
    // 5. Initiate incident bridge
    await this.initiateIncidentBridge(incident);
  }
}
```

---

## Security Testing

### ✅ Automated Security Testing

```yaml
# ✅ REQUIRED: Security testing in CI/CD pipeline
name: Security Testing

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    
    # Dependency vulnerability scanning
    - name: Run npm audit
      run: npm audit --audit-level high
      
    - name: Run Snyk test
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        
    # SAST (Static Application Security Testing)
    - name: Run CodeQL Analysis
      uses: github/codeql-action/init@v2
      with:
        languages: typescript, javascript
        
    - name: Run Semgrep
      uses: returntocorp/semgrep-action@v1
      with:
        config: auto
        
    # Container security scanning
    - name: Build Docker image
      run: docker build -t healthtech:test .
      
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'healthtech:test'
        format: 'sarif'
        output: 'trivy-results.sarif'
        
    # Infrastructure security scanning
    - name: Run Checkov
      uses: bridgecrewio/checkov-action@master
      with:
        directory: .
        framework: kubernetes,dockerfile
        
  penetration-testing:
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Run OWASP ZAP
      uses: zaproxy/action-baseline@v0.7.0
      with:
        target: 'https://staging-api.healthtech.com'
        rules_file_name: '.zap/rules.tsv'
        
    - name: Upload ZAP results
      uses: actions/upload-artifact@v3
      with:
        name: zap-report
        path: report_html.html
```

### ✅ Security Testing Checklist

```typescript
// ✅ REQUIRED: Comprehensive security test suite
describe('Security Tests', () => {
  describe('Authentication', () => {
    test('should reject requests without valid JWT', async () => {
      const response = await request(app)
        .get('/api/patients')
        .expect(401);
      
      expect(response.body.error).toContain('Authentication required');
    });
    
    test('should reject expired JWT tokens', async () => {
      const expiredToken = generateExpiredToken();
      
      const response = await request(app)
        .get('/api/patients')
        .set('Authorization', `Bearer ${expiredToken}`)
        .expect(401);
    });
    
    test('should enforce MFA for sensitive operations', async () => {
      const token = generateTokenWithoutMFA();
      
      const response = await request(app)
        .delete('/api/patients/123')
        .set('Authorization', `Bearer ${token}`)
        .expect(403);
      
      expect(response.body.error).toContain('MFA required');
    });
  });
  
  describe('Authorization', () => {
    test('should enforce role-based access control', async () => {
      const patientToken = generateToken({ role: 'patient' });
      
      const response = await request(app)
        .get('/api/admin/users')
        .set('Authorization', `Bearer ${patientToken}`)
        .expect(403);
    });
    
    test('should enforce resource-level permissions', async () => {
      const userToken = generateToken({ userId: 'user1' });
      
      const response = await request(app)
        .get('/api/patients/user2') // Different user's data
        .set('Authorization', `Bearer ${userToken}`)
        .expect(403);
    });
  });
  
  describe('Input Validation', () => {
    test('should reject SQL injection attempts', async () => {
      const maliciousInput = "'; DROP TABLE patients; --";
      
      const response = await request(app)
        .post('/api/patients')
        .send({ name: maliciousInput })
        .expect(400);
      
      expect(response.body.error).toContain('Invalid input');
    });
    
    test('should sanitize XSS attempts', async () => {
      const xssPayload = '<script>alert("xss")</script>';
      
      const response = await request(app)
        .post('/api/patients')
        .send({ name: xssPayload })
        .expect(400);
    });
    
    test('should enforce file upload restrictions', async () => {
      const maliciousFile = Buffer.from('malicious content');
      
      const response = await request(app)
        .post('/api/upload')
        .attach('file', maliciousFile, 'malware.exe')
        .expect(400);
      
      expect(response.body.error).toContain('File type not allowed');
    });
  });
  
  describe('Rate Limiting', () => {
    test('should block excessive requests', async () => {
      const token = generateValidToken();
      
      // Make multiple requests quickly
      const promises = Array(100).fill(null).map(() =>
        request(app)
          .get('/api/patients')
          .set('Authorization', `Bearer ${token}`)
      );
      
      const responses = await Promise.all(promises);
      const rateLimitedResponses = responses.filter(r => r.status === 429);
      
      expect(rateLimitedResponses.length).toBeGreaterThan(0);
    });
  });
  
  describe('Data Protection', () => {
    test('should not leak sensitive data in error messages', async () => {
      // Trigger a database error
      const response = await request(app)
        .get('/api/patients/invalid-uuid')
        .set('Authorization', generateValidToken())
        .expect(400);
      
      // Ensure no internal details are exposed
      expect(response.body.error).not.toContain('database');
      expect(response.body.error).not.toContain('SQL');
      expect(response.body.error).not.toContain('connection');
    });
    
    test('should encrypt sensitive data at rest', async () => {
      const patientData = {
        name: 'John Doe',
        ssn: '123-45-6789',
        medicalHistory: 'Confidential medical information',
      };
      
      // Create patient
      await request(app)
        .post('/api/patients')
        .send(patientData)
        .expect(201);
      
      // Check database directly - sensitive fields should be encrypted
      const dbRecord = await db.query('SELECT * FROM patients WHERE name = $1', ['John Doe']);
      expect(dbRecord.rows[0].ssn).not.toBe(patientData.ssn);
      expect(dbRecord.rows[0].medical_history).not.toBe(patientData.medicalHistory);
    });
  });
});
```

---

## Compliance Checklist

### ✅ HIPAA Compliance Verification

```typescript
// ✅ REQUIRED: HIPAA compliance audit trail
class HIPAAComplianceAuditor {
  async performComplianceAudit(): Promise<ComplianceReport> {
    const checks = [
      this.verifyAccessControls(),
      this.verifyAuditLogs(),
      this.verifyDataEncryption(),
      this.verifyBusinessAssociateAgreements(),
      this.verifyIncidentResponsePlan(),
      this.verifyEmployeeTraining(),
      this.verifyDataBackupAndRecovery(),
    ];
    
    const results = await Promise.all(checks);
    
    return {
      overallCompliance: results.every(r => r.compliant),
      checks: results,
      recommendations: this.generateRecommendations(results),
      auditDate: new Date(),
    };
  }
  
  private async verifyAccessControls(): Promise<ComplianceCheck> {
    // Verify role-based access control implementation
    const rbacImplemented = await this.checkRBACImplementation();
    
    // Verify minimum necessary access principle
    const minimumNecessaryAccess = await this.checkMinimumNecessaryAccess();
    
    // Verify user access reviews
    const accessReviews = await this.checkAccessReviews();
    
    return {
      name: 'Access Controls',
      compliant: rbacImplemented && minimumNecessaryAccess && accessReviews,
      details: {
        rbacImplemented,
        minimumNecessaryAccess,
        accessReviews,
      },
    };
  }
  
  private async verifyAuditLogs(): Promise<ComplianceCheck> {
    // Check if all PHI access is logged
    const phiAccessLogged = await this.checkPHIAccessLogging();
    
    // Check log retention period (6 years minimum)
    const logRetention = await this.checkLogRetention();
    
    // Check log integrity and security
    const logIntegrity = await this.checkLogIntegrity();
    
    return {
      name: 'Audit Logs',
      compliant: phiAccessLogged && logRetention && logIntegrity,
      details: {
        phiAccessLogged,
        logRetention,
        logIntegrity,
      },
    };
  }
}
```

---

## Emergency Response Procedures

### ✅ Security Incident Response Plan

```typescript
// ✅ REQUIRED: Automated incident response
interface SecurityIncidentPlan {
  severity: 'low' | 'medium' | 'high' | 'critical';
  responseTimeMinutes: number;
  escalationPath: string[];
  automatedActions: string[];
  communicationPlan: string[];
}

const INCIDENT_RESPONSE_PLANS: Record<string, SecurityIncidentPlan> = {
  'data_breach': {
    severity: 'critical',
    responseTimeMinutes: 15,
    escalationPath: ['security_team', 'ciso', 'ceo', 'legal_team'],
    automatedActions: [
      'block_affected_accounts',
      'revoke_access_tokens',
      'enable_enhanced_monitoring',
      'preserve_forensic_evidence',
    ],
    communicationPlan: [
      'internal_security_alert',
      'stakeholder_notification',
      'regulatory_reporting',
      'customer_communication',
    ],
  },
  'malware_detected': {
    severity: 'high',
    responseTimeMinutes: 30,
    escalationPath: ['security_team', 'infrastructure_team'],
    automatedActions: [
      'isolate_affected_systems',
      'run_full_antivirus_scan',
      'update_security_signatures',
    ],
    communicationPlan: [
      'internal_security_alert',
      'infrastructure_team_notification',
    ],
  },
  'unauthorized_access': {
    severity: 'high',
    responseTimeMinutes: 20,
    escalationPath: ['security_team', 'user_management_team'],
    automatedActions: [
      'lock_affected_accounts',
      'force_password_reset',
      'enable_enhanced_logging',
    ],
    communicationPlan: [
      'internal_security_alert',
      'affected_user_notification',
    ],
  },
};

class SecurityIncidentResponseSystem {
  async executeIncidentResponse(incidentType: string, details: any): Promise<void> {
    const plan = INCIDENT_RESPONSE_PLANS[incidentType];
    if (!plan) {
      throw new Error(`No response plan found for incident type: ${incidentType}`);
    }
    
    // Start incident timer
    const incidentId = this.createIncidentRecord(incidentType, plan.severity, details);
    
    // Execute automated actions
    await this.executeAutomatedActions(plan.automatedActions, details);
    
    // Start escalation process
    await this.initiateEscalation(plan.escalationPath, incidentId, plan.responseTimeMinutes);
    
    // Execute communication plan
    await this.executeCommunicationPlan(plan.communicationPlan, incidentId, details);
    
    // Monitor resolution
    await this.monitorIncidentResolution(incidentId, plan.responseTimeMinutes);
  }
}
```

---

## Summary: Critical Security Rules (NON-NEGOTIABLE)

### 🔴 MANDATORY REQUIREMENTS

1. **✅ Use Battle-Tested Auth Library** - Never implement custom authentication
2. **✅ Lock Down Protected Endpoints** - Every API request must be authenticated
3. **✅ Never Expose Secrets Frontend** - All secrets server-side only
4. **✅ Git-Ignore Sensitive Files** - Comprehensive .gitignore for secrets
5. **✅ Sanitize Error Messages** - Never reveal internal system details
6. **✅ Middleware Auth Checks** - Gatekeeper pattern for all protected routes
7. **✅ Role-Based Access Control** - Granular permissions system
8. **✅ Secure DB Libraries** - Use ORMs with parameterized queries
9. **✅ Secure Platform Hosting** - Built-in DDoS protection and WAF
10. **✅ HTTPS Everywhere** - Force SSL/TLS in all environments
11. **✅ File Upload Security** - Malware scanning and type validation

### 🟡 IMPLEMENTATION PRIORITIES

1. **Authentication & Authorization** - Foundation of all security
2. **Input Validation & Sanitization** - Prevent injection attacks
3. **Error Handling & Logging** - Secure monitoring without data leaks
4. **Infrastructure Security** - Secure deployment and hosting
5. **Data Protection** - Encryption at rest and in transit
6. **Monitoring & Response** - Automated threat detection
7. **Compliance** - HIPAA/GDPR requirements
8. **Testing** - Continuous security validation

### 🔵 MONITORING REQUIREMENTS

- **Security Event Logging** - All access attempts and failures
- **Automated Threat Detection** - Pattern analysis and blocking
- **Incident Response** - Automated containment and escalation
- **Compliance Auditing** - Regular HIPAA/GDPR compliance checks
- **Penetration Testing** - Quarterly third-party security assessments
- **Employee Training** - Regular security awareness programs

**This security checklist MUST be followed without exception. Any deviation requires explicit approval from the Chief Security Officer and documented risk assessment.**
