# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Run the application (app stays running; Ctrl+C to stop — Maven prints BUILD FAILURE on stop, which is normal)
mvn spring-boot:run

# Build a runnable JAR (preferred for clean start/stop)
mvn package -DskipTests
java -jar target/hospital-management-system-1.0.0.jar

# Compile only (triggers MapStruct code generation into target/generated-sources/annotations/)
mvn compile

# Run all tests
mvn test

# Run a single test class
mvn test -Dtest=PatientServiceTest

# Run a single test method
mvn test -Dtest=PatientServiceTest#createPatient_Success

# Clean and recompile (required when MapStruct mappers behave unexpectedly)
mvn clean compile
```

**Prerequisites:** MySQL 8 must be running on `localhost:3306`. Update `src/main/resources/application.yml` if your MySQL root password differs from `root`. The database `hms_db` is auto-created on first startup.

## URLs (when running)

| Resource | URL |
|---|---|
| Swagger UI | `http://localhost:8080/api/v1/swagger-ui/index.html` |
| OpenAPI JSON | `http://localhost:8080/api/v1/api-docs` |

All non-auth endpoints require `Authorization: Bearer <token>`. Get a token via `POST /api/v1/auth/login`.

## Architecture

### Package layout

Base package: `com.hms`. Every business domain is a self-contained sub-package with the same internal layout:

```
<module>/
  controller/     REST layer — delegates directly to service, no logic
  service/        Interface
  service/impl/   Implementation — all business logic lives here
  repository/     Spring Data JPA interface
  entity/         JPA entity (extends BaseEntity)
  dto/            Request + Response POJOs (never expose entities)
  mapper/         MapStruct interface (Spring-managed bean)
  validator/      Domain-specific validation helpers (optional)
```

Modules: `auth`, `patient`, `doctor`, `appointment`, `medicalrecord`, `prescription`, `billing`  
Cross-cutting: `common` (BaseEntity, ApiResponse, enums), `exception`, `security`, `config`

### Key design decisions

**BaseEntity** (`common/entity/BaseEntity.java`) — all entities extend this. It provides `createdAt`, `updatedAt`, `createdBy`, `updatedBy` via Spring Data JPA auditing. `AuditConfig` resolves the current auditor from the Spring Security context.

**MapStruct + Lombok interaction** — entities use `@Builder`, but Lombok's `@Builder` does not include fields inherited from `BaseEntity`. As a result, all `toEntity()` mapper methods must include `@BeanMapping(builder = @Builder(disableBuilder = true))` to force MapStruct to use setters instead of the builder. Omitting this causes a compile error (`Unknown property "createdAt" in result type XxxBuilder`).

**JWT flow** — `JwtAuthenticationFilter` extracts the Bearer token, validates it via `JwtTokenProvider` (JJWT 0.12.x API), loads `UserPrincipal` from `CustomUserDetailsService`, and sets the `SecurityContext`. Roles are stored as `ROLE_<ENUM_NAME>` (e.g. `ROLE_ADMIN`). Method-level security (`@PreAuthorize`) uses `hasRole('ADMIN')` which matches against the `ROLE_` prefix automatically.

**Appointment conflict detection** — `AppointmentRepository` has two named queries (`findConflictingAppointment` / `findConflictingAppointmentExcluding`) that check for overlapping doctor time slots while excluding `CANCELLED` and `NO_SHOW` statuses. The service calls the `Excluding` variant during reschedule so the appointment being rescheduled does not conflict with itself.

**ApiResponse wrapper** — all controllers return `ApiResponse<T>` (from `common/dto/ApiResponse.java`). Use the static factory methods: `ApiResponse.success(data)`, `ApiResponse.success(message, data)`, `ApiResponse.error(message)`.

**Exception handling** — throw domain exceptions (`ResourceNotFoundException`, `AppointmentConflictException`, `DoctorUnavailableException`, `BusinessValidationException`) from service layer. `GlobalExceptionHandler` (`@RestControllerAdvice`) maps them to HTTP status codes and `ErrorResponse` bodies.

**`ddl-auto: update`** — schema is managed automatically by Hibernate. No migration tool is configured.
