# **Tech Stack**

## **System Design Overview**
The health tech platform follows a **microservices architecture** designed as a comprehensive, ABDM-compliant healthcare ecosystem. Built on the Universal Health Interface (UHI) standards and Ayushman Bharat Digital Mission (ABDM) framework, this platform serves as a unified, interoperable, and scalable solution for India's digital health transformation.

### **Architecture Pattern**
- **ABDM-Compliant Microservices**: Each service aligns with ABDM building blocks and healthcare standards
- **Backend-for-Frontend (BFF)**: Optimizes client-specific interactions with healthcare workflows
- **gRPC Communication**: High-performance inter-service communication with healthcare data security
- **Event-Driven Architecture**: Asynchronous processing for scalability and real-time healthcare operations
- **FHIR R4 Integration**: Healthcare interoperability using HL7 FHIR R4 standards
- **UHI Protocol Implementation**: Full compliance with Universal Health Interface specifications
- **Healthcare-Grade Security**: End-to-end encryption with healthcare data privacy compliance

### **ABDM Building Blocks Integration**
- **ABHA (Ayushman Bharat Health Account)**: 14-digit health ID management and verification
- **Health Facility Registry (HFR)**: Comprehensive facility registration and service catalog
- **Healthcare Professional Registry (HPR)**: Professional verification and credential management
- **Personal Health Records (PHR)**: Interoperable health record management with consent
- **UHI Protocol**: Healthcare service discovery and transaction processing
- **NHCX Integration**: National Health Claims Exchange for insurance processing

---

## **Frontend Technologies**

### **Desktop Client (Web Application)**
- **Framework**: React 18+ with TypeScript
- **Build Tool**: Vite for fast development and optimized builds
- **Styling**: Tailwind CSS for consistent design system
- **State Management**: Redux Toolkit with RTK Query
- **Routing**: React Router v6
- **UI Components**: Custom component library built on headless UI
- **Testing**: Jest + React Testing Library
- **Bundle Analysis**: webpack-bundle-analyzer

**Why React + TypeScript?**
- Type safety reduces runtime errors
- Large ecosystem and community support
- Excellent developer experience
- Easy integration with BFF via REST/GraphQL APIs

### **iOS Client**
- **Language**: Swift 5.7+
- **Framework**: SwiftUI for declarative UI
- **Architecture**: MVVM (Model-View-ViewModel)
- **Networking**: Combine framework for reactive programming
- **Data Persistence**: Core Data with CloudKit sync
- **Testing**: XCTest framework
- **CI/CD**: Xcode Cloud integration

**Why SwiftUI?**
- Native iOS performance and feel
- Declarative syntax for maintainable code
- Seamless integration with iOS ecosystem
- Built-in accessibility support

### **Android Client**
- **Language**: Kotlin with coroutines
- **Framework**: Jetpack Compose for modern UI
- **Architecture**: MVVM with LiveData/StateFlow
- **Dependency Injection**: Hilt (Dagger-based)
- **Networking**: Retrofit with OkHttp
- **Database**: Room with SQLite
- **Testing**: JUnit + Espresso
- **Build System**: Gradle with Kotlin DSL

**Why Jetpack Compose?**
- Modern declarative UI toolkit
- Reduced boilerplate code
- Better performance than traditional Views
- Excellent interop with existing Android code

---

## **Backend Technologies**

### **Core Backend Language**
- **Rust**: Primary backend language for performance and safety
- **Version**: Rust 1.70+ with 2021 edition

### **gRPC Framework**
- **Tonic**: High-performance gRPC framework for Rust
- **Tower**: Middleware and service abstractions for gRPC services
- **Prost**: Protocol Buffers implementation for Rust

**Why Rust?**
- Memory safety without garbage collection
- Zero-cost abstractions for high performance
- Excellent concurrency support with async/await
- Strong type system prevents many runtime errors
- Growing ecosystem for gRPC and microservices

### **Backend Services Architecture**

#### **1. Backend-for-Frontend (BFF)**
- **Framework**: Tonic gRPC service with Tower middleware
- **Purpose**: Aggregates and optimizes data for specific clients via gRPC
- **Features**: gRPC request routing, response transformation, caching
- **Client Communication**: Exposes REST APIs to frontend clients while using gRPC internally

#### **2. Identity and Access Management (IAM)**
- **Framework**: Tonic gRPC service with custom auth interceptors
- **Authentication**: JWT tokens with RS256 signing
- **Authorization**: Role-Based Access Control (RBAC)
- **Features**: User registration, login, password reset, MFA
- **Communication**: Pure gRPC for all inter-service communication

#### **3. Organization Management**
- **Framework**: Tonic gRPC service with domain-driven design
- **Purpose**: Manage healthcare providers, clinics, hospitals
- **Features**: CRUD operations, verification workflows, staff management
- **Communication**: gRPC APIs for all service interactions

#### **4. Appointment Management**
- **Framework**: Tonic gRPC service with event sourcing
- **Purpose**: Handle scheduling and booking
- **Features**: Availability management, booking workflows, notifications
- **Communication**: gRPC for real-time appointment updates

#### **5. Payment Service**
- **Framework**: Tonic gRPC service with financial transaction patterns
- **Purpose**: Process payments and billing
- **Integrations**: Razorpay, Stripe, UPI gateways (via HTTP clients within gRPC service)
- **Communication**: Secure gRPC for payment processing

#### **6. Notification Service**
- **Framework**: Tonic gRPC service with pub/sub patterns
- **Purpose**: Handle real-time notifications
- **Features**: Push notifications, email, SMS coordination
- **Communication**: gRPC streaming for real-time notification delivery

#### **7. Electronic Health Records (EHR) Service**
- **Framework**: Tonic gRPC service with FHIR R4 compliance
- **Purpose**: Comprehensive health record management and storage
- **Features**: FHIR-compliant data storage, document management, secure sharing, audit logging
- **Integrations**: FHIR R4 server, medical imaging support, lab result integration
- **Communication**: gRPC for structured data operations, secure file transfer protocols

#### **8. Discussion Forum Service**
- **Framework**: Tonic gRPC service with real-time messaging patterns
- **Purpose**: Community features and patient-provider communication
- **Features**: Threaded discussions, moderation tools, search functionality, real-time chat
- **Integrations**: Notification service for real-time updates, IAM for permissions
- **Communication**: gRPC with WebSocket support for real-time messaging

#### **9. UHI Gateway Service**
- **Framework**: Tonic gRPC service with UHI protocol implementation
- **Purpose**: Universal Health Interface protocol compliance and integration
- **Features**: UHI message routing, ed25519 cryptographic signing, network registry integration
- **Standards**: Full UHI protocol compliance, ABDM integration, healthcare interoperability
- **Communication**: gRPC for internal communication, HTTP/JSON for UHI protocol endpoints

#### **10. Analytics Service**
- **Framework**: Tonic gRPC service with data processing pipelines
- **Purpose**: Advanced analytics, reporting, and business intelligence
- **Features**: Real-time dashboards, custom report generation, predictive analytics, data visualization
- **Integrations**: All microservices for data collection, external BI tools
- **Communication**: gRPC for data queries, GraphQL endpoints for complex analytics queries

#### **11. ABHA Service**
- **Framework**: Tonic gRPC service with ABDM integration
- **Purpose**: Ayushman Bharat Health Account management and verification
- **Features**: 14-digit Health ID creation, Aadhaar-based verification, QR code generation, demographic validation
- **ABDM Integration**: Government database integration, real-time ABHA verification, face authentication
- **Communication**: gRPC for internal operations, REST APIs for ABDM gateway integration

#### **12. Healthcare Professional Registry (HPR) Service**
- **Framework**: Tonic gRPC service with professional credential management
- **Purpose**: Healthcare professional verification and registry management
- **Features**: HPRID creation, digital license verification, credential validation, continuing education tracking
- **Professional Networks**: Peer collaboration, specialty certification management, medical council integration
- **Communication**: gRPC for registry operations, secure API integration with medical councils

#### **13. Health Facility Registry (HFR) Service**
- **Framework**: Tonic gRPC service with facility management
- **Purpose**: Healthcare facility registration and service catalog management
- **Features**: Facility onboarding, accreditation tracking, capacity management, multi-location coordination
- **Regulatory Compliance**: Automated compliance monitoring, quality certification tracking
- **Communication**: gRPC for facility operations, integration with regulatory bodies

#### **14. NHCX Integration Service**
- **Framework**: Tonic gRPC service with claims processing workflows
- **Purpose**: National Health Claims Exchange integration for insurance processing
- **Features**: Real-time claims processing, preauthorization workflows, payment reconciliation, policy management
- **FHIR Compliance**: NHCX-compliant claim bundles, billing artifacts, payment processing
- **Communication**: gRPC for internal claims workflows, secure NHCX protocol endpoints

#### **15. Consent Management Service**
- **Framework**: Tonic gRPC service with privacy-first design
- **Purpose**: Healthcare data consent management and patient privacy control
- **Features**: Granular consent artifacts, care context linking, data sharing agreements, consent revocation
- **Privacy Controls**: FHIR-compliant consent management, audit trails, patient data access logs
- **Communication**: gRPC for consent operations, integration with all healthcare data services

#### **16. Clinical Decision Support Service**
- **Framework**: Tonic gRPC service with AI/ML integration
- **Purpose**: AI-powered clinical decision support and diagnostic assistance
- **Features**: Symptom assessment, differential diagnosis, drug interaction checking, treatment recommendations
- **AI Integration**: Machine learning models for predictive analytics, natural language processing for clinical notes
- **Communication**: gRPC for clinical support queries, REST APIs for external AI model integration

#### **17. Telemedicine Service**
- **Framework**: Tonic gRPC service with real-time communication
- **Purpose**: Comprehensive telemedicine and remote consultation platform
- **Features**: Video/audio consultations, session recording, remote monitoring, digital prescriptions
- **Integration**: WebRTC for real-time communication, EHR integration for consultation records
- **Communication**: gRPC for session management, WebSocket for real-time video/audio streams

#### **18. Emergency Response Service**
- **Framework**: Tonic gRPC service with critical care workflows
- **Purpose**: Emergency healthcare coordination and critical patient management
- **Features**: Emergency contact notification, ambulance dispatch, hospital bed allocation, critical data access
- **Emergency Protocols**: Real-time patient monitoring, disaster response coordination, mass casualty management
- **Communication**: gRPC for emergency coordination, priority message queues for critical alerts

---

## **Database & Storage**

### **Primary Database**
- **Supabase**: Managed PostgreSQL with built-in features
- **Version**: PostgreSQL 15+
- **Features**: 
  - Row-Level Security (RLS)
  - Real-time subscriptions
  - Auto-generated REST APIs
  - Built-in authentication

### **Database Access**
- **ORM**: SQLx for compile-time verified queries
- **Migrations**: SQLx migrations for schema versioning
- **Connection Pooling**: sqlx::Pool for efficient connections

### **Caching Layer**
- **Redis**: In-memory caching and session storage
- **Use Cases**: Session management, API response caching, rate limiting

### **File Storage**
- **Supabase Storage**: For user uploads and documents
- **CDN**: Integrated CDN for global file delivery
- **Security**: Signed URLs for secure file access

**Why Supabase?**
- Managed PostgreSQL reduces operational overhead
- Built-in real-time capabilities
- Row-level security for fine-grained access control
- Integrated authentication and storage
- Good Rust ecosystem support

---

## **Communication Protocols**

### **Inter-Service Communication**
- **gRPC**: High-performance RPC for service-to-service communication
- **Protocol Buffers**: Efficient serialization format
- **HTTP/2**: For transport layer efficiency

### **Client-Server Communication**
- **REST APIs**: BFF service exposes REST APIs to frontend clients
- **gRPC-Web**: Direct gRPC communication for advanced web clients (implemented in Phase 2)
- **WebSocket**: For real-time features (chat, notifications) exposed by BFF
- **GraphQL**: Complex data fetching for analytics dashboard (implemented in Phase 2)

#### **GraphQL Implementation Details**
- **Framework**: Async-GraphQL for Rust-based GraphQL server
- **Schema**: Auto-generated from gRPC service definitions
- **Features**: Real-time subscriptions, DataLoader pattern for N+1 prevention
- **Use Cases**: Analytics dashboard, complex reporting queries, admin interfaces
- **Integration**: BFF service acts as GraphQL gateway, translating to gRPC calls

#### **gRPC-Web Implementation Details**
- **Framework**: Tonic-Web for direct browser-to-service communication
- **Features**: Streaming support, TypeScript client generation, CORS handling
- **Use Cases**: Advanced web clients requiring real-time data streams
- **Security**: JWT authentication, same security model as REST APIs

### **Message Queue**
- **Redis Pub/Sub**: For simple event broadcasting
- **Alternative**: Consider Apache Kafka for high-throughput scenarios

---

## **Security & Authentication**

### **Authentication Strategy**
- **Custom IAM Service**: Built in Rust for security and performance
- **JWT Tokens**: RS256 algorithm for token signing
- **Refresh Tokens**: Long-lived tokens for session management
- **Multi-Factor Authentication**: TOTP and SMS-based MFA

### **Authorization**
- **Role-Based Access Control (RBAC)**: Fine-grained permissions for user roles (patient, provider, admin, staff)
- **Attribute-Based Access Control (ABAC)**: Context-aware permissions based on user attributes, resource properties, and environmental factors
- **API Key Management**: Secure service-to-service authentication with rotation and scope management

#### **RBAC Implementation Details**
- **Roles**: Patient, Healthcare Provider, Clinic Admin, Hospital Admin, System Admin
- **Permissions**: Granular permissions for each resource and operation
- **Inheritance**: Hierarchical role inheritance for efficient permission management
- **Dynamic Assignment**: Runtime role assignment based on organization membership

#### **ABAC Implementation Details**
- **Attributes**: User (role, department, clearance), Resource (type, sensitivity, owner), Environment (time, location, device)
- **Policies**: XACML-inspired policy language for complex authorization rules
- **Decision Engine**: Real-time policy evaluation with caching for performance
- **Use Cases**: Healthcare data access based on patient consent, time-based access, location-restricted operations

#### **API Key Management Details**
- **Key Types**: Service keys, application keys, temporary access keys
- **Scopes**: Fine-grained scopes for different API endpoints and operations
- **Rotation**: Automatic key rotation with configurable intervals
- **Monitoring**: Comprehensive logging and monitoring of API key usage
- **Revocation**: Immediate key revocation with distributed cache invalidation

### **Security Measures**
- **HTTPS Everywhere**: TLS 1.3 for all communications
- **API Rate Limiting**: Prevent abuse and DDoS
- **Input Validation**: Comprehensive input sanitization
- **CORS Configuration**: Proper cross-origin policies

---

## **Development & Deployment**

### **Containerization**
- **Docker**: All services containerized for consistency
- **Multi-stage Builds**: Optimized container images
- **Base Images**: Distroless images for security

### **Orchestration**
- **Kubernetes**: Container orchestration for production
- **Helm Charts**: Package management for Kubernetes deployments
- **Ingress Controllers**: NGINX or Traefik for traffic management

### **CI/CD Pipeline**
- **GitHub Actions**: Automated testing and deployment
- **Cargo**: Rust build system and package manager
- **Testing**: Unit tests, integration tests, and end-to-end tests

### **Monitoring & Observability**
- **Prometheus**: Metrics collection and alerting (gRPC interceptors + metrics endpoint)
- **Grafana**: Metrics visualization and dashboards
- **Jaeger**: Distributed tracing for gRPC services
- **OpenTelemetry**: Rust tracing instrumentation for gRPC
- **Structured Logging**: JSON-based logging with correlation IDs via `tracing` crate
- **Health Checks**: gRPC health checking protocol or minimal HTTP sidecar

---

## **UHI Compliance**

### **UHI Protocol Integration**
- **Gateway Service**: Implementation of UHI gateway patterns
- **API Standards**: Compliance with UHI API specifications
- **Message Formats**: JSON-based message exchange
- **Authentication**: ed25519 cryptographic signing for UHI messages

### **ABDM & Healthcare Standards Compliance**

#### **FHIR R4 Implementation**
- **Core Resources**: Patient, Observation, DiagnosticReport, Medication, Encounter, Practitioner, Organization
- **Clinical Artifacts**: 7 core FHIR resources for clinical document exchange
- **Billing Artifacts**: NHCX-compliant claim bundles and payment processing
- **Validation Engine**: HAPI FHIR library integration for resource validation
- **Terminology Services**: SNOMED CT, ICD-10, and LOINC code validation

#### **UHI Gateway & Protocol Implementation**
- **ed25519 Cryptographic Signing**: Message authentication and integrity verification
- **RSA-2048 Encryption**: Secure data transmission and key exchange
- **JWT Token Management**: Session handling and API authentication
- **ECDH Key Exchange**: Secure communication establishment
- **Message Routing**: Intelligent message routing based on UHI specifications

#### **ABDM Sandbox Integration**
- **Development Environment**: Access to ABDM sandbox for development and testing
- **API Testing Framework**: Comprehensive testing of UHI protocol compliance
- **Interoperability Validation**: Cross-platform data exchange verification
- **Performance Benchmarking**: Load testing and scalability validation

#### **Healthcare Data Standards**
- **HL7 FHIR R4**: International healthcare data exchange standards
- **SNOMED CT**: Clinical terminology and coding system
- **ICD-10**: International classification of diseases
- **LOINC**: Laboratory data identification and exchange
- **DICOM**: Medical imaging data format and communication

#### **Regulatory Compliance**
- **HIPAA-Equivalent Privacy**: Healthcare data protection measures
- **GDPR Compliance**: European data protection regulation adherence
- **Data Localization**: Healthcare data storage within Indian jurisdiction
- **Audit Trail**: Comprehensive logging for regulatory compliance
- **Consent Management**: Patient consent tracking and management

#### **Security Standards**
- **Healthcare-Grade Encryption**: AES-256 for data at rest, TLS 1.3 for transit
- **Zero-Trust Architecture**: Continuous verification and minimal privilege access
- **Biometric Authentication**: Face authentication and multi-factor security
- **Healthcare Audit Logging**: Comprehensive security event tracking
- **PHI Protection**: Protected Health Information security measures

---

## **Technology Justification**

### **Performance Benefits**
- **Rust Backend**: Zero-cost abstractions and memory safety
- **gRPC**: Binary protocol with HTTP/2 multiplexing
- **Supabase**: Managed database with connection pooling
- **Redis Caching**: Sub-millisecond response times

### **Scalability Features**
- **Microservices**: Independent scaling of services
- **Kubernetes**: Horizontal pod autoscaling
- **Database Sharding**: Future-ready data distribution
- **CDN Integration**: Global content delivery

### **Developer Experience**
- **TypeScript**: Type safety across the stack
- **Hot Reloading**: Fast development iteration
- **Comprehensive Testing**: Automated quality assurance
- **Documentation**: Auto-generated API documentation

### **Security First**
- **Memory Safety**: Rust prevents entire classes of vulnerabilities
- **Encrypted Communication**: End-to-end encryption
- **Access Controls**: Fine-grained permission systems
- **Audit Logging**: Comprehensive security event tracking

---

## **Migration Strategy**

### **Phase 1: Core Services**
- Set up basic infrastructure (databases, containers)
- Implement IAM and BFF services
- Deploy basic frontend clients

### **Phase 2: Feature Services**
- Add appointment and organization management
- Implement payment and notification services
- Enable UHI protocol compliance

### **Phase 3: Advanced Features**
- Add analytics and reporting
- Implement real-time communication
- Deploy monitoring and observability stack

### **Phase 4: Optimization**
- Performance tuning and optimization
- Advanced security hardening
- Compliance certification
