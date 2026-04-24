# SpaceVox AI Backend Development Instructions

**CRITICAL:** When working with Java/Spring Boot backend code in this workspace, ALWAYS follow these patterns. This document is the source of truth for creating new services or extending existing ones.

---

## ⛔ DEPRECATED SERVICES - DO NOT MODIFY

> **The following services are DEPRECATED and should NOT be modified:**
>
> | Deprecated Service | Replacement | Notes |
> |-------------------|-------------|-------|
> | `router/` | `workflow-gateway/` | Routing merged into unified gateway |
> | `dataplane-gateway/` | `workflow-gateway/` | gRPC gateway merged into unified gateway |
>
> **When you see requests related to these deprecated services:**
> 1. Direct all changes to `workflow-gateway/` instead
> 2. Refer to `DEPRECATED.md` in each folder for migration details
> 3. Update callers to use the new gateway endpoints

---

## 🔴 QUICK START - SERVICE IMPLEMENTATION GUIDE

> **AI Agents & Developers:** Before creating a new service or enhancing an existing one, **ALWAYS consult** the quick reference guide:
>
> 📖 **[SERVICE-IMPLEMENTATION-GUIDE.md](../internal-docs/SERVICE-IMPLEMENTATION-GUIDE.md)**
>
> This guide provides:
> - Quick decision tree for what you're building
> - Copy-paste templates for all common patterns
> - Common mistakes to avoid
> - Checklist validation before committing
>
> **Use this guide as your first reference**, then return here for detailed explanations if needed.

---

## 🚫 TOP 10 MISTAKES (Check Before Asking Why Something Doesn't Work)

| # | Mistake | Symptom | Fix |
|---|---------|---------|-----|
| 1 | JPA annotations on fields | `Hibernate can't find property` | Move annotations to GETTERS |
| 2 | Missing `@EnableRLS` on entity | Query returns all rows or none | Add `@EnableRLS` to entity class |
| 3 | Using `javax.persistence` | Compile error | Use `jakarta.persistence` |
| 4 | Using `_api_admin` at runtime | RLS policies don't apply | Use `_api_least` user in local.env |
| 5 | Business logic in Controller | Logic not reusable from events/jobs | Move ALL logic to Service class |
| 6 | Missing `@Transient` on helper methods | `Unknown column 'xxx'` errors | Add `@Transient` to `isXxx()` / `getXxx()` helpers |
| 7 | MapStruct not ignoring audit fields | `NullPointerException` on create | Add `@Mapping(target = "id", ignore = true)` etc. |
| 8 | Spring annotations in workflow-engine | `NoSuchBeanDefinitionException` | Use Quarkus `@Inject`, `@ApplicationScoped` |
| 9 | Missing tenant/env headers | Empty results or 403 | Include `X-SELECTED-TENANT` + `X-SELECTED-ENV` |
| 10 | Event doesn't extend BaseChannelEvent | Event not published/received | Extend `BaseChannelEvent`, include tenantId/envId |

---

## 🏗️ FOUNDATIONAL ARCHITECTURE PATTERNS (CRITICAL)

### Framework Detection: Spring Boot vs Quarkus

| Service | Framework | DI Annotations | Notes |
|---------|-----------|----------------|-------|
| **workflow-engine** | **Quarkus** | `@ApplicationScoped`, `@Inject` | Only Quarkus service |
| All other services | **Spring Boot 3.x** | `@Service`, `@Autowired`, `@Component` | Extend `WorkflowApp` |

**When modifying code, ALWAYS check which framework the service uses:**

```java
// ❌ WRONG - Using Spring in workflow-engine
@Service
@Autowired

// ✅ CORRECT - workflow-engine uses Quarkus
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class MyHandler {
    @Inject
    SomeService service;
}
```

```java
// ✅ CORRECT - All other services use Spring Boot
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;

@Service
@RequiredArgsConstructor
public class MyService {
    private final SomeRepository repo;
}
```

---

### Multi-Tenancy Architecture (CRITICAL)

**Every table MUST have these columns for RLS (Row-Level Security):**

| Column | Type | Purpose | Required |
|--------|------|---------|----------|
| `tenant_id` | UUID | Tenant isolation | YES (except cross-tenant tables) |
| `env_id` | UUID | Environment isolation within tenant | YES (nullable for tenant-level data) |
| `deleted` | BOOLEAN | Soft delete support | YES (default: false) |

**RLS Context Variables (set by base-http interceptor):**
- `app.tenant_id` - Current tenant UUID (from JWT or X-SELECTED-TENANT header)
- `app.env_id` - Current environment UUID (from X-SELECTED-ENV header)

**Super-Admin Bypass UUID:** `a8737fb0-0d10-4aab-809b-d54ef71dc8f1`
- This UUID in policies allows super-admin access to all tenants/environments

**RLS Policy Structure:**
```sql
-- SELECT/UPDATE/DELETE policy
CREATE POLICY {table}_tenant_isolation ON wf_{schema}.{table}
    USING ('a8737fb0-0d10-4aab-809b-d54ef71dc8f1' = ANY(current_setting('app.tenant_id')::uuid[])
        OR (tenant_id = ANY(current_setting('app.tenant_id')::uuid[])
            AND (env_id IS NULL OR env_id = ANY(current_setting('app.env_id')::uuid[]))));

-- INSERT policy (requires WITH CHECK)
CREATE POLICY {table}_tenant_isolation_insert ON wf_{schema}.{table}
    FOR INSERT
    WITH CHECK ('a8737fb0-0d10-4aab-809b-d54ef71dc8f1' = ANY(current_setting('app.tenant_id')::uuid[])
        OR (tenant_id = ANY(current_setting('app.tenant_id')::uuid[])
            AND (env_id IS NULL OR env_id = ANY(current_setting('app.env_id')::uuid[]))));
```

---

### Base Module Dependencies

| Module | Package | Purpose | When to Use |
|--------|---------|---------|-------------|
| **base-multi-tenant** | `com.svai.basemt.*` | `BaseEntity`, audit fields, tenant context | ALL entities extend this |
| **base-http** | `com.svai.entrycp.*` | `WorkflowApp`, JWT handling, context propagation | ALL Spring Boot services |
| **base-rls** | `com.svai.baserls.*` | `@EnableRLS` annotation, RLS generation | ALL multi-tenant entities |
| **base-api-contracts** | `com.svai.workflow.contract.*` | Shared DTOs (`BaseDto`), API interfaces | Inter-service communication |
| **base-common** | `com.svai.basecommon.*` | `SearchRequest`, utilities, compensation | Pagination, cross-service ops |
| **base-data-exchange** | `com.svai.basedataex.*` | `MessagePublisher`, `IChannelEventHandler`, `@Topic` | Event publishing/consuming |

---

### Entity Hierarchy (CRITICAL - Follow Exactly)

```
AggregateBase<A>                           (base-multi-tenant: domain event registration)
  └── BaseEntity                           (base-multi-tenant: id, version, audit fields)
       └── Base                            (YOUR SERVICE's model package)
            └── YourEntity                 (YOUR domain fields with @EnableRLS)
```

**BaseEntity Fields (inherited automatically):**
```java
// From base-multi-tenant BaseEntity
UUID id;                    // @Id @UuidGenerator - auto-generated UUIDv7
Long version;               // @Version - optimistic locking
OffsetDateTime createdAt;   // @CreatedDate - auto-set on insert
OffsetDateTime updatedAt;   // @LastModifiedDate - auto-set on update
String createdBy;           // @CreatedBy - auto-set from SecurityContext
String updatedBy;           // @LastModifiedBy - auto-set from SecurityContext
```

**Your Service's Base.java (REQUIRED):**
```java
package com.svai.workflow.{yourservice}.model;  // or .entity

import com.svai.basemt.tenant.BaseEntity;
import jakarta.persistence.EntityListeners;
import jakarta.persistence.MappedSuperclass;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Base extends BaseEntity {
    private static final long serialVersionUID = 1L;
}
```

---

### 🚨 JPA ANNOTATIONS: GETTERS ONLY (CRITICAL)

**SpaceVox Convention: ALL JPA/Hibernate annotations go on GETTER methods, NEVER on fields.**

```java
@Entity
@Table(name = "my_entity")
@EnableRLS  // ALWAYS add for multi-tenant tables
public class MyEntity extends Base {

    // ✅ CORRECT: Fields are PLAIN - no annotations
    private String name;
    private UUID accountId;
    private MyStatus status;
    private OffsetDateTime activatedAt;

    // ✅ CORRECT: Annotations on GETTERS
    @Column(name = "name", nullable = false, length = 255)
    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Column(name = "account_id", nullable = false)
    public UUID getAccountId() {
        return accountId;
    }

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false, length = 50)
    public MyStatus getStatus() {
        return status;
    }

    @Column(name = "activated_at")
    public OffsetDateTime getActivatedAt() {
        return activatedAt;
    }
    
    // ❌ WRONG - Never do this:
    // @Column(name = "name")
    // private String name;
}
```

**Why Getters?**
- Consistent with Hibernate property-access strategy
- Allows proxy interception for lazy loading
- Platform convention across all services

---

### DTO Pattern (BaseDto from base-api-contracts)

**All DTOs in base-api-contracts MUST extend BaseDto:**

```java
package com.svai.workflow.contract.{service}.dto;

import com.svai.workflow.contract.common.dto.BaseDto;
import lombok.Data;
import lombok.EqualsAndHashCode;
import lombok.NoArgsConstructor;
import lombok.experimental.SuperBuilder;

@Data
@SuperBuilder             // Required for inheritance
@NoArgsConstructor        // Required for Jackson
@EqualsAndHashCode(callSuper = true)
public class MyEntityDto extends BaseDto {
    // Domain fields only - id, tenantId, envId, audit fields inherited
    private String name;
    private String description;
}
```

**BaseDto Fields (inherited):**
```java
UUID id;
UUID tenantId;
UUID envId;
OffsetDateTime createdAt;
OffsetDateTime updatedAt;
String createdBy;
String updatedBy;
String contractVersion;  // For API versioning
```

---

### Service Pattern (ALL Logic Here)

**Controllers, Message Handlers, and Scheduled Jobs MUST delegate to services:**

```java
@Service
@Transactional
@Slf4j
@RequiredArgsConstructor
public class MyEntityService {
    
    private final MyEntityRepository repository;
    private final MyEntityMapper mapper;
    
    @Transactional(readOnly = true)
    public MyEntityDto findById(UUID id) {
        return repository.findById(id)
            .map(mapper::toDto)
            .orElseThrow(() -> new EntityNotFoundException("MyEntity not found: " + id));
    }
    
    public MyEntityDto create(MyEntityDto dto) {
        MyEntity entity = mapper.toEntity(dto);
        entity = repository.save(entity);
        log.info("Created MyEntity with id: {}", entity.getId());
        return mapper.toDto(entity);
    }
    
    @Transactional(readOnly = true)
    public Page<MyEntityDto> search(SearchRequest request) {
        Pageable pageable = request.toPageable();  // From base-common
        return repository.findAll(pageable).map(mapper::toDto);
    }
}
```

---

### MapStruct Mapper Pattern (NOT ModelMapper)

```java
@Mapper(componentModel = "spring", 
        nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface MyEntityMapper {
    
    MyEntityDto toDto(MyEntity entity);
    
    // ALWAYS ignore managed fields when converting DTO → Entity
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "version", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "updatedBy", ignore = true)
    @Mapping(target = "tenantId", ignore = true)
    @Mapping(target = "envId", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    MyEntity toEntity(MyEntityDto dto);
    
    // For updates - only map domain fields
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "version", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "updatedBy", ignore = true)
    @Mapping(target = "tenantId", ignore = true)
    @Mapping(target = "envId", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    void updateEntity(MyEntityDto dto, @MappingTarget MyEntity entity);
}
```

---

### Transient Helper Methods (Hibernate Gotcha)

**If you add `isXxx()` or `getXxx()` helper methods, mark them `@Transient`:**

```java
@Entity
public class WorkflowWaitRequest extends Base {
    
    private WaitStatus status;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "status")
    public WaitStatus getStatus() { return status; }
    
    // ✅ CORRECT - Transient prevents Hibernate from treating as property
    @Transient
    public boolean isPending() {
        return status == WaitStatus.PENDING;
    }
    
    @Transient
    public boolean isResolved() {
        return status == WaitStatus.RESOLVED;
    }
    
    // ❌ WRONG - Hibernate will look for 'pending' column
    // public boolean isPending() { ... }
}
```

---

### Import Guidelines

```java
// JPA - use jakarta.persistence (NOT javax.persistence)
import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import jakarta.persistence.Transient;
import jakarta.persistence.Enumerated;
import jakarta.persistence.EnumType;

// For workflow-engine (Quarkus) - use jakarta.enterprise
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

// For Spring Boot services
import org.springframework.stereotype.Service;
import org.springframework.beans.factory.annotation.Autowired;
import lombok.RequiredArgsConstructor;

// RLS annotation
import com.svai.baserls.annotation.EnableRLS;

// Base classes
import com.svai.basemt.tenant.BaseEntity;
import com.svai.workflow.contract.common.dto.BaseDto;
import com.svai.basecommon.search.SearchRequest;
```

---

## � WHERE TO FIND THINGS (Package Location Index)

| Looking for... | Package/Class |
|----------------|---------------|
| Tenant context | `com.svai.basemt.context.TenantContext` |
| Request headers constants | `com.svai.entrycp.http.util.Constant` |
| Search/pagination | `com.svai.basecommon.search.SearchRequest` |
| Search spec builder | `com.svai.basecommon.search.SearchSpecificationBuilder` |
| Event publishing | `com.svai.basedataex.exchange.channel.MessagePublisher` |
| Event handling | `com.svai.basedataex.exchange.channel.eventhandler.IChannelEventHandler` |
| Topic annotation | `com.svai.basedataex.exchange.annotation.Topic` |
| Notification sending | `com.svai.basedataex.exchange.notification.NotificationPublisher` |
| Internal JWT | `com.svai.entrycp.internal.InternalJwtService` |
| BaseEntity | `com.svai.basemt.tenant.BaseEntity` |
| BaseDto | `com.svai.workflow.contract.common.dto.BaseDto` |
| RLS annotation | `com.svai.baserls.annotation.EnableRLS` |
| Compensation | `com.svai.basecommon.compensation.CompensationService` |
| Connection reference | `com.svai.workflow.contract.connection.ConnectionReference` |
| AI observability events | `com.svai.workflow.contract.observability.AIObservabilityEvent` |
| AI event types | `com.svai.workflow.contract.observability.AIEventType` |
| Distributed locking | `com.svai.basecommon.concurrency.DistributedLockService` |
| Get-or-create pattern | `com.svai.basecommon.concurrency.ConcurrencyUtils` |
| Optimistic lock retry | `com.svai.basecommon.concurrency.ConcurrencyUtils` |

---

## �📁 DOCUMENTATION GUIDELINES

All markdown (.md) documentation files should be placed in `internal-docs/`:

| Type | Location | Examples |
|------|----------|----------|
| **Architecture Designs** | `internal-docs/designs/` | Control plane architecture, state management designs |
| **Coding Patterns** | `internal-docs/` | spacevox-coding-patterns.md |
| **API Documentation** | `internal-docs/docs/` | API references, integration guides |
| **Infrastructure** | `internal-docs/docs/50-infrastructure/` | Messaging architecture, deployment configs |
| **Runbooks** | `internal-docs/runbooks/` | Operational procedures, troubleshooting |

**DO NOT** create `.md` files in:
- Service root directories (except README.md)
- `.github/` folder (except copilot-instructions.md and issue/PR templates)
- Random locations in the workspace

---

## �🚨 MANDATORY CHECKLISTS

### Creating a NEW Service Checklist

When asked to create a new microservice, ALWAYS complete ALL items:

- [ ] **1. Database Setup** - Add to `infrastructure/local/db-init/01-create-users-and-schemas.sql`:
  - [ ] Create admin user: `{service}_api_admin` (for Flyway migrations)
  - [ ] Create API user: `{service}_api_least` (for runtime - **required for RLS**)
  - [ ] Create schema: `wf_{service}` owned by admin user
  - [ ] Add GRANT permissions section for `{service}_api_least`

- [ ] **2. Project Structure** - Create service with:
  - [ ] `pom.xml` with all base-* dependencies (see template below)
  - [ ] Main application class extending `WorkflowApp`
  - [ ] `application.yml` / `application-local.yml`
  - [ ] `local.env` with proper DB credentials
  - [ ] `Dockerfile` and `Makefile`
  - [ ] `logback-spring.xml` that includes base-http's logback-base.xml

- [ ] **3. Flyway Baseline Migration** - Create `V1__baseline_schema.sql`:
  - [ ] All tables with required columns (tenant_id, env_id, deleted, version, audit fields)
  - [ ] Enable RLS on each table
  - [ ] Create tenant_isolation policies (SELECT/UPDATE/DELETE and INSERT)
  - [ ] Create indexes on tenant_id

- [ ] **4. Entity Setup**:
  - [ ] Create `Base.java` extending `BaseEntity` from `base-multi-tenant`
  - [ ] All entities extend local `Base.java`
  - [ ] Add `@Entity`, `@Table`, `@EnableRLS` annotations
  - [ ] JPA annotations on **GETTERS**, not fields

- [ ] **5. Messaging (if needed)** - Add to LocalStack init scripts:
  - [ ] Create SNS topic in `infrastructure/local/localstack-init/init-aws-resources.sh`
  - [ ] Create SQS queue (with DLQ) if consuming events
  - [ ] Subscribe queue to topic if using fan-out pattern
  - [ ] Update `internal-docs/docs/50-infrastructure/messaging-architecture.md`

- [ ] **6. API Contracts** - Add to `base-api-contracts`:
  - [ ] DTOs in `contract/{service}/dto/`
  - [ ] API interface in `contract/{service}/api/`

### Extending an EXISTING Service Checklist

When adding new features to an existing service:

- [ ] **1. New Entity?** - If adding a new table:
  - [ ] Create Flyway migration `V{N}__{description}.sql`
  - [ ] Include RLS policies for new table
  - [ ] Create entity extending local `Base.java` with `@EnableRLS`
  - [ ] Create repository, mapper, service, controller

- [ ] **2. New DTO?** - Before creating:
  - [ ] Check `base-api-contracts/contract/common/dto/` for existing shared DTOs
  - [ ] Check other service's `dto/` packages
  - [ ] Shared DTOs → `common/dto/`, service-specific → `{service}/dto/`

- [ ] **3. New Event/Messaging?**:
  - [ ] Add topic/queue to LocalStack init scripts
  - [ ] Update messaging-architecture.md
  - [ ] Add publisher/listener configuration

---

## Database Schema Setup (CRITICAL)

### User & Schema Naming Convention

| Component | Pattern | Example |
|-----------|---------|---------|
| Schema | `wf_{service}` | `wf_notification`, `wf_catalog` |
| Admin User | `{service}_api_admin` | `notification_api_admin` |
| API User | `{service}_api_least` | `notification_api_least` |

### SQL Template for New Service

Add this to `infrastructure/local/db-init/01-create-users-and-schemas.sql`:

```sql
-- ========================================
-- SERVICE {N}: {SERVICE_NAME} (wf_{service})
-- ========================================
CREATE USER {service}_api_admin WITH PASSWORD 'SpaceVox@123';
GRANT {service}_api_admin TO postgres;
CREATE SCHEMA IF NOT EXISTS wf_{service} AUTHORIZATION {service}_api_admin;
ALTER SCHEMA wf_{service} OWNER TO {service}_api_admin;
ALTER ROLE {service}_api_admin SET search_path TO wf_{service};

CREATE USER {service}_api_least WITH PASSWORD 'SpaceVox@123';
ALTER ROLE {service}_api_least SET search_path TO wf_{service};

-- Add to GRANT PERMISSIONS section:
-- Run as {service}_api_admin:
\c workflow {service}_api_admin
GRANT USAGE ON SCHEMA wf_{service} TO {service}_api_least;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA wf_{service} TO {service}_api_least;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA wf_{service} TO {service}_api_least;
ALTER DEFAULT PRIVILEGES IN SCHEMA wf_{service} GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO {service}_api_least;
ALTER DEFAULT PRIVILEGES IN SCHEMA wf_{service} GRANT USAGE, SELECT ON SEQUENCES TO {service}_api_least;
```

### Flyway Migration Template (V1__baseline_schema.sql)

```sql
-- Table with all required columns
CREATE TABLE IF NOT EXISTS wf_{schema}.{table_name} (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    env_id UUID,
    
    -- Domain-specific columns here
    name VARCHAR(255) NOT NULL,
    
    -- Required audit/soft-delete columns
    deleted BOOLEAN DEFAULT FALSE,
    version INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by VARCHAR(255),
    updated_by VARCHAR(255)
);

-- Index for RLS performance
CREATE INDEX IF NOT EXISTS idx_{table_name}_tenant ON wf_{schema}.{table_name}(tenant_id);

-- Enable RLS
ALTER TABLE wf_{schema}.{table_name} ENABLE ROW LEVEL SECURITY;

-- Tenant isolation policy (SELECT/UPDATE/DELETE)
CREATE POLICY {table_name}_tenant_isolation ON wf_{schema}.{table_name}
    USING ('a8737fb0-0d10-4aab-809b-d54ef71dc8f1' = ANY(current_setting('app.tenant_id')::uuid[])
        OR (tenant_id = ANY(current_setting('app.tenant_id')::uuid[])
            AND (env_id IS NULL OR env_id = ANY(current_setting('app.env_id')::uuid[]))));

-- INSERT policy (requires WITH CHECK)
CREATE POLICY {table_name}_tenant_isolation_insert ON wf_{schema}.{table_name}
    FOR INSERT
    WITH CHECK ('a8737fb0-0d10-4aab-809b-d54ef71dc8f1' = ANY(current_setting('app.tenant_id')::uuid[])
        OR (tenant_id = ANY(current_setting('app.tenant_id')::uuid[])
            AND (env_id IS NULL OR env_id = ANY(current_setting('app.env_id')::uuid[]))));
```

> **Note:** UUID `a8737fb0-0d10-4aab-809b-d54ef71dc8f1` is the super-admin bypass tenant ID.

---

## Project Structure

### Required Files for New Service

```
{service}/
├── pom.xml                              # Maven config with base-* deps
├── Dockerfile                           # Docker image build
├── Makefile                             # Build/run commands
├── local.env                            # Local environment variables
├── src/main/
│   ├── java/com/svai/workflow/{service}/
│   │   ├── WF{Service}Application.java  # Main class extends WorkflowApp
│   │   ├── controller/                  # REST controllers
│   │   ├── dto/                         # Data Transfer Objects
│   │   ├── entity/
│   │   │   ├── Base.java                # Local base extending BaseEntity
│   │   │   └── {Entity}.java            # Domain entities
│   │   ├── exception/                   # Custom exceptions
│   │   ├── mapper/                      # MapStruct mappers
│   │   ├── repo/                        # JPA repositories
│   │   ├── service/                     # Business logic
│   │   └── event/                       # Domain events + handlers
│   └── resources/
│       ├── application.yml
│       ├── application-local.yml
│       ├── logback-spring.xml           # MUST include base-http's logback-base.xml
│       └── db/migration/
│           └── V1__baseline_schema.sql
```

### local.env Template

```dotenv
profile=local

# Database (CRITICAL: Use _api_least for RLS enforcement)
DB_URL=jdbc:postgresql://localhost:5432/workflow?currentSchema=wf_{service}
DB_SCHEMA=wf_{service}
DB_USERNAME={service}_api_least
DB_PASSWORD=SpaceVox@123
DB_MAX_POOL_SIZE=2
FLYWAY_USER={service}_api_admin
FLYWAY_PASSWORD=SpaceVox@123

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# JWT Providers
JWT_PROVIDER_CUSTOMER=https://external.localhost
JWT_PROVIDER_INTERNAL=https://internal.localhost

# CORS
CORS_ALLOWED_ORIGINS=http://localhost:3000,https://localhost:5173

# Internal Auth
INTERNAL_AUTH_ENABLED=false

# LocalStack (for local AWS emulation)
CLOUD_PROVIDER_AWS_ENABLED=true
CLOUD_PROVIDER_AWS_REGION=us-west-2
LOCAL_CLOUD_ENDPOINT=http://localhost:4566
```

### logback-spring.xml Template

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration scan="true" scanPeriod="30 seconds">
    <property name="SERVICE_NAME" value="{service}"/>
    <include resource="logback-base.xml"/>
</configuration>
```

### Dockerfile Template

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE {port}
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Makefile Template

```makefile
.PHONY: build run test clean docker docker-run

build:
	mvn clean package -DskipTests

run:
	mvn spring-boot:run -Dspring-boot.run.profiles=local

test:
	mvn test

clean:
	mvn clean

docker:
	docker build -t {service}-service .

docker-run:
	docker run -p {port}:{port} --env-file local.env {service}-service
```

---

## Entity/DTO/Controller Patterns

### Entity Hierarchy

```
AggregateBase<A> (domain event registration - base-multi-tenant)
  └── BaseEntity (id, version, audit fields - base-multi-tenant)
       └── Base (service-local, com.svai.workflow.{service}.entity.Base)
            └── YourEntity (domain fields only)
```

### Base.java Template (per service)

```java
package com.svai.workflow.{service}.entity;

import com.svai.basemt.tenant.BaseEntity;
import jakarta.persistence.EntityListeners;
import jakarta.persistence.MappedSuperclass;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class Base extends BaseEntity {
    private static final long serialVersionUID = 1L;
}
```

### Entity Template

```java
@Entity
@Table(name = "my_entity")
@EnableRLS  // CRITICAL: Required for RLS policy generation
public class MyEntity extends Base {
    
    private String name;
    private String description;
    
    // JPA annotations on GETTERS, not fields
    @Column(name = "name", nullable = false)
    public String getName() { return name; }
    
    public void setName(String name) { this.name = name; }
}
```

### DTO Pattern

```java
@Data
@NoArgsConstructor
@EqualsAndHashCode(callSuper = true)
public class MyEntityDto extends BaseDTO {
    // Domain fields only - id, tenantId, envId, audit fields inherited
    private String name;
    private String description;
}
```

### MapStruct Mapper Pattern

```java
@Mapper(componentModel = "spring", 
        nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface MyEntityMapper {
    
    MyEntityDto toDto(MyEntity entity);
    
    // ALWAYS ignore audit fields when converting to entity
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "version", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "updatedBy", ignore = true)
    @Mapping(target = "tenantId", ignore = true)
    @Mapping(target = "envId", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    MyEntity toEntity(MyEntityDto dto);
    
    @Mapping(target = "id", ignore = true)
    void updateEntity(MyEntityDto dto, @MappingTarget MyEntity entity);
}
```

### Service Pattern

```java
@Service
@Transactional
@Slf4j
@RequiredArgsConstructor
public class MyEntityService {
    
    private final MyEntityRepository repository;
    private final MyEntityMapper mapper;
    
    @Transactional(readOnly = true)
    public MyEntityDto findById(UUID id) {
        return repository.findById(id)
            .map(mapper::toDto)
            .orElseThrow(() -> new EntityNotFoundException("MyEntity not found"));
    }
    
    public MyEntityDto create(MyEntityDto dto) {
        MyEntity entity = mapper.toEntity(dto);
        entity = repository.save(entity);
        return mapper.toDto(entity);
    }
}
```

### Controller Pattern

**CRITICAL:** Controllers MUST NOT contain business logic. Always delegate to services.
This allows the same logic to be invoked via REST API, message handlers (SQS), scheduled jobs, etc.

```java
@RestController
@RequestMapping(Constant.WF_SERVICE_MOUNT_POINT + "/my-entities")
@Tag(name = "MyEntity", description = "MyEntity management APIs")
@RequiredArgsConstructor
@Slf4j
public class MyEntityController {
    
    private final MyEntityService service;  // ALL logic in service
    
    @GetMapping("/{id}")
    public ResponseEntity<MyEntityDto> findById(@PathVariable UUID id) {
        return ResponseEntity.ok(service.findById(id));
    }
    
    @PostMapping
    public ResponseEntity<MyEntityDto> create(@RequestBody @Valid MyEntityDto dto) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.create(dto));
    }
    
    // Pagination using SearchRequest from base-common
    @GetMapping
    public ResponseEntity<Page<MyEntityDto>> search(SearchRequest searchRequest) {
        return ResponseEntity.ok(service.search(searchRequest));
    }
}
```

### Pagination Pattern (CRITICAL)

**ALWAYS** use `com.svai.basecommon.search` for paginated APIs:

```java
import com.svai.basecommon.search.SearchRequest;
import com.svai.basecommon.search.SearchSpecificationBuilder;

@Service
@RequiredArgsConstructor
public class MyEntityService {
    
    private final MyEntityRepository repository;
    private final SearchSpecificationBuilder<MyEntity> searchBuilder;
    
    @Transactional(readOnly = true)
    public Page<MyEntityDto> search(SearchRequest request) {
        Pageable pageable = PageRequest.of(
            request.getPage() != null ? request.getPage() : 0,
            request.getSize() != null ? request.getSize() : 20,
            Sort.by(Sort.Direction.DESC, "createdAt")
        );
        
        Optional<Specification<MyEntity>> spec = searchBuilder.build(
            request.getSearch(), 
            request.getSort()
        );
        
        return spec.isPresent()
            ? repository.findAll(spec.get(), pageable).map(mapper::toDto)
            : repository.findAll(pageable).map(mapper::toDto);
    }
}
```

---

## Messaging Patterns (Cloud-Agnostic)

### Overview

| Pattern | AWS Service | Use Case |
|---------|-------------|----------|
| Pub/Sub (Broadcast) | SNS Topics | Events to multiple subscribers |
| Point-to-Point | SQS Queues | Direct service-to-service |
| Fan-out | SNS → SQS | Broadcast with reliable delivery |
| Session/Cache | Redis | Conversation state, short-lived data |

### When Adding New Messaging

1. **Update LocalStack init scripts** (`infrastructure/local/localstack-init/`):
   - `init-aws-resources.sh` (Bash for Docker/CI)
   - `init-aws-resources.ps1` (PowerShell for Windows dev)

2. **Update messaging-architecture.md** in internal-docs

3. **Add environment variables** to `local.env` and Helm values

### SNS/SQS Creation Template (LocalStack)

```bash
# Create SNS topic
aws --endpoint-url=http://localhost:4566 sns create-topic --name {service}-events

# Create SQS queue with DLQ
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name {consumer}-sub-for-{producer}-dlq
aws --endpoint-url=http://localhost:4566 sqs create-queue --queue-name {consumer}-sub-for-{producer} \
  --attributes '{"RedrivePolicy":"{\"deadLetterTargetArn\":\"arn:aws:sqs:us-west-2:000000000000:{consumer}-sub-for-{producer}-dlq\",\"maxReceiveCount\":\"3\"}"}'

# Subscribe queue to topic
aws --endpoint-url=http://localhost:4566 sns subscribe \
  --topic-arn arn:aws:sns:us-west-2:000000000000:{service}-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-west-2:000000000000:{consumer}-sub-for-{producer}
```

---

## Message Publishing (CRITICAL - Use base-data-exchange)

### Step 1: Create Event Publisher with @Topic annotation

The `@Topic` annotation specifies which SNS topic to publish to. The `MessagePublisher` handles cloud abstraction.

```java
import com.svai.basedataex.exchange.annotation.Topic;
import com.svai.basedataex.exchange.channel.MessagePublisher;
import com.svai.basedataex.exchange.channel.event.BaseChannelEvent;

@Topic("account-events")  // Maps to messaging.topics.account-events in yml
@Service
@ConditionalOnProperty(name = "messaging.topics.account-events")  // Only if topic configured
public class AccountEventPublisher {
    
    @Autowired
    private MessagePublisher publisher;  // Cloud-agnostic publisher
    
    /**
     * Generic publish method for any event.
     */
    public void publish(BaseChannelEvent event) {
        log.info("Publishing {} to account-events topic", event.getClass().getSimpleName());
        publisher.publish(event);  // Uses @Topic annotation to determine destination
    }
    
    /**
     * Typed convenience method.
     */
    public void publishAccountCreated(AccountCreatedEvent event) {
        log.info("Publishing AccountCreatedEvent for accountId: {}", event.getAccountId());
        publisher.publish(event);
    }
}
```

### Step 2: Bridge Domain Events to Message Publishing

Use `@TransactionalEventListener` to bridge Spring domain events to external message publishing:

```java
@Component
public class MyDomainEventHandler {
    
    @Autowired(required = false)  // Optional - graceful degradation if messaging disabled
    private AccountEventPublisher eventPublisher;
    
    @TransactionalEventListener  // Fires after transaction commits
    public void handleEnvironmentCreated(NewEnvironmentAddedDomainEvent domainEvent) {
        if (eventPublisher != null) {
            log.info("Publishing NewEnvironmentAddedDomainEvent for environment: {}", domainEvent.getUid());
            eventPublisher.publish(domainEvent);  // Publishes to SNS/SQS
        } else {
            log.debug("AccountEventPublisher not available, skipping event publish");
        }
    }
}
```

### Step 3: application.yml configuration

```yaml
messaging:
  topics:
    account-events: ${TOPIC_ACCOUNT_EVENTS:}  # ARN from env var
    catalog-events: ${TOPIC_CATALOG_EVENTS:}
```

---

## Message Consumption (CRITICAL - Use IChannelEventHandler)

### Step 1: Implement IChannelEventHandler interface

Event handlers implement `IChannelEventHandler<T>` with the specific event type:

```java
import com.svai.basedataex.exchange.channel.eventhandler.IChannelEventHandler;

@Component
public class NewEnvironmentAddedDomainEventHandler implements IChannelEventHandler<NewEnvironmentAddedDomainEvent> {
    
    @Autowired
    private ConnectorService connectorService;  // Delegate to service!
    
    @Override
    public void handle(NewEnvironmentAddedDomainEvent channelEvent) {
        log.info("Handling NewEnvironmentAddedDomainEvent for Env ID: {}", channelEvent.getEnvId());
        
        // IMPORTANT: Delegate ALL business logic to service
        // This allows same logic to be called from REST API or message handler
        connectorService.initializeEnvironmentDefaults(channelEvent.getTenantId(), channelEvent.getEnvId());
    }
}
```

### Key Rules for Message Handlers:

1. **Implement `IChannelEventHandler<T>`** - Parameterized with your event type
2. **Delegate to services** - No business logic in handlers (same as controllers)
3. **Events extend `BaseChannelEvent`** - Must have tenantId, envId, entityId

---

## Inter-Service Communication

### Two Types of Service-to-Service Calls

| Type | When to Use | Auth Mechanism | Headers Propagated |
|------|-------------|----------------|-------------------|
| **User Context Calls** | User-initiated requests that chain through services (e.g., Portal → Trigger → Run) | User's JWT token | Authorization, X-SELECTED-TENANT, X-SELECTED-ENV |
| **System/Internal Calls** | Background jobs, scheduled tasks, service-initiated operations | Internal JWT (auto-generated) | X-Internal-Service, tenant context |

### User Context Calls (Default)

For calls that originate from a user request, the context (JWT + tenant) is automatically propagated:

```
User → Portal (JWT + tenant headers)
       │
       └─→ Trigger (JWT + headers propagated automatically)
            │
            └─→ Run (JWT + headers propagated automatically)
```

The `ServiceCallRestTemplateConfig` provides a `@Primary` RestTemplate that automatically:
1. Propagates the `Authorization` header (user's JWT)
2. Propagates `X-SELECTED-TENANT` and `X-SELECTED-ENV` headers
3. Propagates `X-Request-ID` for distributed tracing

**No special configuration needed** - just inject `RestTemplate`:

```java
@Configuration
public class ApiConfig {
    
    @Value("${workflow.trigger.url}")
    private String triggerUrl;
    
    @Bean
    public TriggerApi triggerApi(RestTemplate restTemplate) {
        // RestTemplate automatically propagates Authorization + tenant headers
        return RestTemplateApiClient.create(TriggerApi.class, triggerUrl, restTemplate);
    }
}
```

### Internal/System Calls (Background Jobs)

For calls that don't have a user context (scheduled jobs, event handlers, message consumers), 
use the `internalRestTemplate` with internal JWT authentication:

```java
@Service
public class MyBackgroundService {
    
    @Autowired
    @Qualifier("internalRestTemplate")
    private RestTemplate internalRestTemplate;
    
    public void processEvent(UUID tenantId, UUID envId) {
        // Internal JWT is automatically generated with tenant context from ThreadLocal
        // or generates a system-level token if no tenant context
        ConnectorDto connector = internalRestTemplate.getForObject(
            catalogUrl + "/internal/api/workflow/catalog/connectors/" + id,
            ConnectorDto.class
        );
    }
}
```

**Internal JWT Features (from base-http):**
- Auto-configured via `InternalJwtService` (available in all services with base-http dependency)
- **Local Development:** Uses shared default secret - works across all services automatically
- **Production:** Set `INTERNAL_JWT_SECRET` environment variable (base64-encoded 256-bit key)
- 5-minute token validity with caching
- Includes tenant context (`tenant_id`, `env_id`) in JWT claims
- `InternalApiContextFilter` extracts context for `/internal/**` endpoints

**Internal endpoint pattern:**
```java
// In the TARGET service - expose internal endpoint
@RestController
@RequestMapping(Constant.WF_SERVICE_MOUNT_POINT + "/internal")
@InternalApi(service = "catalog")
public class CatalogInternalController {
    
    @GetMapping("/connectors/{id}")
    public ConnectorDto getConnector(@PathVariable UUID id) {
        // Context automatically set from internal JWT
        return service.getConnector(id);
    }
}
```

### ❌ NEVER use deprecated dependencies:
- `workflow-run-api`
- `workflow-portal-api`
- `workflow-catalog-api`
- `workflow-connection-api`
- `workflow-config-api`

### ✅ ALWAYS use `base-api-contracts`:

```xml
<dependency>
    <groupId>com.svai</groupId>
    <artifactId>base-api-contracts</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```

### Client Configuration Pattern

```java
@Configuration
public class ApiConfig {
    
    @Value("${workflow.catalog.api.url:http://localhost:8087}")
    private String catalogApiUrl;
    
    @Bean
    public CatalogApi catalogClient(RestTemplate restTemplate) {
        return RestTemplateApiClient.create(CatalogApi.class, catalogApiUrl, restTemplate);
    }

}
```

---

## Required pom.xml Dependencies

All services must include these base modules:

```xml
<dependencies>
    <!-- Spring Boot Starters -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
    <!-- SpaceVox Base Modules -->
    <dependency>
        <groupId>com.svai</groupId>
        <artifactId>base-http-entry-checkpoint</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>com.svai</groupId>
        <artifactId>base-rls</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>com.svai</groupId>
        <artifactId>base-multi-tenant</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>com.svai</groupId>
        <artifactId>base-api-contracts</artifactId>
        <version>0.0.1-SNAPSHOT</version>
    </dependency>
    
    <!-- MapStruct (NOT ModelMapper) -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>1.5.5.Final</version>
    </dependency>
    
    <!-- Flyway -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-database-postgresql</artifactId>
    </dependency>
</dependencies>
```

---

## Application Main Class Template

```java
@EnableJpaRepositories("com.svai.workflow.{service}")
@EnableScheduling
@EnableAsync
@SpringBootApplication(scanBasePackages = "com.svai")
@OpenAPIDefinition(info = @Info(title = "Workflow {Service} Service", version = "1.0"))
public class WF{Service}Application extends WorkflowApp {
    public static void main(String[] args) {
        SpringApplication.run(WF{Service}Application.class, args);
    }
}
```

---

## Sending Notifications (Common Utility)

### NotificationPublisher (from base-data-exchange)

All services can send notifications using `NotificationPublisher` from base-data-exchange.
This publishes to the "notification-requests" topic, consumed by the notification service.

**Key Principle:** Services specify WHAT they're notifying about (category), the notification
service decides HOW to deliver (channel) based on user preferences.

```java
import com.svai.basedataex.exchange.notification.*;

@Service
public class MyService {
    
    @Autowired(required = false)  // Optional - graceful degradation
    private NotificationPublisher notificationPublisher;
    
    // RECOMMENDED: Category-based notifications
    // The notification service determines channels based on user preferences
    public void notifyWorkflowComplete(UUID tenantId, UUID envId, UUID userId, String workflowName) {
        if (notificationPublisher != null) {
            notificationPublisher.notify(
                tenantId, envId, userId,
                NotificationCategory.WORKFLOW,  // Category, not channel
                "workflow-complete",            // Template code
                Map.of("workflowName", workflowName)
            );
        }
    }
    
    // High-priority notification (may bypass some user preferences)
    public void notifySecurityEvent(UUID tenantId, UUID envId, UUID userId, String event) {
        if (notificationPublisher != null) {
            notificationPublisher.notifyUrgent(
                tenantId, envId, userId,
                NotificationCategory.SECURITY,
                "security-alert",
                Map.of("event", event, "timestamp", Instant.now().toString())
            );
        }
    }
    
    // Direct channel (only when channel is REQUIRED, e.g., password reset must be email)
    public void sendPasswordResetEmail(UUID tenantId, UUID envId, UUID userId, String email, String resetLink) {
        if (notificationPublisher != null) {
            notificationPublisher.sendEmail(
                tenantId, envId, userId,
                email,
                "password-reset",
                Map.of("resetLink", resetLink)
            );
        }
    }
    
    // In-app alert only
    public void sendSystemAlert(UUID tenantId, UUID envId, UUID userId) {
        if (notificationPublisher != null) {
            notificationPublisher.sendAlert(
                tenantId, envId, userId,
                NotificationCategory.SYSTEM,
                NotificationSeverity.WARNING,
                "System Maintenance",
                "Scheduled maintenance at 2am UTC"
            );
        }
    }
}
```

### Configuration Required

Add to `application.yml`:
```yaml
messaging:
  topics:
    notification-requests: ${TOPIC_NOTIFICATION_REQUESTS:}
```

### Notification Categories

Services specify the notification category, which the notification service uses to:
1. Look up user preferences for that category
2. Deliver to enabled channels (email, SMS, push, in-app)
3. Fall back to system defaults if no preference set

| Category | When to Use |
|----------|-------------|
| `SECURITY` | Login alerts, password changes, suspicious activity |
| `WORKFLOW` | Workflow completions, failures, approvals |
| `BILLING` | Invoices, payment issues, subscription changes |
| `TEAM` | Team invitations, role changes, member updates |
| `SYSTEM` | Maintenance, downtime, feature announcements |
| `INTEGRATION` | Connection status, sync failures, API issues |
| `TRIGGER` | Trigger executions, schedule notifications |
| `STORAGE` | Upload completions, storage limits, file events |
| `GENERAL` | Catch-all for other notifications |

### Notification Priority

| Priority | Behavior |
|----------|----------|
| `LOW` | Batched, may be digest-only |
| `NORMAL` | Standard delivery per user preferences |
| `HIGH` | May bypass some preferences, ensures delivery |
| `URGENT` | All channels enabled, may include SMS |

### Available Methods

| Method | Description |
|--------|-------------|
| `notify(tenantId, envId, userId, category, template, vars)` | **Recommended** - category-based |
| `notifyHigh(...)` | High priority notification |
| `notifyUrgent(...)` | Urgent - all channels |
| `notifySimple(tenantId, envId, userId, category, subject, body)` | No template |
| `sendEmail(...)` | Direct email (only for required cases) |
| `sendEmailSimple(...)` | Direct email without template |
| `sendAlert(...)` | In-app alert only |
| `send(NotificationRequest)` | Full control with builder |
| `sendEmailWithAlert(...)` | Email + in-app alert |

---

## Distributed Transactions & Compensation (CRITICAL)

### When to Use Compensation

Use `com.svai.basecommon.compensation` when:
- Making changes across multiple services (cross-service transactions)
- Calling external APIs that may fail after local DB changes
- Orchestrating multi-step operations that can partially fail

### CompensationService Pattern

The compensation pattern uses an "outbox" approach:
1. **Track** the operation BEFORE executing
2. **Execute** the operation
3. **Mark committed** AFTER success
4. Failed operations are automatically retried via scheduled task

```java
import com.svai.basecommon.compensation.CompensationService;
import com.svai.basecommon.compensation.PendingCompensation;
import com.svai.basecommon.compensation.CompensationType;

@Service
public class MyCompensationService extends CompensationService {
    
    @Autowired
    private ExternalApiClient externalApi;
    
    /**
     * Create resource with compensation tracking
     */
    @Transactional
    public MyEntity createWithCompensation(CreateRequest request) {
        // 1. Save entity locally first
        MyEntity entity = repository.save(new MyEntity(request));
        
        // 2. Track the compensation BEFORE external call
        PendingCompensation pending = trackOperation(
            CompensationType.CREATE,
            "external-resource",        // resource type
            entity.getId().toString(),  // resource id
            Map.of(                      // context for recovery
                "entityId", entity.getId(),
                "externalPayload", request.toExternalPayload()
            )
        );
        
        try {
            // 3. Make external call
            ExternalResult result = externalApi.create(request.toExternalPayload());
            
            // 4. Update entity with external reference
            entity.setExternalId(result.getExternalId());
            repository.save(entity);
            
            // 5. Mark as committed (removes from pending)
            markCommitted(pending);
            
            return entity;
            
        } catch (Exception e) {
            // Compensation will be retried by scheduled task
            log.error("External call failed, compensation pending: {}", pending.getId(), e);
            throw new CompensationException("External call failed", e);
        }
    }
    
    /**
     * Recovery method called by scheduled task for failed operations
     */
    @Override
    protected void recover(PendingCompensation pending) {
        switch (pending.getCompensationType()) {
            case CREATE -> {
                // Retry the external call
                Map<String, Object> context = pending.getContext();
                externalApi.create((String) context.get("externalPayload"));
            }
            case DELETE -> {
                // Rollback local changes
                UUID entityId = (UUID) pending.getContext().get("entityId");
                repository.deleteById(entityId);
            }
        }
    }
}
```

### Key Points:
1. `trackOperation()` - Records pending operation BEFORE external call
2. `markCommitted()` - Clears pending record AFTER success
3. Scheduler automatically retries failed operations with exponential backoff
4. Override `recover()` to define compensation logic

---

## Key References

| Document | Location | Purpose |
|----------|----------|---------|
| **SERVICE-IMPLEMENTATION-GUIDE.md** | `internal-docs/SERVICE-IMPLEMENTATION-GUIDE.md` | **⭐ QUICK REFERENCE - Consult FIRST for new/enhanced services** |
| **portable-connection-architecture.md** | `internal-docs/designs/portable-connection-architecture.md` | **⭐ Environment portability for connections** |
| spacevox-coding-patterns.md | `.github/spacevox-coding-patterns.md` | Detailed code patterns |
| schema_creation.sql | `infrastructure/local/db-init/01-create-users-and-schemas.sql` | Database setup |
| messaging-architecture.md | `internal-docs/docs/50-infrastructure/messaging-architecture.md` | Event patterns |
| LocalStack init | `infrastructure/local/localstack-init/` | Local AWS emulation |
| ci.yml | `.github/workflows/ci.yml` | CI/CD patterns |
| CompensationService | `base-common/.../compensation/` | Distributed transaction handling |
| SearchRequest | `base-common/.../search/` | Pagination pattern |
| NotificationPublisher | `base-data-exchange/.../notification/` | Send notifications from any service |

---

## Quick Decision Tree

```
Creating something new?
│
├─ NEW SERVICE
│   └─ Follow "Creating a NEW Service Checklist" (all items)
│
├─ NEW TABLE in existing service
│   └─ Create Flyway migration with RLS
│   └─ Create Entity extending Base with @EnableRLS
│   └─ Create Repo, Mapper, Service, Controller
│
├─ NEW DTO
│   └─ Check base-api-contracts first!
│   └─ Shared? → common/dto/
│   └─ Service-specific? → {service}/dto/
│
├─ NEW EVENT/MESSAGE
│   └─ Update LocalStack init scripts
│   └─ Update messaging-architecture.md
│   └─ Use @Topic + MessagePublisher for publishing
│   └─ Use IChannelEventHandler<T> for consuming
│
├─ NEW API ENDPOINT
│   └─ Use Constant.WF_SERVICE_MOUNT_POINT prefix
│   └─ Return ResponseEntity<T>
│   └─ Add OpenAPI annotations
│   └─ Controller delegates ALL logic to Service
│   └─ Use SearchRequest for pagination
│
├─ NEW MESSAGE HANDLER
│   └─ Implement IChannelEventHandler<T>
│   └─ Delegate ALL logic to Service
│
└─ CROSS-SERVICE OPERATION
    └─ Use CompensationService pattern
    └─ Track before → Execute → Mark committed after
```

---

## Architecture Principles Summary

| Principle | Correct | Incorrect |
|-----------|---------|-----------|
| Controller logic | Delegates to Service | Contains business logic |
| Message handler logic | Delegates to Service | Contains business logic |
| Publishing events | `@Topic` + `MessagePublisher` | Direct SNS/SQS calls |
| Consuming events | `IChannelEventHandler<T>` | Manual polling |
| Pagination | `SearchRequest` + `SearchSpecificationBuilder` | Custom page/size params |
| Distributed transactions | `CompensationService` | @Transactional across services |
| Notifications | `NotificationPublisher` with `NotificationCategory` | Hardcoded channels in service |
| DTOs | MapStruct | ModelMapper |
| **Connection references** | `ConnectionReference.binding("name")` | Direct connectionId UUIDs |
| **Data source origin** | `DataSourceOrigin` enum | Hardcoded origin assumptions |
| **Distributed locks** | `DistributedLockService.executeWithLock()` | Inline Redis `setIfAbsent` |
| **Get-or-create** | `ConcurrencyUtils.getOrCreate()` | Uncaught race conditions |
| **Optimistic retry** | `ConcurrencyUtils.withOptimisticRetry()` | No retry on version conflict |

---

## 🔄 CONCURRENCY PATTERNS (base-common.concurrency)

### Pattern 1: Get-or-Create (Singleton/Upsert)

When creating records that may already exist (singleton tables, first-access init):

```java
import com.svai.basecommon.concurrency.ConcurrencyUtils;

@Transactional(noRollbackFor = DataIntegrityViolationException.class)
public Config getOrCreateConfig() {
    return ConcurrencyUtils.getOrCreate(
        () -> repo.findByTenantId(tenantId),     // finder
        () -> repo.saveAndFlush(defaultConfig),   // creator (MUST use saveAndFlush)
        entityManager
    );
}
```

### Pattern 2: Distributed Lock (Cluster-Wide Singleton Operations)

When only ONE pod should execute an operation (scheduled jobs, sweeps):

```java
import com.svai.basecommon.concurrency.DistributedLockService;

@Autowired
private DistributedLockService lockService;

public Result doSweep() {
    var result = lockService.executeWithLock(
        "my-sweep-job",
        Duration.ofMinutes(30),
        () -> performActualSweep()
    );
    
    if (result.wasSkipped()) {
        log.info("Skipped: {}", result.getSkipReason());
        return Result.skipped();
    }
    return result.getValue();
}
```

### Pattern 3: Optimistic Lock Retry

When concurrent updates may conflict on the same entity:

```java
return ConcurrencyUtils.withOptimisticRetry(3, () -> {
    Order order = orderRepo.findById(orderId).orElseThrow();
    order.setStatus(newStatus);
    return orderRepo.save(order);
});
```

### Decision Guide

| Pattern | When to Use | Scope |
|---------|-------------|-------|
| `ConcurrencyUtils.getOrCreate()` | First one creates, others find | Database row |
| `DistributedLockService` | Only one instance should execute | Cluster-wide |
| `ConcurrencyUtils.withOptimisticRetry()` | Concurrent updates to same row | Single entity |
| `@Version` (existing) | Detect stale writes | Single entity |

---

## 📋 COPY-PASTE SNIPPETS

### Complete Entity Template
```java
// Replace: {EntityName}, {table_name}, {yourservice}
package com.svai.workflow.{yourservice}.entity;

import com.svai.baserls.annotation.EnableRLS;
import jakarta.persistence.*;

@Entity
@Table(name = "{table_name}")
@EnableRLS
public class {EntityName} extends Base {
    
    private String name;
    private String description;
    
    @Column(name = "name", nullable = false, length = 255)
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    
    @Column(name = "description", length = 2000)
    public String getDescription() { return description; }
    public void setDescription(String description) { this.description = description; }
}
```

### Complete MapStruct Mapper
```java
// Replace: {EntityName}, {yourservice}
package com.svai.workflow.{yourservice}.mapper;

import org.mapstruct.*;

@Mapper(componentModel = "spring", nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
public interface {EntityName}Mapper {
    
    {EntityName}Dto toDto({EntityName} entity);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "version", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "updatedBy", ignore = true)
    @Mapping(target = "tenantId", ignore = true)
    @Mapping(target = "envId", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    {EntityName} toEntity({EntityName}Dto dto);
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "version", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "createdBy", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    @Mapping(target = "updatedBy", ignore = true)
    @Mapping(target = "tenantId", ignore = true)
    @Mapping(target = "envId", ignore = true)
    @Mapping(target = "deleted", ignore = true)
    void updateEntity({EntityName}Dto dto, @MappingTarget {EntityName} entity);
}
```

### Complete Flyway Migration Header
```sql
-- V{N}__{description}.sql
-- Service: {service}
-- Date: YYYY-MM-DD
-- Author: {author}
-- Description: {what this migration does}

-- Create table with all required columns
CREATE TABLE IF NOT EXISTS wf_{schema}.{table_name} (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    env_id UUID,
    
    -- Domain columns here
    name VARCHAR(255) NOT NULL,
    
    -- Required columns
    deleted BOOLEAN DEFAULT FALSE,
    version INTEGER DEFAULT 0,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by VARCHAR(255),
    updated_by VARCHAR(255)
);

-- Index for RLS
CREATE INDEX IF NOT EXISTS idx_{table_name}_tenant ON wf_{schema}.{table_name}(tenant_id);

-- Enable RLS
ALTER TABLE wf_{schema}.{table_name} ENABLE ROW LEVEL SECURITY;

-- Tenant isolation policies
CREATE POLICY {table_name}_tenant_isolation ON wf_{schema}.{table_name}
    USING ('a8737fb0-0d10-4aab-809b-d54ef71dc8f1' = ANY(current_setting('app.tenant_id')::uuid[])
        OR (tenant_id = ANY(current_setting('app.tenant_id')::uuid[])
            AND (env_id IS NULL OR env_id = ANY(current_setting('app.env_id')::uuid[]))));

CREATE POLICY {table_name}_tenant_isolation_insert ON wf_{schema}.{table_name}
    FOR INSERT WITH CHECK ('a8737fb0-0d10-4aab-809b-d54ef71dc8f1' = ANY(current_setting('app.tenant_id')::uuid[])
        OR (tenant_id = ANY(current_setting('app.tenant_id')::uuid[])
            AND (env_id IS NULL OR env_id = ANY(current_setting('app.env_id')::uuid[]))));
```

### Service findById with Error Handling
```java
public EntityDto findById(UUID id) {
    return repository.findById(id)
        .map(mapper::toDto)
        .orElseThrow(() -> new EntityNotFoundException("Entity not found: " + id));
}
```

### Event Publisher Setup
```java
@Topic("my-events")  // Maps to messaging.topics.my-events in yml
@Service
@ConditionalOnProperty(name = "messaging.topics.my-events")
public class MyEventPublisher {
    @Autowired
    private MessagePublisher publisher;
    
    public void publish(BaseChannelEvent event) {
        publisher.publish(event);
    }
}
```

### Event Handler Setup
```java
@Component
public class MyEventHandler implements IChannelEventHandler<MyEvent> {
    @Autowired
    private MyService service;  // Delegate to service!
    
    @Override
    public void handle(MyEvent event) {
        service.processEvent(event);  // ALL logic in service
    }
}
```

---

## 🌐 FRONTEND PATTERNS (Designer, Work-Studio, Admin-UI)

### API Service Pattern
```typescript
// designer/src/api/services/{domain}Service.ts
import { apiClient, API_CONFIG } from '../shared/apiConfig';

export interface MyEntityDto {
  id: string;
  name: string;
  // ...
}

export const myService = {
  async getById(id: string): Promise<MyEntityDto> {
    const response = await apiClient.get(`/my-entities/${id}`);
    return response.data;
  },
  
  async create(dto: Partial<MyEntityDto>): Promise<MyEntityDto> {
    const response = await apiClient.post('/my-entities', dto);
    return response.data;
  },
  
  async search(params?: { page?: number; size?: number }): Promise<Page<MyEntityDto>> {
    const response = await apiClient.get('/my-entities', { params });
    return response.data;
  },
};
```

### Environment Variables
- All API URLs via `VITE_*` env vars in `.env.local`
- **Never hardcode localhost** - use `apiConfig.ts`
- Required vars validated in `validateApiConfig()`
- Example: `VITE_WORKFLOW_API_URL=http://localhost:8080/api/v1/workflow`

### State Management
| Type | Library | When to Use |
|------|---------|-------------|
| Server state | React Query / TanStack Query | API data, caching, refetching |
| Form state | React Hook Form | Form validation, submission |
| Global UI state | Zustand stores | Theme, sidebar, user prefs |
| Component state | useState/useReducer | Local component state |

### Component File Structure
```
components/
├── {domain}/
│   ├── {EntityName}List.tsx      # List/table view
│   ├── {EntityName}Detail.tsx    # Single entity view
│   ├── {EntityName}Form.tsx      # Create/edit form
│   └── {EntityName}Card.tsx      # Card component
```

### API Client Interceptors (auto-configured)
- `headerInterceptor`: Adds `X-SELECTED-TENANT`, `X-SELECTED-ENV`, `Authorization`
- `responseErrorInterceptor`: Handles 401/403/500 errors globally

---

## 🤖 AI RUNTIME PATTERNS

### Observability Event Flow
```
ai-runtime (AIMetricsRecorder)
    ↓
DataplaneObservabilityEventPublisher
    ↓
dataplane-agent (forwards to control plane)
    ↓
Kinesis Stream
    ↓
insights service (ClickHouseEventWriter)
    ↓
ClickHouse (observability.events table)
```

### AI Event Types
| Event Type | When Published | Key Fields |
|------------|---------------|------------|
| `SESSION_STARTED` | New conversation begins | sessionId, assistantId, assistantName |
| `SESSION_ENDED` | Conversation ends | sessionId, totalTurns, totalTokens, sessionDurationMs |
| `LLM_CALL` | Each model invocation | model, llmProvider, promptTokens, completionTokens, estimatedCostUsd, llmDurationMs |
| `TOOL_COMPLETED` | Tool returns successfully | toolName, toolType, toolDurationMs, toolSuccess=true |
| `TOOL_FAILED` | Tool throws error | toolName, toolError, toolSuccess=false |
| `ADMISSION_REJECTED` | Quota/rate limit hit | status (contains reason: concurrent_limit, token_quota, credits_exhausted) |
| `STREAM_COMPLETED` | Streaming response done | timeToFirstTokenMs, totalTokens |

### Adding New AI Metrics
1. **Add event type** to `AIEventType` enum in `base-api-contracts`
2. **Record the event** in `AIMetricsRecorder.java` (ai-runtime)
3. **Add ClickHouse column** if needed in `infrastructure/local/clickhouse-init/`
4. **Update ClickHouseEventWriter** to extract new field
5. **Query via service** in `AIInsightsService` or `ClickHouseAnalyticsService`
6. **Expose via API** in `AIInsightsController`
7. **Add frontend client** in `insightsService.ts`

### Key AI Runtime Classes
| Class | Purpose |
|-------|--------|
| `AIMetricsRecorder` | Records all AI metrics, publishes events |
| `SessionOrchestrator` | Manages conversation turns, tool calls |
| `UnifiedToolExecutor` | Executes MCP and built-in tools |
| `SubscriptionAwareResourceGovernor` | Admission control, quota enforcement |
| `ModelPricingService` | Token cost calculations |

### Governance Reasons
| Reason | When Triggered |
|--------|---------------|
| `concurrent_limit` | Too many simultaneous sessions |
| `token_quota` | Monthly token budget exhausted |
| `credits_exhausted` | Prepaid credits depleted |

---

## 🔧 TROUBLESHOOTING

### "No rows returned" / Empty results
1. ✅ Check tenant header present: `X-SELECTED-TENANT: {uuid}`
2. ✅ Check env header present: `X-SELECTED-ENV: {uuid}`
3. ✅ Verify using `_api_least` user (not `_api_admin`) in `local.env`
4. ✅ Verify RLS policy exists: `\d+ wf_schema.tablename` in psql
5. ✅ Check data actually exists with that tenant_id

### Compilation error after adding entity
1. ✅ JPA annotations on **getters**, not fields
2. ✅ Import `jakarta.persistence.*` (not `javax.persistence`)
3. ✅ Run `mvn clean compile` to regenerate MapStruct mappers
4. ✅ Entity extends local `Base.java`, not `BaseEntity` directly

### Event not received / published
1. ✅ Check LocalStack queue exists: `aws --endpoint-url=http://localhost:4566 sqs list-queues`
2. ✅ Check SNS subscription: `aws --endpoint-url=http://localhost:4566 sns list-subscriptions`
3. ✅ Check DLQ for failed messages
4. ✅ Verify `messaging.topics.xxx` configured in `application.yml`
5. ✅ Event class extends `BaseChannelEvent`
6. ✅ Handler implements `IChannelEventHandler<YourEventType>`

### 403 Forbidden on internal service call
1. ✅ Use `@Qualifier("internalRestTemplate")` for service-to-service calls
2. ✅ Target endpoint has `/internal/` in path
3. ✅ Target controller has `@InternalApi` annotation
4. ✅ Set `INTERNAL_AUTH_ENABLED=true` if required

### RLS policy not applying
1. ✅ Table has `ENABLE ROW LEVEL SECURITY` run
2. ✅ Using `_api_least` user (admin bypasses RLS)
3. ✅ Entity has `@EnableRLS` annotation
4. ✅ Both SELECT and INSERT policies exist

### MapStruct "Unmapped target property" warning
Add to mapper method:
```java
@Mapping(target = "problematicField", ignore = true)
```

### Hibernate "Unknown column" for helper method
Add `@Transient` annotation:
```java
@Transient
public boolean isActive() {
    return status == Status.ACTIVE;
}
```

### ClickHouse queries return empty
1. ✅ Check ClickHouse container running: `docker ps | grep clickhouse`
2. ✅ Verify data being written: check `insights` service logs
3. ✅ Check table exists: `clickhouse-client -q "SHOW TABLES FROM observability"`
4. ✅ Verify tenant/env filters in query

### Frontend API call fails with CORS
1. ✅ Check `CORS_ALLOWED_ORIGINS` in backend `local.env`
2. ✅ Verify frontend URL is in allowed origins list
3. ✅ Check browser dev tools Network tab for preflight (OPTIONS) response
