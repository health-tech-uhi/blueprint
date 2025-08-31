# **ABDM-Compliant Healthcare Platform Project Rules**

## **Healthcare-Specific Development Standards**

### **ABDM Ecosystem Compliance**

#### **ABHA (Ayushman Bharat Health Account) Integration**
- **Health ID Management**: All user authentication must support 14-digit ABHA Health IDs
- **Demographic Verification**: Real-time validation with ABDM demographic services
- **Consent Management**: Implement granular consent mechanisms for health data sharing
- **QR Code Generation**: Support offline QR codes for healthcare provider scanning

#### **UHI Protocol Implementation**
- **Service Discovery**: Complete UHI-compliant healthcare service catalog
- **Order Management**: Standardized appointment booking and service ordering
- **Message Signing**: ed25519 cryptographic signing for all UHI messages
- **Gateway Integration**: Proper UHI Gateway routing and protocol adherence

#### **FHIR R4 Standards**
- **Resource Validation**: All clinical data must conform to FHIR R4 specifications
- **Bundle Management**: Support for Document, Message, and Collection bundles
- **Terminology Services**: Integration with SNOMED CT, ICD-10, and LOINC
- **Interoperability**: Seamless data exchange between healthcare providers

### **Healthcare Data Security Standards**

#### **Healthcare-Grade Encryption**
- **Data at Rest**: AES-256 encryption for all healthcare data storage
- **Data in Transit**: TLS 1.3 for all API communications
- **Key Management**: Healthcare-specific key rotation and management
- **Biometric Security**: Face authentication and multi-factor verification

#### **Privacy Protection**
- **Data Minimization**: Collect only necessary healthcare information
- **Purpose Limitation**: Use health data only for declared medical purposes
- **Consent Tracking**: Audit trail for all consent grants and revocations
- **Right to Erasure**: Implement secure data deletion capabilities

### **Code Quality Standards**

#### **Rust Backend Development**
- **Formatting**: Use `rustfmt` with default settings for consistent code formatting
- **Linting**: Apply `clippy` with `--deny warnings` to catch common mistakes
- **Documentation**: All public APIs must have comprehensive doc comments
- **Testing**: Minimum 90% code coverage for healthcare business logic (increased from 80%)
- **Error Handling**: Use `Result<T, E>` for all fallible operations
- **Async Patterns**: Prefer `async/await` over manual future combinators
- **Healthcare Audit**: All healthcare operations must include audit logging

```rust
// Example: Healthcare-specific error handling pattern
#[derive(Debug, thiserror::Error)]
pub enum HealthcareServiceError {
    #[error("Database error: {0}")]
    Database(#[from] sqlx::Error),
    #[error("FHIR validation error: {message}")]
    FhirValidation { message: String, resource_type: String },
    #[error("ABDM authentication error: {error_code}")]
    AbdmAuth { error_code: String },
    #[error("UHI protocol error: {0}")]
    UhiProtocol(#[from] UhiError),
    #[error("Healthcare consent required: {resource}")]
    ConsentRequired { resource: String, patient_id: String },
    #[error("HIPAA compliance violation: {details}")]
    ComplianceViolation { details: String },
}

pub type HealthcareResult<T> = Result<T, HealthcareServiceError>;
```

#### **Healthcare Microservices Standards**
- **Service Isolation**: Each healthcare domain (EHR, Consent, Analytics) in separate services
- **gRPC Communication**: Use Tonic framework for inter-service communication
- **Health Checks**: Comprehensive health endpoints for Kubernetes probes
- **Circuit Breakers**: Implement failover for critical healthcare operations
- **Rate Limiting**: Protect healthcare APIs from abuse and ensure availability

#### **Frontend Development Standards**

**React/TypeScript Rules:**
- **Type Safety**: Strict TypeScript configuration with no `any` types
- **Component Design**: Prefer functional components with hooks
- **Props Interface**: All components must have explicitly typed props
- **State Management**: Use Redux Toolkit for global state, local state for component-specific data
- **Styling**: Tailwind CSS classes only, no inline styles

```typescript
// Example: Well-typed React component
interface PatientCardProps {
  patient: Patient;
  onEdit: (patient: Patient) => void;
  className?: string;
}

const PatientCard: React.FC<PatientCardProps> = ({ 
  patient, 
  onEdit, 
  className = '' 
}) => {
  // Component implementation
};
```

**Mobile Development Standards:**

**iOS (Swift/SwiftUI):**
- Follow Apple's Human Interface Guidelines
- Use SwiftUI for all new UI development
- Implement proper accessibility support (VoiceOver)
- Use Combine for reactive programming patterns

**Android (Kotlin/Jetpack Compose):**
- Follow Material Design 3 guidelines
- Use Jetpack Compose for all new UI development
- Implement proper accessibility support (TalkBack)
- Use Kotlin coroutines for asynchronous operations

---

## **Version Control & Branching Strategy**

### **Git Workflow**
We follow the **GitFlow** branching model with the following branches:

#### **Branch Types**
- **`main`**: Production-ready code, protected branch
- **`develop`**: Integration branch for features
- **`feature/*`**: Individual feature development
- **`release/*`**: Preparation for production releases
- **`hotfix/*`**: Critical fixes for production issues

#### **Branch Naming Conventions**
```
feature/user-authentication
feature/appointment-booking
release/v1.2.0
hotfix/security-patch-v1.1.1
```

#### **Commit Message Format**
Follow [Conventional Commits](https://www.conventionalcommits.org/) specification:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**Commit Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks

**Examples:**
```
feat(auth): implement JWT token refresh mechanism
fix(appointment): resolve double booking issue
docs(api): update authentication endpoint documentation
```

### **Pull Request Guidelines**

#### **PR Requirements**
1. **Branch is up-to-date** with target branch
2. **All tests pass** in CI pipeline
3. **Code coverage** maintains or improves current levels
4. **Security scan** passes without critical issues
5. **Performance benchmarks** don't regress significantly

#### **PR Template**
```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No breaking changes (or documented)
```

#### **Review Process**
- **Minimum 2 reviewers** for code affecting core business logic
- **1 reviewer** for documentation and minor fixes
- **Security team review** for authentication/authorization changes
- **Performance team review** for database schema changes

---

## **Testing Standards**

### **Testing Pyramid**

#### **Unit Tests (70% of test suite)**
- **Purpose**: Test individual functions and methods in isolation
- **Framework**: Rust built-in test framework, Jest for TypeScript
- **Coverage**: Minimum 80% line coverage
- **Execution**: Must run in under 30 seconds total

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn test_create_appointment_success() {
        // Arrange
        let service = AppointmentService::new_mock();
        let request = CreateAppointmentRequest { /* ... */ };

        // Act
        let result = service.create_appointment(request).await;

        // Assert
        assert!(result.is_ok());
        let appointment = result.unwrap();
        assert_eq!(appointment.status, AppointmentStatus::Confirmed);
    }
}
```

#### **Integration Tests (20% of test suite)**
- **Purpose**: Test interaction between services and external dependencies
- **Framework**: Testcontainers for database testing
- **Environment**: Isolated test environment per test suite
- **Data**: Test data fixtures managed through migrations

#### **End-to-End Tests (10% of test suite)**
- **Purpose**: Test complete user workflows
- **Framework**: Playwright for web, Maestro for mobile
- **Environment**: Staging environment with production-like data
- **Scenarios**: Critical user journeys (signup, booking, payment)

### **Test Data Management**
- **Test Databases**: Separate database per test suite
- **Fixtures**: JSON/YAML files for consistent test data
- **Cleanup**: Automatic cleanup after test completion
- **Privacy**: No production data in test environments

---

## **Documentation Standards**

### **Code Documentation**

#### **API Documentation**
- **OpenAPI 3.0** specifications for all REST endpoints
- **Generated docs** from code annotations
- **Examples** for all request/response formats
- **Error codes** with detailed descriptions

#### **Code Comments**
- **Public APIs**: Complete rustdoc/JSDoc comments
- **Complex Logic**: Inline comments explaining business rules
- **TODOs**: Include ticket numbers and target dates
- **Architecture**: High-level design decisions documented

### **Technical Documentation**
- **Architecture Decision Records (ADRs)** for major technical decisions
- **Runbooks** for operational procedures
- **Deployment guides** for each environment
- **Troubleshooting guides** for common issues

### **User Documentation**
- **API Reference** with interactive examples
- **Integration guides** for healthcare providers
- **Mobile app user guides**
- **Video tutorials** for complex workflows

---

## **Performance Standards**

### **Response Time Requirements**

#### **API Performance**
- **Authentication**: < 200ms (95th percentile)
- **Search queries**: < 500ms (95th percentile)
- **CRUD operations**: < 300ms (95th percentile)
- **Payment processing**: < 2000ms (95th percentile)

#### **Frontend Performance**
- **First Contentful Paint**: < 1.5s
- **Largest Contentful Paint**: < 2.5s
- **Time to Interactive**: < 3.5s
- **Cumulative Layout Shift**: < 0.1

#### **Mobile Performance**
- **App launch time**: < 3s cold start
- **Screen transition**: < 300ms
- **Network request timeout**: 10s
- **Memory usage**: < 200MB peak

### **Scalability Requirements**
- **Concurrent users**: Support 10,000 simultaneous users
- **Database connections**: Efficient connection pooling
- **Horizontal scaling**: Auto-scaling based on CPU/memory metrics
- **Load testing**: Regular performance testing with realistic load

### **Performance Monitoring**
- **APM tools**: Prometheus + Grafana for metrics
- **Distributed tracing**: Jaeger for request tracing
- **Real user monitoring**: Frontend performance tracking
- **Alerts**: Performance degradation notifications

---

## **Security Standards**

### **Secure Development Lifecycle**

#### **Threat Modeling**
- **Regular threat assessments** for new features
- **STRIDE methodology** for systematic threat analysis
- **Security requirements** defined during design phase
- **Penetration testing** before major releases

#### **Secure Coding Practices**
- **Input validation**: All user inputs validated and sanitized
- **Output encoding**: Proper encoding for all output contexts
- **Authentication**: Strong password policies and MFA
- **Authorization**: Principle of least privilege
- **Cryptography**: Industry-standard algorithms only

### **Security Testing**
- **Static analysis**: Automated security scanning in CI/CD
- **Dependency scanning**: Regular vulnerability assessments
- **Infrastructure scanning**: Container and infrastructure security
- **Manual reviews**: Security code reviews for critical changes

### **Incident Response**
- **Security incident procedures** clearly documented
- **Escalation paths** defined for different severity levels
- **Communication plans** for user and stakeholder notification
- **Post-incident reviews** to improve security posture

---

## **Accessibility Requirements**

### **Web Accessibility (WCAG 2.1 AA)**
- **Keyboard navigation**: All functionality accessible via keyboard
- **Screen reader support**: Proper semantic HTML and ARIA labels
- **Color contrast**: Minimum 4.5:1 ratio for normal text
- **Focus indicators**: Clear visual focus indicators
- **Alternative text**: Descriptive alt text for all images

### **Mobile Accessibility**
- **iOS**: VoiceOver support with proper accessibility labels
- **Android**: TalkBack support with content descriptions
- **Touch targets**: Minimum 44pt touch targets
- **Dynamic type**: Support for user font size preferences

### **Testing**
- **Automated testing**: Lighthouse and axe-core integration
- **Manual testing**: Regular testing with assistive technologies
- **User testing**: Include users with disabilities in testing process

---

## **Monitoring & Observability**

### **Logging Standards**

#### **Structured Logging**
```rust
use tracing::{info, error, instrument};

#[instrument(skip(user_id))]
async fn create_appointment(user_id: Uuid, request: CreateAppointmentRequest) -> ServiceResult<Appointment> {
    info!(user_id = %user_id, appointment_type = %request.appointment_type, "Creating appointment");
    
    match appointment_service.create(request).await {
        Ok(appointment) => {
            info!(appointment_id = %appointment.id, "Appointment created successfully");
            Ok(appointment)
        },
        Err(error) => {
            error!(error = %error, "Failed to create appointment");
            Err(error)
        }
    }
}
```

#### **Log Levels**
- **ERROR**: System errors, exceptions, failures
- **WARN**: Unusual conditions, deprecated API usage
- **INFO**: Business events, user actions, system state changes
- **DEBUG**: Detailed debugging information (development only)

### **Metrics Collection**
- **Business metrics**: User signups, appointments booked, revenue
- **Technical metrics**: Response times, error rates, resource usage
- **Custom metrics**: Healthcare-specific KPIs
- **SLA tracking**: Availability, performance, error budgets

### **Alerting Strategy**
- **Critical alerts**: Service down, security incidents
- **Warning alerts**: Performance degradation, high error rates
- **Information alerts**: Deployment notifications, scheduled maintenance
- **Escalation**: Automated escalation for unacknowledged critical alerts

---

## **Release Management**

### **Release Process**

#### **Release Cycle**
- **Major releases**: Quarterly (breaking changes, new features)
- **Minor releases**: Monthly (new features, enhancements)
- **Patch releases**: As needed (bug fixes, security patches)
- **Hotfixes**: Emergency releases for critical issues

#### **Release Stages**
1. **Development**: Feature development in feature branches
2. **Integration**: Merge to develop branch, integration testing
3. **Release candidate**: Create release branch, final testing
4. **Staging deployment**: Deploy to staging environment
5. **Production deployment**: Deploy to production with rollback plan

### **Deployment Strategy**

#### **Blue-Green Deployment**
- **Zero-downtime deployments** using blue-green strategy
- **Database migrations** applied before application deployment
- **Health checks** verify deployment success
- **Automatic rollback** on deployment failure

#### **Feature Flags**
- **Gradual rollout** of new features using feature flags
- **A/B testing** for user experience improvements
- **Emergency shutdown** capability for problematic features
- **Metrics tracking** for feature adoption and performance

### **Rollback Procedures**
- **Automatic rollback** triggers for deployment failures
- **Manual rollback** procedures documented and tested
- **Database rollback** strategy for schema changes
- **Communication plan** for rollback notifications

---

## **Compliance & Governance**

### **Healthcare Compliance**
- **UHI compliance**: Adherence to Universal Health Interface standards
- **ABDM integration**: National Digital Health Mission requirements
- **Data privacy**: Healthcare data protection regulations
- **Audit trails**: Comprehensive logging for compliance audits

### **Code Reviews**
- **Mandatory reviews**: All code changes require review
- **Review checklist**: Security, performance, maintainability
- **Documentation updates**: Ensure docs are updated with code changes
- **Knowledge sharing**: Use reviews for team learning

### **Quality Gates**
- **Automated checks**: Linting, testing, security scanning
- **Manual verification**: Code review approval required
- **Performance validation**: No performance regressions
- **Security clearance**: Security team approval for sensitive changes

---

These project rules ensure consistent, high-quality development practices across all teams and provide clear guidelines for maintaining the health tech platform's reliability, security, and performance.
