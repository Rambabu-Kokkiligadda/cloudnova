# TaskTracker - Architecture & Design Diagrams

## Table of Contents
1. [System Architecture](#1-system-architecture)
2. [AWS Deployment Architecture](#2-aws-deployment-architecture)
3. [Application Layers](#3-application-layers)
4. [Database Schema](#4-database-schema)
5. [Multi-Tenancy Architecture](#5-multi-tenancy-architecture)
6. [Authentication Flow](#6-authentication-flow)
7. [Task Management Flow](#7-task-management-flow)
8. [Class Diagram - Domain Entities](#8-class-diagram---domain-entities)
9. [Component Diagram](#9-component-diagram)
10. [Background Services Architecture](#10-background-services-architecture)

---

## 1. System Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        Web[React Web App<br/>Material-UI]
    end
    
    subgraph "Load Balancing"
        ALB[Application Load Balancer<br/>AWS ALB]
    end
    
    subgraph "Application Layer - ECS Fargate"
        API1[API Service 1<br/>.NET 8]
        API2[API Service 2<br/>.NET 8]
        Worker[Worker Service<br/>Background Jobs]
    end
    
    subgraph "Data Layer"
        RDS[(PostgreSQL<br/>AWS RDS)]
        Redis[Redis Cache<br/>ElastiCache]
    end
    
    subgraph "Storage"
        S3[S3 Bucket<br/>File Storage]
    end
    
    subgraph "Container Registry"
        ECR[Amazon ECR<br/>Docker Images]
    end
    
    Web -->|HTTPS| ALB
    ALB -->|Round Robin| API1
    ALB -->|Round Robin| API2
    API1 -->|Query/Command| RDS
    API2 -->|Query/Command| RDS
    Worker -->|Read/Write| RDS
    API1 -->|Cache| Redis
    API2 -->|Cache| Redis
    API1 -->|Upload/Download| S3
    API2 -->|Upload/Download| S3
    ECR -->|Pull Images| API1
    ECR -->|Pull Images| API2
    ECR -->|Pull Images| Worker
    
    style Web fill:#61dafb
    style ALB fill:#ff9900
    style API1 fill:#512bd4
    style API2 fill:#512bd4
    style Worker fill:#512bd4
    style RDS fill:#336791
    style Redis fill:#dc382d
    style S3 fill:#ff9900
    style ECR fill:#ff9900
```

---

## 2. AWS Deployment Architecture

```mermaid
graph TB
    subgraph "AWS Cloud - ap-south-1"
        subgraph "VPC"
            subgraph "Public Subnet"
                ALB[Application Load Balancer<br/>tasktracker-alb]
            end
            
            subgraph "Private Subnet"
                subgraph "ECS Cluster"
                    WebService[Web Service<br/>Port 80/443]
                    APIService[API Service<br/>Port 5000]
                    WorkerService[Worker Service<br/>Background Tasks]
                end
                
                RDS[(RDS PostgreSQL 14<br/>tasktracker-db<br/>Multi-AZ)]
                Cache[ElastiCache Redis<br/>Session Store]
            end
        end
        
        ECR[ECR Repository<br/>tasktracker-web<br/>tasktracker-api]
        S3[S3 Bucket<br/>Task Attachments]
        CloudWatch[CloudWatch Logs<br/>Monitoring]
    end
    
    Internet([Internet]) -->|HTTPS| ALB
    ALB -->|HTTP| WebService
    ALB -->|HTTP| APIService
    WebService -->|REST API| APIService
    APIService -->|TCP 5432| RDS
    WorkerService -->|TCP 5432| RDS
    APIService -->|TCP 6379| Cache
    APIService -->|HTTPS| S3
    
    WebService -.->|Logs| CloudWatch
    APIService -.->|Logs| CloudWatch
    WorkerService -.->|Logs| CloudWatch
    
    ECR -.->|Pull| WebService
    ECR -.->|Pull| APIService
    ECR -.->|Pull| WorkerService
    
    style ALB fill:#ff9900
    style WebService fill:#61dafb
    style APIService fill:#512bd4
    style WorkerService fill:#512bd4
    style RDS fill:#336791
    style Cache fill:#dc382d
    style S3 fill:#ff9900
    style ECR fill:#ff9900
    style CloudWatch fill:#ff9900
```

---

## 3. Application Layers

```mermaid
graph TB
    subgraph "Presentation Layer"
        UI[React Components<br/>Material-UI<br/>TypeScript]
    end
    
    subgraph "API Layer"
        Controllers[Controllers<br/>AuthController<br/>TasksController<br/>TenantsController]
        Middleware[Middleware<br/>JWT Authentication<br/>Tenant Validation<br/>Error Handling]
    end
    
    subgraph "Application Layer"
        CQRS[CQRS Handlers<br/>Commands & Queries<br/>MediatR]
        Validators[FluentValidation<br/>Input Validation]
    end
    
    subgraph "Domain Layer"
        Entities[Domain Entities<br/>User, Tenant, Task<br/>Business Rules]
        Enums[Domain Enums<br/>TaskStatus, Priority<br/>TenantRole]
    end
    
    subgraph "Infrastructure Layer"
        Repositories[Repositories<br/>User, Tenant, Task]
        DbContext[EF Core DbContext<br/>PostgreSQL]
        Services[Services<br/>JWT, Email<br/>File Storage]
    end
    
    subgraph "Data Layer"
        DB[(PostgreSQL<br/>Database)]
    end
    
    UI -->|HTTP/REST| Controllers
    Controllers -->|Route| Middleware
    Middleware -->|Authorize| CQRS
    CQRS -->|Validate| Validators
    CQRS -->|Business Logic| Entities
    CQRS -->|Data Access| Repositories
    Repositories -->|ORM| DbContext
    DbContext -->|SQL| DB
    CQRS -->|External Services| Services
    
    style UI fill:#61dafb
    style Controllers fill:#512bd4
    style CQRS fill:#68217a
    style Entities fill:#239120
    style DbContext fill:#512bd4
    style DB fill:#336791
```

---

## 4. Database Schema

```mermaid
erDiagram
    Users ||--o{ UserTenants : "belongs to"
    Tenants ||--o{ UserTenants : "has"
    Users ||--o{ Tasks : "assigned"
    Users ||--o{ Tasks : "created"
    Tenants ||--o{ Tasks : "owns"
    Tasks ||--o{ Tasks : "parent/child"
    Tasks ||--o{ Attachments : "has"
    Tasks ||--o{ EffortEntries : "has"
    Tasks ||--o{ RemindersSent : "has"
    Tasks ||--o{ AuditEntries : "has"
    Users ||--o{ Notifications : "receives"
    Tasks ||--o{ Notifications : "about"
    
    Users {
        uuid Id PK
        string Email UK
        string FirstName
        string LastName
        string PasswordHash
        bool IsActive
        datetime LastLoginAt
        string RefreshToken
        datetime RefreshTokenExpiry
    }
    
    Tenants {
        uuid Id PK
        string Name UK
        string TimeZone
        bool IsActive
        json Settings
    }
    
    UserTenants {
        uuid UserId PK,FK
        uuid TenantId PK,FK
        int Role
        bool IsActive
        datetime JoinedAt
        datetime LeftAt
    }
    
    Tasks {
        uuid Id PK
        uuid TenantId FK
        string Title
        string Description
        int Status
        int Priority
        uuid AssignedToId FK
        uuid CreatedBy FK
        uuid ParentTaskId FK
        datetime DueDate
        int EstimatedEffort
        int ActualEffort
    }
    
    Attachments {
        uuid Id PK
        uuid TaskId FK
        uuid TenantId FK
        string FileName
        string StorageKey
        string ContentType
        long FileSize
        uuid CreatedBy FK
    }
    
    EffortEntries {
        uuid Id PK
        uuid TaskId FK
        uuid TenantId FK
        uuid UserId FK
        int Hours
        datetime Date
        string Description
    }
    
    RemindersSent {
        uuid Id PK
        uuid TaskId FK
        datetime SentAt
        datetime ReminderWindow
        bool IsSuccessful
    }
    
    AuditEntries {
        uuid Id PK
        uuid TenantId FK
        uuid EntityId
        string EntityType
        string Action
        string Changes
        uuid UserId FK
        datetime CreatedAt
    }
    
    Notifications {
        uuid Id PK
        uuid TenantId FK
        uuid UserId FK
        uuid TaskId FK
        string Type
        string Message
        bool IsRead
        datetime CreatedAt
    }
```

---

## 5. Multi-Tenancy Architecture

```mermaid
graph TB
    subgraph "Request Flow"
        Request[HTTP Request<br/>with JWT Token]
        Middleware[Tenant Validation<br/>Middleware]
        TenantContext[ITenantContext<br/>CurrentTenantId<br/>CurrentUserId]
    end
    
    subgraph "Data Access Layer"
        QueryFilter[Global Query Filter<br/>WHERE TenantId = @CurrentTenant]
        Repository[Repository<br/>GetTasks GetUsers]
    end
    
    subgraph "Database"
        DB[(Multi-Tenant Database<br/>Shared Schema)]
    end
    
    Request -->|JWT Claims| Middleware
    Middleware -->|Extract Tenant| TenantContext
    TenantContext -->|Inject| Repository
    Repository -->|Apply Filter| QueryFilter
    QueryFilter -->|SQL Query| DB
    
    subgraph "Data Isolation"
        Tenant1[Tenant 1 Data]
        Tenant2[Tenant 2 Data]
        Tenant3[Tenant 3 Data]
    end
    
    DB -.->|Isolated| Tenant1
    DB -.->|Isolated| Tenant2
    DB -.->|Isolated| Tenant3
    
    style Request fill:#61dafb
    style Middleware fill:#ff6b6b
    style TenantContext fill:#51cf66
    style QueryFilter fill:#ffd43b
    style DB fill:#336791
```

### Multi-Tenancy Features:
- **Shared Database**: All tenants in single database with TenantId column
- **Query Filtering**: EF Core global filters ensure tenant isolation
- **JWT-Based Context**: Current tenant extracted from JWT claims
- **Role-Based Access**: Different roles per tenant (Member/Manager/Admin)
- **Tenant Switching**: Users can switch between tenants they belong to

---

## 6. Authentication Flow

```mermaid
sequenceDiagram
    participant User
    participant React as React Web App
    participant ALB as Load Balancer
    participant API as API Service
    participant Auth as Auth Service
    participant DB as PostgreSQL
    participant JWT as JWT Service
    
    User->>React: Enter Credentials
    React->>ALB: POST /api/auth/login
    ALB->>API: Forward Request
    API->>Auth: Login(email, password)
    Auth->>DB: GetUserByEmail(email)
    DB-->>Auth: User Entity
    Auth->>Auth: VerifyPassword(password, hash)
    Auth->>DB: GetUserTenants(userId)
    DB-->>Auth: List<Tenant>
    Auth->>JWT: GenerateToken(user, tenants)
    JWT-->>Auth: JWT Token
    Auth-->>API: LoginResponse
    API-->>ALB: 200 OK + Token
    ALB-->>React: Response
    React->>React: Store Token
    React->>React: Store Current Tenant
    React-->>User: Redirect to Dashboard
    
    Note over User,JWT: Token contains: userId, email, tenantIds, roles per tenant
    
    User->>React: Access Protected Route
    React->>ALB: GET /api/tasks<br/>Authorization: Bearer <token>
    ALB->>API: Forward with Token
    API->>API: JWT Middleware<br/>Validate Token
    API->>API: Extract TenantId<br/>from Claims
    API->>API: Tenant Validation<br/>Check Access
    API->>DB: GetTasks(tenantId)
    DB-->>API: Filtered Tasks
    API-->>ALB: 200 OK + Data
    ALB-->>React: Response
    React-->>User: Display Tasks
```

---

## 7. Task Management Flow

```mermaid
sequenceDiagram
    participant User
    participant UI as React UI
    participant API as API Controller
    participant Handler as CQRS Handler
    participant Validator as FluentValidator
    participant Domain as Task Entity
    participant Repo as Repository
    participant DB as Database
    participant Worker as Background Worker
    
    User->>UI: Create New Task
    UI->>API: POST /api/tenants/{id}/tasks
    API->>Handler: CreateTaskCommand
    Handler->>Validator: Validate(command)
    Validator-->>Handler: ValidationResult
    
    alt Validation Failed
        Handler-->>API: ValidationError
        API-->>UI: 400 Bad Request
        UI-->>User: Show Error
    else Validation Passed
        Handler->>Domain: new Task(title, desc, etc)
        Domain->>Domain: Apply Business Rules
        Handler->>Repo: AddAsync(task)
        Repo->>DB: INSERT INTO Tasks
        DB-->>Repo: Success
        Repo-->>Handler: Task Created
        Handler->>DB: Create AuditEntry
        Handler-->>API: TaskDto
        API-->>UI: 201 Created
        UI-->>User: Show Success
        
        Note over Worker: Background Process (every 15 min)
        Worker->>DB: Query Due Tasks
        DB-->>Worker: Tasks with DueDate
        Worker->>Worker: Check Business Hours
        Worker->>DB: Create ReminderSent
        Worker->>Worker: Send Notification
        Worker->>DB: Create Notification
    end
    
    User->>UI: Update Task Status
    UI->>API: PUT /api/tenants/{id}/tasks/{taskId}
    API->>Handler: UpdateTaskCommand
    Handler->>Repo: GetByIdAsync(taskId)
    Repo->>DB: SELECT * FROM Tasks<br/>WHERE Id = @taskId AND TenantId = @tenantId
    DB-->>Repo: Task Entity
    Repo-->>Handler: Task
    Handler->>Domain: CanTransitionTo(newStatus)
    
    alt Invalid Transition
        Domain-->>Handler: BusinessRuleException
        Handler-->>API: 400 Bad Request
        API-->>UI: Error
    else Valid Transition
        Domain->>Domain: UpdateStatus(newStatus)
        Handler->>Repo: UpdateAsync(task)
        Repo->>DB: UPDATE Tasks SET Status = @status
        DB-->>Repo: Success
        Repo-->>Handler: Updated Task
        Handler->>DB: Create AuditEntry
        Handler-->>API: TaskDto
        API-->>UI: 200 OK
        UI-->>User: Show Updated Task
    end
```

---

## 8. Class Diagram - Domain Entities

```mermaid
classDiagram
    class User {
        +Guid Id
        +string Email
        +string FirstName
        +string LastName
        +string PasswordHash
        +bool IsActive
        +DateTime? LastLoginAt
        +string? RefreshToken
        +List~UserTenant~ UserTenants
        +List~TaskItem~ AssignedTasks
        +List~TaskItem~ CreatedTasks
        +string FullName
        +bool HasAccessToTenant(Guid tenantId)
        +TenantRole? GetRoleInTenant(Guid tenantId)
    }
    
    class Tenant {
        +Guid Id
        +string Name
        +string TimeZone
        +bool IsActive
        +TenantSettings Settings
        +List~UserTenant~ UserTenants
        +List~TaskItem~ Tasks
        +bool IsBusinessHours(DateTime time)
    }
    
    class UserTenant {
        +Guid UserId
        +Guid TenantId
        +TenantRole Role
        +bool IsActive
        +DateTime JoinedAt
        +DateTime? LeftAt
        +User User
        +Tenant Tenant
    }
    
    class TaskItem {
        +Guid Id
        +Guid TenantId
        +string Title
        +string Description
        +TaskStatus Status
        +Priority Priority
        +Guid? AssignedToId
        +Guid CreatedBy
        +Guid? ParentTaskId
        +DateTime? DueDate
        +int? EstimatedEffort
        +int? ActualEffort
        +List~TaskItem~ SubTasks
        +List~Attachment~ Attachments
        +List~EffortEntry~ EffortEntries
        +User AssignedTo
        +User CreatedByUser
        +TaskItem ParentTask
        +bool CanTransitionTo(TaskStatus newStatus)
        +bool CanComplete()
        +bool IsOverdue()
    }
    
    class Attachment {
        +Guid Id
        +Guid TaskId
        +Guid TenantId
        +string FileName
        +string StorageKey
        +string ContentType
        +long FileSize
        +Guid CreatedBy
        +TaskItem Task
        +User CreatedByUser
    }
    
    class EffortEntry {
        +Guid Id
        +Guid TaskId
        +Guid TenantId
        +Guid UserId
        +int Hours
        +DateTime Date
        +string Description
        +TaskItem Task
        +User User
    }
    
    class AuditEntry {
        +Guid Id
        +Guid TenantId
        +Guid EntityId
        +string EntityType
        +string Action
        +string Changes
        +Guid? UserId
        +DateTime CreatedAt
    }
    
    class Notification {
        +Guid Id
        +Guid TenantId
        +Guid UserId
        +Guid? TaskId
        +NotificationType Type
        +string Message
        +bool IsRead
        +DateTime CreatedAt
        +User User
        +TaskItem Task
    }
    
    class TenantRole {
        <<enumeration>>
        Member
        Manager
        Admin
    }
    
    class TaskStatus {
        <<enumeration>>
        New
        InProgress
        Blocked
        Done
        Archived
    }
    
    class Priority {
        <<enumeration>>
        Low
        Medium
        High
        Critical
    }
    
    User "1" --> "*" UserTenant : has
    Tenant "1" --> "*" UserTenant : has
    User "1" --> "*" TaskItem : assigned
    User "1" --> "*" TaskItem : created
    Tenant "1" --> "*" TaskItem : owns
    TaskItem "1" --> "*" TaskItem : parent/children
    TaskItem "1" --> "*" Attachment : has
    TaskItem "1" --> "*" EffortEntry : has
    User "1" --> "*" Notification : receives
    TaskItem "1" --> "*" Notification : about
    UserTenant --> TenantRole
    TaskItem --> TaskStatus
    TaskItem --> Priority
```

---

## 9. Component Diagram

```mermaid
graph TB
    subgraph "Frontend - React SPA"
        Components[React Components<br/>Pages & UI]
        Services[Frontend Services<br/>AuthService<br/>TaskService<br/>ProfileService]
        Context[React Context<br/>AuthContext<br/>State Management]
    end
    
    subgraph "Backend API - .NET 8"
        subgraph "API Layer"
            Controllers[Controllers]
            Middleware[Middleware<br/>Pipeline]
        end
        
        subgraph "Application Layer"
            Commands[Commands]
            Queries[Queries]
            Handlers[MediatR Handlers]
            Validators[FluentValidation]
        end
        
        subgraph "Domain Layer"
            Entities[Entities]
            ValueObjects[Value Objects]
            BusinessRules[Business Rules]
        end
        
        subgraph "Infrastructure Layer"
            Repos[Repositories]
            EFCore[EF Core Context]
            ExternalServices[External Services<br/>JWT, Email, Storage]
        end
    end
    
    subgraph "Background Services"
        Worker[Worker Service]
        ReminderService[Reminder Service]
        NotificationService[Notification Service]
    end
    
    subgraph "Data & Storage"
        PostgreSQL[(PostgreSQL)]
        Redis[Redis Cache]
        S3[S3 Storage]
    end
    
    Components --> Services
    Services --> Context
    Services -->|REST API| Controllers
    Controllers --> Middleware
    Middleware --> Handlers
    Handlers --> Commands
    Handlers --> Queries
    Handlers --> Validators
    Handlers --> Entities
    Handlers --> BusinessRules
    Handlers --> Repos
    Repos --> EFCore
    Handlers --> ExternalServices
    EFCore --> PostgreSQL
    ExternalServices --> Redis
    ExternalServices --> S3
    Worker --> ReminderService
    Worker --> NotificationService
    ReminderService --> PostgreSQL
    NotificationService --> PostgreSQL
    
    style Components fill:#61dafb
    style Controllers fill:#512bd4
    style Handlers fill:#68217a
    style Entities fill:#239120
    style PostgreSQL fill:#336791
    style Redis fill:#dc382d
```

---

## 10. Background Services Architecture

```mermaid
graph TB
    subgraph "Worker Service Process"
        Host[.NET Hosted Service]
        Timer[Background Timer<br/>Every 15 minutes]
    end
    
    subgraph "Reminder Service"
        CheckTasks[Check Due Tasks<br/>Next 24 Hours]
        FilterSent[Filter Already Sent<br/>Based on ReminderWindow]
        CheckHours[Check Business Hours<br/>Tenant TimeZone]
        SendReminder[Create Reminder Entry]
        CreateNotif[Create Notification]
    end
    
    subgraph "Database Operations"
        QueryTasks[(Query Tasks<br/>WHERE DueDate < NOW + 24h)]
        QueryReminders[(Query RemindersSent<br/>Check Duplicates)]
        InsertReminder[(Insert ReminderSent)]
        InsertNotif[(Insert Notification)]
    end
    
    subgraph "Error Handling"
        Retry[Retry Logic<br/>Exponential Backoff]
        Log[Structured Logging<br/>Serilog]
    end
    
    Host --> Timer
    Timer --> CheckTasks
    CheckTasks --> QueryTasks
    QueryTasks --> FilterSent
    FilterSent --> QueryReminders
    QueryReminders --> CheckHours
    CheckHours -->|Business Hours| SendReminder
    CheckHours -->|Outside Hours| Timer
    SendReminder --> InsertReminder
    SendReminder --> CreateNotif
    CreateNotif --> InsertNotif
    SendReminder -->|Error| Retry
    Retry -->|Log| Log
    Log --> Timer
    
    style Host fill:#512bd4
    style Timer fill:#ffd43b
    style CheckTasks fill:#51cf66
    style QueryTasks fill:#336791
    style Log fill:#ff6b6b
```

### Background Service Features:
- **Scheduled Execution**: Runs every 15 minutes
- **Idempotency**: ReminderWindow prevents duplicate reminders
- **Business Hours**: Respects tenant timezone settings
- **Error Handling**: Exponential backoff with structured logging
- **Graceful Shutdown**: Proper cancellation token handling

---

## Key Architecture Principles

### 1. **Clean Architecture**
- Clear separation of concerns across layers
- Domain layer has no external dependencies
- Infrastructure depends on abstractions

### 2. **Multi-Tenancy**
- Shared database with tenant isolation
- Query filters ensure data separation
- JWT-based tenant context

### 3. **CQRS Pattern**
- Commands for writes (Create/Update/Delete)
- Queries for reads (Get/List)
- MediatR for handler orchestration

### 4. **Repository Pattern**
- Abstraction over data access
- Testability through interfaces
- Specification pattern for complex queries

### 5. **Security**
- JWT authentication with tenant claims
- Role-based authorization per tenant
- Input validation with FluentValidation
- XSS/SQL injection protection

### 6. **Scalability**
- Stateless API services (horizontal scaling)
- Redis caching for performance
- ECS Fargate auto-scaling
- Database read replicas (future)

### 7. **Observability**
- Structured logging (Serilog)
- CloudWatch metrics
- Correlation IDs for request tracing
- Health checks

---

## Technology Stack Summary

| Layer | Technology |
|-------|------------|
| **Frontend** | React 18, TypeScript 5.9, Material-UI 5 |
| **API** | ASP.NET Core 8, C# 12 |
| **Database** | PostgreSQL 14 |
| **ORM** | Entity Framework Core 8 |
| **Caching** | Redis (ElastiCache) |
| **Storage** | AWS S3 |
| **Containerization** | Docker, Multi-stage builds |
| **Orchestration** | AWS ECS Fargate |
| **Load Balancing** | AWS Application Load Balancer |
| **CI/CD** | Docker, AWS ECR, ECS |
| **Monitoring** | AWS CloudWatch |
| **Patterns** | CQRS, Repository, Clean Architecture |

---

## Deployment Configuration

### Production Environment
- **Region**: ap-south-1 (Mumbai)
- **Load Balancer**: tasktracker-alb-1161871022
- **ECS Cluster**: tasktracker-cluster
- **Database**: tasktracker-db (Multi-AZ)
- **ECR Repository**: 141820512164.dkr.ecr.ap-south-1.amazonaws.com

### Scaling Configuration
- **API Service**: Auto-scale 1-4 instances
- **Web Service**: Auto-scale 1-4 instances
- **Database**: db.t3.medium with Multi-AZ
- **Target CPU**: 70% utilization

---

*Generated: December 17, 2025*
*Version: 1.0*
