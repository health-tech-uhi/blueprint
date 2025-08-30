# **App Flowchart**

## **High-Level System Architecture**

```mermaid
graph TB
    %% Frontend Clients
    subgraph "Frontend Clients"
        WEB[Desktop Web Client<br/>React + TypeScript]
        IOS[iOS Client<br/>Swift + SwiftUI]
        AND[Android Client<br/>Kotlin + Jetpack Compose]
    end

    %% Backend for Frontend
    subgraph "Backend for Frontend Layer"
        BFF[BFF Service<br/>Tonic gRPC + Rust]
    end

    %% Core Backend Services
    subgraph "Core Backend Services"
        IAM[Identity & Access Management<br/>Rust + JWT]
        ORG[Organization Management<br/>Rust + Domain Logic]
        APT[Appointment Management<br/>Rust + Event Sourcing]
        PAY[Payment Service<br/>Rust + Financial Patterns]
        NOT[Notification Service<br/>Rust + Pub/Sub]
        DISC[Discussion Forum<br/>Rust + Real-time]
        SUB[Subscription Service<br/>Rust + Billing Logic]
    end

    %% Data Layer
    subgraph "Data & Storage Layer"
        SUPA[(Supabase<br/>PostgreSQL + RLS)]
        REDIS[(Redis<br/>Caching + Sessions)]
        STORAGE[Supabase Storage<br/>File Management]
    end

    %% External Services
    subgraph "External Integrations"
        UHI[UHI Gateway<br/>Healthcare Network]
        PAYMENT[Payment Gateways<br/>Razorpay, Stripe, UPI]
        SMS[SMS/Email<br/>Notification Providers]
        ABDM[ABDM Services<br/>National Health Registry]
    end

    %% Connections
    WEB --> BFF
    IOS --> BFF
    AND --> BFF

    BFF --> IAM
    BFF --> ORG
    BFF --> APT
    BFF --> PAY
    BFF --> NOT
    BFF --> DISC
    BFF --> SUB

    IAM --> SUPA
    ORG --> SUPA
    APT --> SUPA
    PAY --> SUPA
    NOT --> REDIS
    DISC --> SUPA
    SUB --> SUPA

    BFF --> REDIS
    
    ORG --> STORAGE
    APT --> STORAGE
    
    ORG --> UHI
    APT --> UHI
    PAY --> PAYMENT
    NOT --> SMS
    IAM --> ABDM
```

---

## **User Journey Flowcharts**

### **Patient User Journey**

```mermaid
graph TD
    START([Patient Opens App]) --> AUTH{Already Logged In?}
    
    AUTH -->|No| LOGIN[Login/Register Screen]
    AUTH -->|Yes| DASHBOARD[Patient Dashboard]
    
    LOGIN --> VERIFY[Verify Credentials]
    VERIFY --> DASHBOARD
    
    DASHBOARD --> SEARCH[Search Healthcare Services]
    DASHBOARD --> RECORDS[View Health Records]
    DASHBOARD --> APPOINTMENTS[My Appointments]
    DASHBOARD --> PROFILE[Profile Management]
    
    SEARCH --> FILTERS[Apply Filters<br/>Location, Specialty, Price]
    FILTERS --> RESULTS[View Search Results]
    RESULTS --> SELECT[Select Provider]
    
    SELECT --> BOOK[Book Appointment]
    BOOK --> PAYMENT[Payment Processing]
    PAYMENT --> CONFIRM[Appointment Confirmed]
    CONFIRM --> NOTIFY[Send Notifications]
    
    APPOINTMENTS --> MANAGE[Manage Existing Appointments]
    MANAGE --> RESCHEDULE[Reschedule]
    MANAGE --> CANCEL[Cancel]
    MANAGE --> JOIN[Join Teleconsultation]
    
    RECORDS --> VIEW[View Medical Records]
    RECORDS --> SHARE[Share with Provider]
    RECORDS --> DOWNLOAD[Download Records]
    
    PROFILE --> UPDATE[Update Information]
    PROFILE --> FAMILY[Manage Family Members]
    PROFILE --> PREFERENCES[Set Preferences]
```

### **Healthcare Provider User Journey**

```mermaid
graph TD
    START([Provider Opens App]) --> AUTH{Already Logged In?}
    
    AUTH -->|No| LOGIN[Login Screen]
    AUTH -->|Yes| DASHBOARD[Provider Dashboard]
    
    LOGIN --> VERIFY[Verify Credentials & Role]
    VERIFY --> DASHBOARD
    
    DASHBOARD --> SCHEDULE[Manage Schedule]
    DASHBOARD --> PATIENTS[Patient Management]
    DASHBOARD --> ANALYTICS[View Analytics]
    DASHBOARD --> SETTINGS[Organization Settings]
    
    SCHEDULE --> SLOTS[Set Available Slots]
    SCHEDULE --> BOOKINGS[View Bookings]
    SCHEDULE --> TELECONSULT[Teleconsultation Sessions]
    
    PATIENTS --> RECORDS[Patient Records]
    PATIENTS --> COMMUNICATION[Patient Communication]
    PATIENTS --> PRESCRIPTIONS[Manage Prescriptions]
    
    BOOKINGS --> APPROVE[Approve Booking]
    BOOKINGS --> MODIFY[Modify Appointment]
    BOOKINGS --> CANCEL_PROV[Cancel Appointment]
    
    TELECONSULT --> START_CALL[Start Video Call]
    TELECONSULT --> NOTES[Add Session Notes]
    TELECONSULT --> PRESCRIBE[Write Prescription]
    
    ANALYTICS --> REVENUE[Revenue Reports]
    ANALYTICS --> PATIENT_STATS[Patient Statistics]
    ANALYTICS --> PERFORMANCE[Performance Metrics]
```

---

## **Data Flow Architecture**

### **Service Discovery & Appointment Booking Flow**

```mermaid
sequenceDiagram
    participant Client
    participant BFF
    participant IAM
    participant OrgMgmt
    participant AptMgmt
    participant Payment
    participant Notification
    participant UHI
    participant Supabase

    Client->>BFF: Search for providers
    BFF->>IAM: Validate user token
    IAM-->>BFF: Token valid
    
    BFF->>OrgMgmt: Get available providers
    OrgMgmt->>UHI: Query UHI network
    UHI-->>OrgMgmt: Provider list
    OrgMgmt->>Supabase: Fetch provider details
    Supabase-->>OrgMgmt: Provider data
    OrgMgmt-->>BFF: Providers with availability
    BFF-->>Client: Search results
    
    Client->>BFF: Select provider & time slot
    BFF->>AptMgmt: Create appointment
    AptMgmt->>Supabase: Check availability
    Supabase-->>AptMgmt: Slot available
    
    AptMgmt->>Payment: Calculate charges
    Payment-->>AptMgmt: Payment amount
    AptMgmt-->>BFF: Appointment details
    BFF-->>Client: Appointment summary
    
    Client->>BFF: Confirm booking with payment
    BFF->>Payment: Process payment
    Payment->>External: Payment gateway
    External-->>Payment: Payment success
    
    Payment->>AptMgmt: Payment confirmed
    AptMgmt->>Supabase: Create appointment
    Supabase-->>AptMgmt: Appointment created
    
    AptMgmt->>Notification: Send confirmations
    Notification->>Client: Push notification
    Notification->>Provider: Email/SMS notification
    
    AptMgmt-->>BFF: Booking confirmed
    BFF-->>Client: Success response
```

### **Authentication & Authorization Flow**

```mermaid
sequenceDiagram
    participant Client
    participant BFF
    participant IAM
    participant Supabase
    participant Redis

    Client->>BFF: Login request
    BFF->>IAM: Authenticate user
    IAM->>Supabase: Validate credentials
    Supabase-->>IAM: User data
    
    IAM->>IAM: Generate JWT tokens
    IAM->>Redis: Store refresh token
    IAM-->>BFF: Access & refresh tokens
    BFF-->>Client: Login success with tokens
    
    Note over Client: Subsequent requests
    Client->>BFF: API request with JWT
    BFF->>IAM: Validate JWT
    IAM->>IAM: Verify token signature
    IAM-->>BFF: Token valid + user roles
    BFF->>Service: Forward request with context
    Service-->>BFF: Response
    BFF-->>Client: API response
    
    Note over Client: Token refresh
    Client->>BFF: Refresh token request
    BFF->>IAM: Validate refresh token
    IAM->>Redis: Check token validity
    Redis-->>IAM: Token valid
    IAM->>IAM: Generate new access token
    IAM-->>BFF: New access token
    BFF-->>Client: New token
```

---

## **Component Interaction Diagrams**

### **Frontend Client Architecture**

```mermaid
graph TD
    subgraph "React Frontend"
        UI[UI Components]
        STATE[State Management<br/>Redux Toolkit]
        API[API Layer<br/>RTK Query]
        AUTH[Auth Context]
        ROUTER[React Router]
    end
    
    subgraph "Mobile Clients"
        IOS_UI[SwiftUI Views]
        IOS_VM[ViewModels]
        IOS_NET[Network Layer]
        
        AND_UI[Compose UI]
        AND_VM[ViewModels]
        AND_NET[Retrofit + OkHttp]
    end
    
    UI --> STATE
    STATE --> API
    API --> BFF[Backend for Frontend]
    AUTH --> API
    ROUTER --> UI
    
    IOS_UI --> IOS_VM
    IOS_VM --> IOS_NET
    IOS_NET --> BFF
    
    AND_UI --> AND_VM
    AND_VM --> AND_NET
    AND_NET --> BFF
```

### **Backend Services Communication**

```mermaid
graph LR
    subgraph "gRPC Internal Network"
        BFF[BFF Service] 
        IAM[IAM Service]
        ORG[Org Management]
        APT[Appointment Service]
        PAY[Payment Service]
        NOT[Notification Service]
    end
    
    BFF -.->|gRPC| IAM
    BFF -.->|gRPC| ORG
    BFF -.->|gRPC| APT
    BFF -.->|gRPC| PAY
    
    APT -.->|gRPC| NOT
    PAY -.->|gRPC| NOT
    ORG -.->|gRPC| IAM
    
    subgraph "Data Layer"
        SUPA[(Supabase)]
        REDIS[(Redis)]
    end
    
    IAM --> SUPA
    ORG --> SUPA
    APT --> SUPA
    PAY --> SUPA
    NOT --> REDIS
    BFF --> REDIS
```

---

## **Real-Time Communication Flow**

### **Teleconsultation Session Flow**

```mermaid
sequenceDiagram
    participant Patient
    participant Provider
    participant BFF
    participant AptMgmt
    participant VideoService
    participant Notification

    Patient->>BFF: Join consultation
    Provider->>BFF: Join consultation
    
    BFF->>AptMgmt: Validate appointment
    AptMgmt-->>BFF: Appointment valid
    
    BFF->>VideoService: Create session
    VideoService-->>BFF: Session details
    
    BFF-->>Patient: Session credentials
    BFF-->>Provider: Session credentials
    
    Patient->>VideoService: Connect to session
    Provider->>VideoService: Connect to session
    
    VideoService->>Notification: Session started
    Notification->>BFF: Update appointment status
    
    Note over Patient,Provider: Video/Audio Communication
    
    Provider->>BFF: End consultation
    BFF->>AptMgmt: Update appointment status
    BFF->>Notification: Send completion notifications
    
    Notification->>Patient: Consultation completed
    Notification->>Provider: Session summary
```

### **Real-Time Notification Flow**

```mermaid
graph TD
    EVENT[System Event] --> TRIGGER[Notification Trigger]
    
    TRIGGER --> QUEUE[Redis Pub/Sub Queue]
    QUEUE --> PROCESSOR[Notification Processor]
    
    PROCESSOR --> FILTER[Apply User Preferences]
    FILTER --> ROUTE[Route by Channel]
    
    ROUTE --> PUSH[Push Notification]
    ROUTE --> EMAIL[Email Service]
    ROUTE --> SMS[SMS Service]
    ROUTE --> WEBSOCKET[WebSocket]
    
    PUSH --> MOBILE[Mobile Devices]
    EMAIL --> INBOX[Email Inbox]
    SMS --> PHONE[Phone Number]
    WEBSOCKET --> WEBAPP[Web Application]
    
    MOBILE --> DELIVERY[Delivery Confirmation]
    INBOX --> DELIVERY
    PHONE --> DELIVERY
    WEBAPP --> DELIVERY
    
    DELIVERY --> ANALYTICS[Notification Analytics]
```

---

## **Error Handling & Recovery Flow**

### **Service Failure Recovery**

```mermaid
graph TD
    REQUEST[Incoming Request] --> VALIDATE[Validate Request]
    VALIDATE -->|Invalid| ERROR[Return Error Response]
    VALIDATE -->|Valid| PROCESS[Process Request]
    
    PROCESS --> SERVICE_CALL[Call Backend Service]
    SERVICE_CALL -->|Success| RESPONSE[Return Success Response]
    SERVICE_CALL -->|Timeout| RETRY[Retry Logic]
    SERVICE_CALL -->|Service Down| FALLBACK[Fallback Mechanism]
    
    RETRY -->|Max Retries| FALLBACK
    RETRY -->|Success| RESPONSE
    
    FALLBACK --> CACHE[Check Cache]
    CACHE -->|Found| CACHED_RESPONSE[Return Cached Data]
    CACHE -->|Not Found| DEGRADED[Degraded Service Response]
    
    ERROR --> LOG[Log Error]
    DEGRADED --> LOG
    
    LOG --> METRICS[Update Metrics]
    METRICS --> ALERT[Alert if Threshold Exceeded]
```

### **Data Consistency Flow**

```mermaid
sequenceDiagram
    participant Client
    participant BFF
    participant Service1
    participant Service2
    participant Database
    participant EventBus

    Client->>BFF: Complex operation request
    BFF->>Service1: Begin transaction
    Service1->>Database: Create transaction
    
    Service1->>Service2: Call dependent service
    Service2->>Database: Update related data
    
    alt Success Path
        Service2-->>Service1: Success
        Service1->>Database: Commit transaction
        Database-->>Service1: Transaction committed
        Service1->>EventBus: Publish success event
        Service1-->>BFF: Operation successful
        BFF-->>Client: Success response
    else Failure Path
        Service2-->>Service1: Error
        Service1->>Database: Rollback transaction
        Database-->>Service1: Transaction rolled back
        Service1->>EventBus: Publish failure event
        Service1-->>BFF: Operation failed
        BFF-->>Client: Error response
    end
    
    EventBus->>Service1: Process compensating actions
    EventBus->>Service2: Process compensating actions
```

---

## **Performance Optimization Flow**

### **Caching Strategy**

```mermaid
graph TD
    REQUEST[Client Request] --> CACHE_CHECK[Check Redis Cache]
    
    CACHE_CHECK -->|Hit| VALIDATE_TTL[Validate TTL]
    CACHE_CHECK -->|Miss| DATABASE[Query Database]
    
    VALIDATE_TTL -->|Valid| RETURN_CACHED[Return Cached Data]
    VALIDATE_TTL -->|Expired| DATABASE
    
    DATABASE --> PROCESS[Process Database Result]
    PROCESS --> UPDATE_CACHE[Update Cache]
    UPDATE_CACHE --> RETURN_FRESH[Return Fresh Data]
    
    RETURN_CACHED --> CLIENT[Send to Client]
    RETURN_FRESH --> CLIENT
    
    CLIENT --> ANALYTICS[Update Cache Analytics]
```

### **Load Balancing & Scaling**

```mermaid
graph TD
    TRAFFIC[Incoming Traffic] --> LB[Load Balancer]
    
    LB --> K8S[Kubernetes Cluster]
    
    subgraph "Auto-Scaling"
        K8S --> HPA[Horizontal Pod Autoscaler]
        HPA --> METRICS[Monitor CPU/Memory]
        METRICS -->|High Load| SCALE_UP[Scale Up Pods]
        METRICS -->|Low Load| SCALE_DOWN[Scale Down Pods]
    end
    
    subgraph "Pod Distribution"
        SCALE_UP --> POD1[Service Pod 1]
        SCALE_UP --> POD2[Service Pod 2]
        SCALE_UP --> POD3[Service Pod N]
    end
    
    POD1 --> HEALTH[Health Check]
    POD2 --> HEALTH
    POD3 --> HEALTH
    
    HEALTH -->|Healthy| ROUTE[Route Traffic]
    HEALTH -->|Unhealthy| REMOVE[Remove from Pool]
```

---

This comprehensive flowchart documentation provides a clear visual representation of the system architecture, user journeys, data flows, and operational procedures for the health tech platform.
