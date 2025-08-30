# Section 7: Backend Guidelines

## Table of Contents
1. [Server Architecture](#server-architecture)
2. [API Design Standards](#api-design-standards)
3. [Database Design](#database-design)
4. [Caching Strategies](#caching-strategies)
5. [Security Implementation](#security-implementation)
6. [Performance Optimization](#performance-optimization)
7. [Error Handling](#error-handling)
8. [Logging & Monitoring](#logging--monitoring)
9. [Testing Strategies](#testing-strategies)
10. [Deployment & DevOps](#deployment--devops)

---

## Server Architecture

### 1. Microservices Architecture with Rust
```rust
// Main gRPC server structure using Tonic
use tonic::{transport::Server, Request, Response, Status};
use tonic_reflection::server::Builder as ReflectionBuilder;
use tower::ServiceBuilder;
use tower_http::trace::TraceLayer;

#[derive(Clone)]
pub struct AppState {
    db: Arc<PgPool>,
    redis: Arc<Redis>,
    config: Arc<Config>,
}

pub async fn create_grpc_server(state: AppState) -> Result<(), Box<dyn std::error::Error>> {
    let addr = "0.0.0.0:50051".parse()?;
    
    let patient_service = PatientServiceImpl::new(state.clone());
    let appointment_service = AppointmentServiceImpl::new(state.clone());
    let payment_service = PaymentServiceImpl::new(state);
    
    let reflection_service = ReflectionBuilder::configure()
        .register_encoded_file_descriptor_set(include_bytes!("../proto/descriptor.bin"))
        .build()?;
    
    Server::builder()
        .layer(
            ServiceBuilder::new()
                .layer(TraceLayer::new_for_grpc())
                .layer(tower::middleware::from_fn(auth_interceptor))
        )
        .add_service(reflection_service)
        .add_service(patient_service_server::PatientServiceServer::new(patient_service))
        .add_service(appointment_service_server::AppointmentServiceServer::new(appointment_service))
        .add_service(payment_service_server::PaymentServiceServer::new(payment_service))
        .serve(addr)
        .await?;
    
    Ok(())
}
```

### 2. Service Layer Pattern
```rust
// Domain-driven design with service layers
pub mod domain {
    pub mod entities {
        #[derive(Debug, Clone, Serialize, Deserialize)]
        pub struct Patient {
            pub id: Uuid,
            pub abha_id: String,
            pub name: String,
            pub email: String,
            pub phone: String,
            pub date_of_birth: NaiveDate,
            pub created_at: DateTime<Utc>,
            pub updated_at: DateTime<Utc>,
        }
    }
    
    pub mod repositories {
        #[async_trait]
        pub trait PatientRepository: Send + Sync {
            async fn create(&self, patient: CreatePatientRequest) -> Result<Patient, RepositoryError>;
            async fn find_by_id(&self, id: Uuid) -> Result<Option<Patient>, RepositoryError>;
            async fn find_by_abha_id(&self, abha_id: &str) -> Result<Option<Patient>, RepositoryError>;
            async fn update(&self, id: Uuid, updates: UpdatePatientRequest) -> Result<Patient, RepositoryError>;
        }
    }
    
    pub mod services {
        pub struct PatientService {
            repository: Arc<dyn PatientRepository>,
            event_bus: Arc<EventBus>,
        }
        
        impl PatientService {
            pub async fn create_patient(&self, request: CreatePatientRequest) -> Result<Patient, ServiceError> {
                // Business logic validation
                self.validate_abha_id(&request.abha_id).await?;
                
                // Create patient
                let patient = self.repository.create(request).await?;
                
                // Publish event
                self.event_bus.publish(PatientCreated { patient_id: patient.id }).await?;
                
                Ok(patient)
            }
        }
    }
}
```

### 3. gRPC Service Definition
```rust
// gRPC service for inter-service communication
use tonic::{Request, Response, Status};

#[derive(Debug, Default)]
pub struct AppointmentService {
    repository: Arc<dyn AppointmentRepository>,
}

#[tonic::async_trait]
impl appointment_service_server::AppointmentService for AppointmentService {
    async fn create_appointment(
        &self,
        request: Request<CreateAppointmentRequest>,
    ) -> Result<Response<AppointmentResponse>, Status> {
        let req = request.into_inner();
        
        let appointment = self.repository
            .create(req.into())
            .await
            .map_err(|e| Status::internal(format!("Failed to create appointment: {}", e)))?;
            
        Ok(Response::new(appointment.into()))
    }
    
    async fn get_appointments(
        &self,
        request: Request<GetAppointmentsRequest>,
    ) -> Result<Response<AppointmentsResponse>, Status> {
        let req = request.into_inner();
        
        let appointments = self.repository
            .find_by_patient_id(req.patient_id)
            .await
            .map_err(|e| Status::internal(format!("Failed to fetch appointments: {}", e)))?;
            
        Ok(Response::new(AppointmentsResponse {
            appointments: appointments.into_iter().map(|a| a.into()).collect(),
        }))
    }
}
```

---

## API Design Standards

### 1. RESTful API Conventions
```rust
// REST endpoint handlers
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    response::Json,
};

// GET /api/v1/patients/{id}
pub async fn get_patient(
    State(state): State<AppState>,
    Path(id): Path<Uuid>,
) -> Result<Json<PatientResponse>, ApiError> {
    let patient = state.patient_service
        .get_by_id(id)
        .await?
        .ok_or(ApiError::NotFound("Patient not found".to_string()))?;
        
    Ok(Json(patient.into()))
}

// GET /api/v1/patients?page=1&limit=20&search=john
pub async fn list_patients(
    State(state): State<AppState>,
    Query(params): Query<ListPatientsQuery>,
) -> Result<Json<PaginatedResponse<PatientResponse>>, ApiError> {
    let patients = state.patient_service
        .list(params.into())
        .await?;
        
    Ok(Json(patients.into()))
}

// POST /api/v1/patients
pub async fn create_patient(
    State(state): State<AppState>,
    Json(request): Json<CreatePatientRequest>,
) -> Result<(StatusCode, Json<PatientResponse>), ApiError> {
    let patient = state.patient_service
        .create(request)
        .await?;
        
    Ok((StatusCode::CREATED, Json(patient.into())))
}
```

### 2. API Versioning Strategy
```rust
// Version-aware routing
pub fn create_versioned_routes() -> Router<AppState> {
    Router::new()
        .nest("/api/v1", v1_routes())
        .nest("/api/v2", v2_routes())
        .route("/api/version", get(get_api_version))
}

// API version response
#[derive(Serialize)]
pub struct ApiVersionResponse {
    current_version: String,
    supported_versions: Vec<String>,
    deprecated_versions: Vec<String>,
}

pub async fn get_api_version() -> Json<ApiVersionResponse> {
    Json(ApiVersionResponse {
        current_version: "v2".to_string(),
        supported_versions: vec!["v1".to_string(), "v2".to_string()],
        deprecated_versions: vec!["v0".to_string()],
    })
}
```

### 3. OpenAPI Documentation
```rust
// Automated API documentation with utoipa
use utoipa::{OpenApi, ToSchema};

#[derive(OpenApi)]
#[openapi(
    paths(
        get_patient,
        list_patients,
        create_patient,
        update_patient,
        delete_patient
    ),
    components(
        schemas(PatientResponse, CreatePatientRequest, UpdatePatientRequest)
    ),
    tags(
        (name = "patients", description = "Patient management endpoints")
    ),
    info(
        title = "HealthTech UHI API",
        version = "1.0.0",
        description = "Universal Health Interface Platform API"
    ),
    servers(
        (url = "https://api.healthtech-uhi.com", description = "Production server"),
        (url = "https://staging-api.healthtech-uhi.com", description = "Staging server")
    )
)]
pub struct ApiDoc;

// Schema definitions
#[derive(Serialize, Deserialize, ToSchema)]
pub struct PatientResponse {
    #[schema(example = "550e8400-e29b-41d4-a716-446655440000")]
    pub id: Uuid,
    #[schema(example = "91-1234-5678-9012")]
    pub abha_id: String,
    #[schema(example = "John Doe")]
    pub name: String,
    #[schema(example = "john.doe@example.com")]
    pub email: String,
    #[schema(example = "+91-9876543210")]
    pub phone: String,
}
```

### 4. UHI Protocol Integration
```rust
// UHI-compliant message handling
use serde_json::Value;

#[derive(Serialize, Deserialize)]
pub struct UhiMessage {
    pub context: UhiContext,
    pub message: Value,
}

#[derive(Serialize, Deserialize)]
pub struct UhiContext {
    pub domain: String,
    pub country: String,
    pub city: String,
    pub action: String,
    pub core_version: String,
    pub bap_id: String,
    pub bap_uri: String,
    pub bpp_id: Option<String>,
    pub bpp_uri: Option<String>,
    pub transaction_id: String,
    pub message_id: String,
    pub timestamp: DateTime<Utc>,
}

// UHI endpoint handlers
pub async fn uhi_search(
    State(state): State<AppState>,
    Json(message): Json<UhiMessage>,
) -> Result<Json<UhiMessage>, ApiError> {
    // Validate UHI message structure
    validate_uhi_context(&message.context)?;
    
    // Process search request
    let search_results = state.search_service
        .search_providers(message.message)
        .await?;
    
    // Return UHI-compliant response
    Ok(Json(UhiMessage {
        context: create_response_context(&message.context),
        message: search_results,
    }))
}
```

---

## Database Design

### 1. PostgreSQL Schema Design
```sql
-- Core tables with proper relationships and constraints
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Patients table
CREATE TABLE patients (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    abha_id VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20) NOT NULL,
    date_of_birth DATE NOT NULL,
    gender VARCHAR(10) CHECK (gender IN ('male', 'female', 'other')),
    address JSONB,
    emergency_contact JSONB,
    medical_history JSONB,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Healthcare providers table
CREATE TABLE healthcare_providers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    uhi_id VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    license_number VARCHAR(100) UNIQUE NOT NULL,
    specialization VARCHAR(100) NOT NULL,
    qualification JSONB NOT NULL,
    experience_years INTEGER,
    consultation_fee DECIMAL(10,2),
    languages TEXT[],
    availability JSONB,
    rating DECIMAL(3,2) DEFAULT 0.00,
    total_reviews INTEGER DEFAULT 0,
    verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Appointments table
CREATE TABLE appointments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    patient_id UUID NOT NULL REFERENCES patients(id) ON DELETE CASCADE,
    provider_id UUID NOT NULL REFERENCES healthcare_providers(id) ON DELETE CASCADE,
    appointment_date TIMESTAMPTZ NOT NULL,
    duration_minutes INTEGER DEFAULT 30,
    type VARCHAR(20) CHECK (type IN ('video', 'audio', 'in_person', 'chat')),
    status VARCHAR(20) DEFAULT 'scheduled' CHECK (status IN ('scheduled', 'confirmed', 'in_progress', 'completed', 'cancelled', 'no_show')),
    symptoms TEXT,
    diagnosis TEXT,
    prescription JSONB,
    notes TEXT,
    payment_status VARCHAR(20) DEFAULT 'pending' CHECK (payment_status IN ('pending', 'paid', 'failed', 'refunded')),
    payment_amount DECIMAL(10,2),
    meeting_link VARCHAR(500),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Indexes for performance
CREATE INDEX idx_patients_abha_id ON patients(abha_id);
CREATE INDEX idx_patients_email ON patients(email);
CREATE INDEX idx_patients_phone ON patients(phone);
CREATE INDEX idx_providers_uhi_id ON healthcare_providers(uhi_id);
CREATE INDEX idx_providers_specialization ON healthcare_providers(specialization);
CREATE INDEX idx_providers_verified ON healthcare_providers(verified) WHERE verified = TRUE;
CREATE INDEX idx_appointments_patient_id ON appointments(patient_id);
CREATE INDEX idx_appointments_provider_id ON appointments(provider_id);
CREATE INDEX idx_appointments_date ON appointments(appointment_date);
CREATE INDEX idx_appointments_status ON appointments(status);

-- Full-text search indexes
CREATE INDEX idx_patients_name_search ON patients USING GIN (to_tsvector('english', name));
CREATE INDEX idx_providers_name_search ON healthcare_providers USING GIN (to_tsvector('english', name));
```

### 2. Database Repository Implementation
```rust
use sqlx::{PgPool, Row};
use uuid::Uuid;

pub struct PostgresPatientRepository {
    pool: Arc<PgPool>,
}

impl PostgresPatientRepository {
    pub fn new(pool: Arc<PgPool>) -> Self {
        Self { pool }
    }
}

#[async_trait]
impl PatientRepository for PostgresPatientRepository {
    async fn create(&self, request: CreatePatientRequest) -> Result<Patient, RepositoryError> {
        let patient = sqlx::query_as!(
            Patient,
            r#"
            INSERT INTO patients (abha_id, name, email, phone, date_of_birth, gender, address)
            VALUES ($1, $2, $3, $4, $5, $6, $7)
            RETURNING id, abha_id, name, email, phone, date_of_birth, gender, 
                      address, emergency_contact, medical_history, created_at, updated_at
            "#,
            request.abha_id,
            request.name,
            request.email,
            request.phone,
            request.date_of_birth,
            request.gender,
            request.address as Option<serde_json::Value>
        )
        .fetch_one(&*self.pool)
        .await
        .map_err(|e| RepositoryError::Database(e.to_string()))?;
        
        Ok(patient)
    }
    
    async fn find_by_id(&self, id: Uuid) -> Result<Option<Patient>, RepositoryError> {
        let patient = sqlx::query_as!(
            Patient,
            "SELECT * FROM patients WHERE id = $1",
            id
        )
        .fetch_optional(&*self.pool)
        .await
        .map_err(|e| RepositoryError::Database(e.to_string()))?;
        
        Ok(patient)
    }
    
    async fn search(&self, params: SearchPatientsParams) -> Result<PaginatedResult<Patient>, RepositoryError> {
        let offset = (params.page - 1) * params.limit;
        
        let patients = sqlx::query_as!(
            Patient,
            r#"
            SELECT * FROM patients
            WHERE ($1::text IS NULL OR to_tsvector('english', name) @@ plainto_tsquery('english', $1))
            AND ($2::text IS NULL OR phone ILIKE $2)
            ORDER BY created_at DESC
            LIMIT $3 OFFSET $4
            "#,
            params.search,
            params.phone.map(|p| format!("%{}%", p)),
            params.limit as i64,
            offset as i64
        )
        .fetch_all(&*self.pool)
        .await
        .map_err(|e| RepositoryError::Database(e.to_string()))?;
        
        let total = sqlx::query_scalar!(
            r#"
            SELECT COUNT(*) FROM patients
            WHERE ($1::text IS NULL OR to_tsvector('english', name) @@ plainto_tsquery('english', $1))
            AND ($2::text IS NULL OR phone ILIKE $2)
            "#,
            params.search,
            params.phone.map(|p| format!("%{}%", p))
        )
        .fetch_one(&*self.pool)
        .await
        .map_err(|e| RepositoryError::Database(e.to_string()))?
        .unwrap_or(0) as u64;
        
        Ok(PaginatedResult {
            data: patients,
            total,
            page: params.page,
            limit: params.limit,
            total_pages: (total + params.limit as u64 - 1) / params.limit as u64,
        })
    }
}
```

### 3. Database Migrations
```rust
// Migration management with sqlx-migrate
use sqlx::migrate::Migrator;

pub static MIGRATOR: Migrator = sqlx::migrate!("./migrations");

pub async fn run_migrations(pool: &PgPool) -> Result<(), sqlx::Error> {
    MIGRATOR.run(pool).await
}

// Migration file: 001_initial_schema.sql
/*
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE patients (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    abha_id VARCHAR(20) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    -- ... rest of schema
);
*/
```

---

## Caching Strategies

### 1. Redis Implementation
```rust
use redis::{Client, AsyncCommands, RedisResult};
use serde::{Serialize, Deserialize};

pub struct RedisCache {
    client: Client,
}

impl RedisCache {
    pub fn new(redis_url: &str) -> Result<Self, redis::RedisError> {
        let client = Client::open(redis_url)?;
        Ok(Self { client })
    }
    
    pub async fn get<T>(&self, key: &str) -> Result<Option<T>, CacheError>
    where
        T: for<'de> Deserialize<'de>,
    {
        let mut conn = self.client.get_async_connection().await?;
        let value: Option<String> = conn.get(key).await?;
        
        match value {
            Some(json) => {
                let data = serde_json::from_str(&json)?;
                Ok(Some(data))
            }
            None => Ok(None),
        }
    }
    
    pub async fn set<T>(&self, key: &str, value: &T, ttl: u64) -> Result<(), CacheError>
    where
        T: Serialize,
    {
        let mut conn = self.client.get_async_connection().await?;
        let json = serde_json::to_string(value)?;
        conn.setex(key, ttl, json).await?;
        Ok(())
    }
    
    pub async fn invalidate(&self, pattern: &str) -> Result<(), CacheError> {
        let mut conn = self.client.get_async_connection().await?;
        let keys: Vec<String> = conn.keys(pattern).await?;
        
        if !keys.is_empty() {
            conn.del(keys).await?;
        }
        
        Ok(())
    }
}

// Cache middleware
pub async fn cache_middleware<B>(
    request: Request<B>,
    next: Next<B>,
) -> Result<Response, StatusCode> {
    let cache_key = format!("api:{}:{}", request.method(), request.uri().path());
    
    // Try to get from cache first
    if let Ok(Some(cached_response)) = CACHE.get::<CachedResponse>(&cache_key).await {
        return Ok(cached_response.into_response());
    }
    
    // Execute the request
    let response = next.run(request).await?;
    
    // Cache successful responses
    if response.status().is_success() {
        let cached = CachedResponse::from_response(&response);
        let _ = CACHE.set(&cache_key, &cached, 300).await; // 5 minutes TTL
    }
    
    Ok(response)
}
```

### 2. Multi-Level Caching Strategy
```rust
// Application-level caching with multiple layers
pub struct CacheManager {
    memory_cache: Arc<MemoryCache>,
    redis_cache: Arc<RedisCache>,
    database: Arc<PgPool>,
}

impl CacheManager {
    pub async fn get_patient(&self, id: Uuid) -> Result<Patient, ServiceError> {
        let cache_key = format!("patient:{}", id);
        
        // L1: Memory cache (fastest)
        if let Some(patient) = self.memory_cache.get(&cache_key) {
            return Ok(patient);
        }
        
        // L2: Redis cache (fast)
        if let Some(patient) = self.redis_cache.get::<Patient>(&cache_key).await? {
            // Warm L1 cache
            self.memory_cache.set(cache_key.clone(), patient.clone(), Duration::from_secs(60));
            return Ok(patient);
        }
        
        // L3: Database (slowest)
        let patient = self.database
            .get_patient(id)
            .await?
            .ok_or(ServiceError::NotFound)?;
        
        // Warm all cache layers
        self.memory_cache.set(cache_key.clone(), patient.clone(), Duration::from_secs(60));
        let _ = self.redis_cache.set(&cache_key, &patient, 300).await;
        
        Ok(patient)
    }
}
```

### 3. Cache Invalidation Strategies
```rust
// Event-driven cache invalidation
pub struct CacheInvalidator {
    cache: Arc<RedisCache>,
    event_bus: Arc<EventBus>,
}

impl CacheInvalidator {
    pub async fn start(&self) -> Result<(), EventError> {
        let mut subscriber = self.event_bus.subscribe().await?;
        
        while let Some(event) = subscriber.next().await {
            match event {
                Event::PatientUpdated { patient_id } => {
                    self.invalidate_patient_cache(patient_id).await?;
                }
                Event::AppointmentCreated { patient_id, provider_id } => {
                    self.invalidate_appointment_cache(patient_id, provider_id).await?;
                }
                _ => {}
            }
        }
        
        Ok(())
    }
    
    async fn invalidate_patient_cache(&self, patient_id: Uuid) -> Result<(), CacheError> {
        let patterns = vec![
            format!("patient:{}", patient_id),
            format!("patient:{}:*", patient_id),
            "patients:list:*",
        ];
        
        for pattern in patterns {
            self.cache.invalidate(&pattern).await?;
        }
        
        Ok(())
    }
}
```

---

## Security Implementation

### 1. Authentication & Authorization
```rust
use jsonwebtoken::{decode, encode, Header, Validation, DecodingKey, EncodingKey};
use axum::middleware::from_fn_with_state;

#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub sub: String,  // Subject (user ID)
    pub role: UserRole,
    pub permissions: Vec<Permission>,
    pub exp: i64,     // Expiration time
    pub iat: i64,     // Issued at
    pub iss: String,  // Issuer
}

#[derive(Debug, Serialize, Deserialize, PartialEq)]
pub enum UserRole {
    Patient,
    Provider,
    Admin,
    Support,
}

#[derive(Debug, Serialize, Deserialize, PartialEq)]
pub enum Permission {
    ReadPatients,
    WritePatients,
    ReadAppointments,
    WriteAppointments,
    ManageProviders,
    AdminAccess,
}

// gRPC interceptor for authentication
pub fn auth_interceptor(req: Request<()>) -> Result<Request<()>, Status> {
    let token = req
        .metadata()
        .get("authorization")
        .and_then(|value| value.to_str().ok())
        .and_then(|auth| auth.strip_prefix("Bearer "));
    
    let token = token.ok_or_else(|| Status::unauthenticated("Missing authorization token"))?;
    
    let claims = validate_jwt(token, &JWT_SECRET)
        .map_err(|_| Status::unauthenticated("Invalid token"))?;
    
    // Add claims to request extensions
    let mut req = req;
    req.extensions_mut().insert(claims);
    
    Ok(req)
}

// Role-based access control
pub fn require_role(required_role: UserRole) -> impl Fn(Request<Body>, Next<Body>) -> Pin<Box<dyn Future<Output = Result<Response, StatusCode>> + Send>> {
    move |request: Request<Body>, next: Next<Body>| {
        let required_role = required_role.clone();
        Box::pin(async move {
            let claims = request
                .extensions()
                .get::<Claims>()
                .ok_or(StatusCode::UNAUTHORIZED)?;
            
            if claims.role != required_role && claims.role != UserRole::Admin {
                return Err(StatusCode::FORBIDDEN);
            }
            
            Ok(next.run(request).await?)
        })
    }
}

// Permission-based access control
pub fn require_permission(required_permission: Permission) -> impl Fn(Request<Body>, Next<Body>) -> Pin<Box<dyn Future<Output = Result<Response, StatusCode>> + Send>> {
    move |request: Request<Body>, next: Next<Body>| {
        let required_permission = required_permission.clone();
        Box::pin(async move {
            let claims = request
                .extensions()
                .get::<Claims>()
                .ok_or(StatusCode::UNAUTHORIZED)?;
            
            if !claims.permissions.contains(&required_permission) && claims.role != UserRole::Admin {
                return Err(StatusCode::FORBIDDEN);
            }
            
            Ok(next.run(request).await?)
        })
    }
}
```

### 2. Input Validation & Sanitization
```rust
use validator::{Validate, ValidationError};
use axum::extract::rejection::JsonRejection;

#[derive(Debug, Deserialize, Validate)]
pub struct CreatePatientRequest {
    #[validate(regex = "ABHA_ID_REGEX")]
    pub abha_id: String,
    
    #[validate(length(min = 2, max = 255))]
    pub name: String,
    
    #[validate(email)]
    pub email: String,
    
    #[validate(regex = "PHONE_REGEX")]
    pub phone: String,
    
    #[validate(custom = "validate_date_of_birth")]
    pub date_of_birth: NaiveDate,
}

// Custom validator functions
fn validate_date_of_birth(date: &NaiveDate) -> Result<(), ValidationError> {
    let today = Utc::now().date_naive();
    let max_age = today.years_since(*date).unwrap_or(0);
    
    if max_age > 150 {
        return Err(ValidationError::new("date_too_old"));
    }
    
    if *date > today {
        return Err(ValidationError::new("future_date"));
    }
    
    Ok(())
}

// Input sanitization
pub fn sanitize_html(input: &str) -> String {
    ammonia::clean(input)
}

pub fn sanitize_sql_like(input: &str) -> String {
    input.replace('%', "\\%").replace('_', "\\_")
}

// Request validation middleware
pub async fn validate_json<T>(
    payload: Result<Json<T>, JsonRejection>,
) -> Result<Json<T>, ApiError>
where
    T: Validate,
{
    match payload {
        Ok(Json(data)) => {
            data.validate()
                .map_err(|e| ApiError::ValidationError(e.to_string()))?;
            Ok(Json(data))
        }
        Err(rejection) => Err(ApiError::BadRequest(format!("Invalid JSON: {}", rejection))),
    }
}
```

### 3. Rate Limiting
```rust
use governor::{Quota, RateLimiter, DefaultKeyedRateLimiter};
use std::collections::HashMap;

pub struct RateLimitMiddleware {
    limiters: HashMap<String, DefaultKeyedRateLimiter<String>>,
}

impl RateLimitMiddleware {
    pub fn new() -> Self {
        let mut limiters = HashMap::new();
        
        // Different rate limits for different endpoints
        limiters.insert(
            "auth".to_string(),
            RateLimiter::keyed(Quota::per_minute(5)), // 5 auth attempts per minute
        );
        limiters.insert(
            "api".to_string(),
            RateLimiter::keyed(Quota::per_minute(100)), // 100 API calls per minute
        );
        limiters.insert(
            "search".to_string(),
            RateLimiter::keyed(Quota::per_minute(20)), // 20 searches per minute
        );
        
        Self { limiters }
    }
}

pub async fn rate_limit_middleware(
    request: Request<Body>,
    next: Next<Body>,
) -> Result<Response, StatusCode> {
    let client_ip = get_client_ip(&request).unwrap_or_else(|| "unknown".to_string());
    let endpoint_type = get_endpoint_type(&request);
    
    if let Some(limiter) = RATE_LIMITERS.get(&endpoint_type) {
        match limiter.check_key(&client_ip) {
            Ok(_) => Ok(next.run(request).await?),
            Err(_) => Err(StatusCode::TOO_MANY_REQUESTS),
        }
    } else {
        Ok(next.run(request).await?)
    }
}
```

---

## Performance Optimization

### 1. Connection Pooling
```rust
use sqlx::{PgPool, postgres::PgPoolOptions};
use std::time::Duration;

pub async fn create_database_pool(database_url: &str) -> Result<PgPool, sqlx::Error> {
    PgPoolOptions::new()
        .max_connections(50)
        .min_connections(5)
        .max_lifetime(Duration::from_secs(1800)) // 30 minutes
        .idle_timeout(Duration::from_secs(600))  // 10 minutes
        .acquire_timeout(Duration::from_secs(30))
        .test_before_acquire(true)
        .connect(database_url)
        .await
}

// Connection health monitoring
pub async fn monitor_database_health(pool: &PgPool) -> Result<DatabaseHealth, sqlx::Error> {
    let start = Instant::now();
    let _: (i32,) = sqlx::query_as("SELECT 1").fetch_one(pool).await?;
    let latency = start.elapsed();
    
    Ok(DatabaseHealth {
        active_connections: pool.size(),
        idle_connections: pool.num_idle(),
        latency_ms: latency.as_millis() as u32,
        status: if latency.as_millis() < 100 { "healthy" } else { "slow" }.to_string(),
    })
}
```

### 2. Query Optimization
```rust
// Optimized queries with proper indexing
impl AppointmentRepository for PostgresAppointmentRepository {
    async fn find_upcoming_by_provider(
        &self,
        provider_id: Uuid,
        limit: u32,
    ) -> Result<Vec<Appointment>, RepositoryError> {
        // Use index on (provider_id, appointment_date, status)
        let appointments = sqlx::query_as!(
            Appointment,
            r#"
            SELECT a.*, p.name as patient_name, p.phone as patient_phone
            FROM appointments a
            JOIN patients p ON a.patient_id = p.id
            WHERE a.provider_id = $1 
            AND a.appointment_date > NOW()
            AND a.status IN ('scheduled', 'confirmed')
            ORDER BY a.appointment_date ASC
            LIMIT $2
            "#,
            provider_id,
            limit as i64
        )
        .fetch_all(&*self.pool)
        .await
        .map_err(|e| RepositoryError::Database(e.to_string()))?;
        
        Ok(appointments)
    }
    
    // Batch operations for better performance
    async fn create_bulk_appointments(
        &self,
        appointments: Vec<CreateAppointmentRequest>,
    ) -> Result<Vec<Appointment>, RepositoryError> {
        let mut tx = self.pool.begin().await
            .map_err(|e| RepositoryError::Database(e.to_string()))?;
        
        let mut created_appointments = Vec::new();
        
        for request in appointments {
            let appointment = sqlx::query_as!(
                Appointment,
                r#"
                INSERT INTO appointments (patient_id, provider_id, appointment_date, type, duration_minutes)
                VALUES ($1, $2, $3, $4, $5)
                RETURNING *
                "#,
                request.patient_id,
                request.provider_id,
                request.appointment_date,
                request.appointment_type,
                request.duration_minutes
            )
            .fetch_one(&mut *tx)
            .await
            .map_err(|e| RepositoryError::Database(e.to_string()))?;
            
            created_appointments.push(appointment);
        }
        
        tx.commit().await
            .map_err(|e| RepositoryError::Database(e.to_string()))?;
        
        Ok(created_appointments)
    }
}
```

### 3. Background Job Processing
```rust
use tokio_cron_scheduler::{JobScheduler, Job};

pub struct BackgroundJobService {
    scheduler: JobScheduler,
    database: Arc<PgPool>,
    notification_service: Arc<NotificationService>,
}

impl BackgroundJobService {
    pub async fn new(
        database: Arc<PgPool>,
        notification_service: Arc<NotificationService>,
    ) -> Result<Self, Box<dyn std::error::Error>> {
        let scheduler = JobScheduler::new().await?;
        
        Ok(Self {
            scheduler,
            database,
            notification_service,
        })
    }
    
    pub async fn start(&mut self) -> Result<(), Box<dyn std::error::Error>> {
        // Appointment reminder job (every hour)
        let db = self.database.clone();
        let notif = self.notification_service.clone();
        self.scheduler.add(
            Job::new_async("0 0 * * * *", move |_uuid, _l| {
                let db = db.clone();
                let notif = notif.clone();
                Box::pin(async move {
                    if let Err(e) = send_appointment_reminders(db, notif).await {
                        error!("Failed to send appointment reminders: {}", e);
                    }
                })
            })?
        ).await?;
        
        // Cleanup expired sessions (daily at 2 AM)
        let db = self.database.clone();
        self.scheduler.add(
            Job::new_async("0 0 2 * * *", move |_uuid, _l| {
                let db = db.clone();
                Box::pin(async move {
                    if let Err(e) = cleanup_expired_sessions(db).await {
                        error!("Failed to cleanup expired sessions: {}", e);
                    }
                })
            })?
        ).await?;
        
        self.scheduler.start().await?;
        Ok(())
    }
}

async fn send_appointment_reminders(
    db: Arc<PgPool>,
    notification_service: Arc<NotificationService>,
) -> Result<(), Box<dyn std::error::Error>> {
    // Find appointments in next 24 hours without reminders sent
    let appointments = sqlx::query!(
        r#"
        SELECT a.id, a.appointment_date, p.name as patient_name, p.email as patient_email,
               pr.name as provider_name
        FROM appointments a
        JOIN patients p ON a.patient_id = p.id
        JOIN healthcare_providers pr ON a.provider_id = pr.id
        WHERE a.appointment_date BETWEEN NOW() AND NOW() + INTERVAL '24 hours'
        AND a.status IN ('scheduled', 'confirmed')
        AND a.reminder_sent = FALSE
        "#
    )
    .fetch_all(&*db)
    .await?;
    
    for appointment in appointments {
        // Send reminder notification
        notification_service.send_appointment_reminder(
            &appointment.patient_email,
            &appointment.patient_name,
            &appointment.provider_name,
            appointment.appointment_date,
        ).await?;
        
        // Mark reminder as sent
        sqlx::query!(
            "UPDATE appointments SET reminder_sent = TRUE WHERE id = $1",
            appointment.id
        )
        .execute(&*db)
        .await?;
    }
    
    Ok(())
}
```

---

## Error Handling

### 1. Structured Error Types
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ApiError {
    #[error("Resource not found: {0}")]
    NotFound(String),
    
    #[error("Bad request: {0}")]
    BadRequest(String),
    
    #[error("Validation error: {0}")]
    ValidationError(String),
    
    #[error("Authentication required")]
    Unauthorized,
    
    #[error("Insufficient permissions")]
    Forbidden,
    
    #[error("Rate limit exceeded")]
    RateLimited,
    
    #[error("Internal server error")]
    InternalServerError,
    
    #[error("Service unavailable: {0}")]
    ServiceUnavailable(String),
    
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    
    #[error("Cache error: {0}")]
    Cache(#[from] redis::RedisError),
    
    #[error("Serialization error: {0}")]
    Serialization(#[from] serde_json::Error),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, error_message) = match self {
            ApiError::NotFound(msg) => (StatusCode::NOT_FOUND, msg),
            ApiError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg),
            ApiError::ValidationError(msg) => (StatusCode::UNPROCESSABLE_ENTITY, msg),
            ApiError::Unauthorized => (StatusCode::UNAUTHORIZED, "Authentication required".to_string()),
            ApiError::Forbidden => (StatusCode::FORBIDDEN, "Insufficient permissions".to_string()),
            ApiError::RateLimited => (StatusCode::TOO_MANY_REQUESTS, "Rate limit exceeded".to_string()),
            ApiError::ServiceUnavailable(msg) => (StatusCode::SERVICE_UNAVAILABLE, msg),
            _ => {
                error!("Internal server error: {:?}", self);
                (StatusCode::INTERNAL_SERVER_ERROR, "Internal server error".to_string())
            }
        };
        
        let body = Json(ErrorResponse {
            error: error_message,
            timestamp: Utc::now(),
            request_id: get_request_id().unwrap_or_default(),
        });
        
        (status, body).into_response()
    }
}

#[derive(Serialize)]
struct ErrorResponse {
    error: String,
    timestamp: DateTime<Utc>,
    request_id: String,
}
```

### 2. Circuit Breaker Pattern
```rust
use circuit_breaker::{CircuitBreaker, CircuitBreakerConfig};

pub struct ResilientService {
    circuit_breaker: CircuitBreaker,
    fallback_cache: Arc<RedisCache>,
}

impl ResilientService {
    pub fn new() -> Self {
        let config = CircuitBreakerConfig::new()
            .failure_threshold(5)
            .recovery_timeout(Duration::from_secs(30))
            .expected_response_time(Duration::from_millis(1000));
            
        Self {
            circuit_breaker: CircuitBreaker::new(config),
            fallback_cache: Arc::new(RedisCache::new("redis://localhost:6379").unwrap()),
        }
    }
    
    pub async fn call_external_service<T, F, Fut>(&self, operation: F) -> Result<T, ServiceError>
    where
        F: FnOnce() -> Fut,
        Fut: Future<Output = Result<T, ServiceError>>,
        T: Serialize + for<'de> Deserialize<'de> + Clone,
    {
        match self.circuit_breaker.call(operation).await {
            Ok(result) => Ok(result),
            Err(_) => {
                // Fallback to cached data
                warn!("Circuit breaker open, falling back to cache");
                self.get_cached_fallback().await
            }
        }
    }
    
    async fn get_cached_fallback<T>(&self) -> Result<T, ServiceError>
    where
        T: for<'de> Deserialize<'de>,
    {
        self.fallback_cache
            .get("fallback_data")
            .await
            .map_err(|_| ServiceError::ServiceUnavailable("Service temporarily unavailable".to_string()))?
            .ok_or_else(|| ServiceError::ServiceUnavailable("No fallback data available".to_string()))
    }
}
```

---

## Logging & Monitoring

### 1. Structured Logging
```rust
use tracing::{info, warn, error, debug, instrument};
use tracing_subscriber::{layer::SubscriberExt, util::SubscriberInitExt};

pub fn init_logging() {
    tracing_subscriber::registry()
        .with(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "info".into()),
        )
        .with(tracing_subscriber::fmt::layer().json())
        .init();
}

// Instrumented service methods
impl PatientService {
    #[instrument(
        name = "create_patient",
        skip(self, request),
        fields(
            abha_id = %request.abha_id,
            patient_name = %request.name
        )
    )]
    pub async fn create_patient(&self, request: CreatePatientRequest) -> Result<Patient, ServiceError> {
        info!("Creating new patient");
        
        // Validate ABHA ID
        debug!("Validating ABHA ID");
        self.validate_abha_id(&request.abha_id).await?;
        
        // Create patient
        let patient = self.repository.create(request).await
            .map_err(|e| {
                error!("Failed to create patient: {}", e);
                ServiceError::Database(e.to_string())
            })?;
        
        info!("Patient created successfully", patient_id = %patient.id);
        
        // Publish event
        self.event_bus.publish(PatientCreated { 
            patient_id: patient.id,
            abha_id: patient.abha_id.clone(),
        }).await?;
        
        Ok(patient)
    }
}
```

### 2. Metrics Collection
```rust
use prometheus::{Counter, Histogram, Gauge, Registry, Encoder, TextEncoder};

lazy_static! {
    static ref REGISTRY: Registry = Registry::new();
    
    static ref HTTP_REQUESTS_TOTAL: Counter = Counter::new(
        "http_requests_total", "Total number of HTTP requests"
    ).expect("metric can be created");
    
    static ref HTTP_REQUEST_DURATION: Histogram = Histogram::with_opts(
        prometheus::HistogramOpts::new(
            "http_request_duration_seconds",
            "HTTP request duration in seconds"
        ).buckets(vec![0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0])
    ).expect("metric can be created");
    
    static ref ACTIVE_CONNECTIONS: Gauge = Gauge::new(
        "active_database_connections", "Number of active database connections"
    ).expect("metric can be created");
}

pub fn init_metrics() {
    REGISTRY.register(Box::new(HTTP_REQUESTS_TOTAL.clone())).unwrap();
    REGISTRY.register(Box::new(HTTP_REQUEST_DURATION.clone())).unwrap();
    REGISTRY.register(Box::new(ACTIVE_CONNECTIONS.clone())).unwrap();
}

// Metrics middleware
pub async fn metrics_middleware(
    request: Request<Body>,
    next: Next<Body>,
) -> Result<Response, StatusCode> {
    let start = Instant::now();
    let path = request.uri().path().to_string();
    let method = request.method().to_string();
    
    HTTP_REQUESTS_TOTAL.inc();
    
    let response = next.run(request).await?;
    
    let duration = start.elapsed().as_secs_f64();
    HTTP_REQUEST_DURATION.observe(duration);
    
    // Log slow requests
    if duration > 1.0 {
        warn!("Slow request: {} {} took {:.3}s", method, path, duration);
    }
    
    Ok(response)
}

// Health check endpoint with metrics
pub async fn health_check(State(state): State<AppState>) -> Json<HealthCheckResponse> {
    let db_health = check_database_health(&state.db).await;
    let redis_health = check_redis_health(&state.redis).await;
    
    ACTIVE_CONNECTIONS.set(state.db.size() as f64);
    
    Json(HealthCheckResponse {
        status: if db_health.is_ok() && redis_health.is_ok() { "healthy" } else { "unhealthy" },
        timestamp: Utc::now(),
        version: env!("CARGO_PKG_VERSION").to_string(),
        database: db_health.unwrap_or_else(|e| format!("unhealthy: {}", e)),
        cache: redis_health.unwrap_or_else(|e| format!("unhealthy: {}", e)),
    })
}
```

### 3. Distributed Tracing
```rust
use opentelemetry::{trace::TraceError, KeyValue};
use opentelemetry_jaeger::new_agent_pipeline;
use tracing_opentelemetry::OpenTelemetryLayer;

pub fn init_tracing() -> Result<(), TraceError> {
    let tracer = new_agent_pipeline()
        .with_service_name("healthtech-backend")
        .with_tags(vec![
            KeyValue::new("service.version", env!("CARGO_PKG_VERSION")),
            KeyValue::new("service.environment", std::env::var("ENVIRONMENT").unwrap_or_else(|_| "development".to_string())),
        ])
        .install_simple()?;
    
    let telemetry = OpenTelemetryLayer::new(tracer);
    
    tracing_subscriber::registry()
        .with(telemetry)
        .with(tracing_subscriber::EnvFilter::from_default_env())
        .with(tracing_subscriber::fmt::layer())
        .init();
    
    Ok(())
}

// Custom span creation for database operations
#[instrument(
    name = "db.query",
    skip(pool, query),
    fields(
        db.statement = %query,
        db.operation = "SELECT"
    )
)]
async fn execute_query<T>(pool: &PgPool, query: &str) -> Result<T, sqlx::Error>
where
    T: for<'r> FromRow<'r, PgRow> + Send + Unpin,
{
    sqlx::query_as::<_, T>(query)
        .fetch_one(pool)
        .await
}
```

---

## Testing Strategies

### 1. Unit Testing
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use sqlx::PgPool;
    use testcontainers::{clients::Cli, images::postgres::Postgres, Docker};
    
    struct TestContext {
        db: PgPool,
        _container: testcontainers::Container<'static, Postgres>,
    }
    
    impl TestContext {
        async fn new() -> Self {
            let docker = Cli::default();
            let container = docker.run(Postgres::default());
            let connection_string = format!(
                "postgres://postgres:password@127.0.0.1:{}/postgres",
                container.get_host_port_ipv4(5432)
            );
            
            let db = PgPoolOptions::new()
                .max_connections(1)
                .connect(&connection_string)
                .await
                .expect("Failed to connect to test database");
                
            // Run migrations
            sqlx::migrate!("./migrations").run(&db).await.unwrap();
            
            Self {
                db,
                _container: container,
            }
        }
    }
    
    #[tokio::test]
    async fn test_create_patient() {
        let ctx = TestContext::new().await;
        let repository = PostgresPatientRepository::new(Arc::new(ctx.db));
        
        let request = CreatePatientRequest {
            abha_id: "91-1234-5678-9012".to_string(),
            name: "John Doe".to_string(),
            email: "john@example.com".to_string(),
            phone: "+91-9876543210".to_string(),
            date_of_birth: NaiveDate::from_ymd_opt(1990, 1, 1).unwrap(),
            gender: Some("male".to_string()),
            address: None,
        };
        
        let patient = repository.create(request).await.unwrap();
        
        assert_eq!(patient.name, "John Doe");
        assert_eq!(patient.abha_id, "91-1234-5678-9012");
        assert_eq!(patient.email, "john@example.com");
    }
    
    #[tokio::test]
    async fn test_find_patient_by_abha_id() {
        let ctx = TestContext::new().await;
        let repository = PostgresPatientRepository::new(Arc::new(ctx.db));
        
        // Create test patient
        let request = CreatePatientRequest {
            abha_id: "91-1234-5678-9013".to_string(),
            name: "Jane Doe".to_string(),
            email: "jane@example.com".to_string(),
            phone: "+91-9876543211".to_string(),
            date_of_birth: NaiveDate::from_ymd_opt(1985, 5, 15).unwrap(),
            gender: Some("female".to_string()),
            address: None,
        };
        
        let created_patient = repository.create(request).await.unwrap();
        
        // Find by ABHA ID
        let found_patient = repository
            .find_by_abha_id("91-1234-5678-9013")
            .await
            .unwrap()
            .unwrap();
        
        assert_eq!(found_patient.id, created_patient.id);
        assert_eq!(found_patient.name, "Jane Doe");
    }
}
```

### 2. Integration Testing
```rust
#[cfg(test)]
mod integration_tests {
    use super::*;
    use axum_test::TestServer;
    
    async fn create_test_server() -> TestServer {
        let ctx = TestContext::new().await;
        let state = AppState {
            db: Arc::new(ctx.db),
            redis: Arc::new(create_test_redis().await),
            config: Arc::new(load_test_config()),
        };
        
        let app = create_app(state);
        TestServer::new(app).unwrap()
    }
    
    #[tokio::test]
    async fn test_create_patient_endpoint() {
        let server = create_test_server().await;
        
        let request_body = serde_json::json!({
            "abha_id": "91-1234-5678-9014",
            "name": "Test Patient",
            "email": "test@example.com",
            "phone": "+91-9876543212",
            "date_of_birth": "1990-01-01",
            "gender": "male"
        });
        
        let response = server
            .post("/api/v1/patients")
            .json(&request_body)
            .await;
        
        response.assert_status_created();
        response.assert_json_contains(&serde_json::json!({
            "name": "Test Patient",
            "abha_id": "91-1234-5678-9014"
        }));
    }
    
    #[tokio::test]
    async fn test_authentication_required() {
        let server = create_test_server().await;
        
        let response = server
            .get("/api/v1/patients")
            .await;
        
        response.assert_status_unauthorized();
    }
    
    #[tokio::test]
    async fn test_patient_list_with_auth() {
        let server = create_test_server().await;
        
        // Create test JWT
        let token = create_test_jwt(UserRole::Provider).await;
        
        let response = server
            .get("/api/v1/patients")
            .add_header("Authorization", format!("Bearer {}", token))
            .await;
        
        response.assert_status_ok();
        response.assert_json_contains(&serde_json::json!({
            "data": [],
            "total": 0,
            "page": 1
        }));
    }
}
```

### 3. Load Testing
```rust
// Load testing with criterion
use criterion::{black_box, criterion_group, criterion_main, Criterion};

async fn benchmark_patient_creation(c: &mut Criterion) {
    let rt = tokio::runtime::Runtime::new().unwrap();
    let ctx = rt.block_on(TestContext::new());
    let repository = PostgresPatientRepository::new(Arc::new(ctx.db));
    
    c.bench_function("create_patient", |b| {
        b.to_async(&rt).iter(|| async {
            let request = CreatePatientRequest {
                abha_id: format!("91-1234-5678-{}", fastrand::u16(1000..9999)),
                name: "Benchmark Patient".to_string(),
                email: format!("bench-{}@example.com", fastrand::u32(..)),
                phone: format!("+91-98765432{:02}", fastrand::u8(10..99)),
                date_of_birth: NaiveDate::from_ymd_opt(1990, 1, 1).unwrap(),
                gender: Some("male".to_string()),
                address: None,
            };
            
            black_box(repository.create(request).await.unwrap())
        })
    });
}

criterion_group!(benches, benchmark_patient_creation);
criterion_main!(benches);
```

---

## Deployment & DevOps

### 1. Docker Configuration
```dockerfile
# Multi-stage Dockerfile for Rust backend
FROM rust:1.75 AS builder

WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
COPY migrations ./migrations

# Build for release
RUN cargo build --release

# Runtime stage
FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl3 \
    libpq5 \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Copy the binary from builder stage
COPY --from=builder /app/target/release/healthtech-backend ./
COPY --from=builder /app/migrations ./migrations

# Create non-root user
RUN useradd -r -s /bin/false healthtech
USER healthtech

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["./healthtech-backend"]
```

### 2. Kubernetes Deployment
```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: healthtech-backend
  labels:
    app: healthtech-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: healthtech-backend
  template:
    metadata:
      labels:
        app: healthtech-backend
    spec:
      containers:
      - name: healthtech-backend
        image: healthtech/backend:latest
        ports:
        - containerPort: 8080
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
        resources:
          requests:
            memory: "256Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      imagePullSecrets:
      - name: regcred
```

### 3. CI/CD Pipeline
```yaml
# .github/workflows/backend.yml
name: Backend CI/CD

on:
  push:
    branches: [main, develop]
    paths: ['backend/**']
  pull_request:
    branches: [main]
    paths: ['backend/**']

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
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
