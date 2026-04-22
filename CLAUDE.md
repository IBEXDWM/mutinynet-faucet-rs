# Service Architecture Overview

This document outlines the architectural patterns and structure used in our microservices. We follow a Domain-Driven Design (DDD) inspired layered architecture, emphasizing separation of concerns, testability, and modularity.

---

## 1. High-Level Architecture Patterns

Dependencies flow in one direction: **Outer Layers → Inner Layers**.

| Layer | Location | Responsibility |
|---|---|---|
| Transport | `internal/servers` | Handles incoming requests (gRPC/HTTP) |
| Service | `internal/services` | Orchestrates workflows and business rules |
| Domain | `internal/domains` | Core entities and logic, DB/transport agnostic |
| Repository | `internal/repos` | Database interaction abstractions |
| Infrastructure | `pkg/` | Cross-cutting concerns (logging, errors, clients) |

---

## 2. Directory Structure & Responsibilities

### `/apis` — Contract Definition
- `/grpc`: Proto definitions and generated Go code for gRPC.
- `/swagger`: OpenAPI/Swagger specs for REST endpoints.

### `/internal` — Private Implementation

#### `internal/domains` — The Core
- Pure Go structs (Entities) and Value Objects.
- **No external dependencies** (no SQL, no HTTP drivers).

#### `internal/servers` — The Entry Point
- `/grpc`: Implements gRPC server interface.
- `/rest`: Gin controllers — parse JSON, validate input, call Services.

#### `internal/services` — The Brain
- Orchestrates business logic.
- Calls Repositories or other Services.
- Responsible for background tasks (e.g., RabbitMQ listeners).

#### `internal/repos` — The Storage
- Repository Pattern: each repo defined by an interface.
- Raw SQL or ORM mapping `internal/domains` objects to DB tables.

### `/pkg` — Shared Infrastructure
Domain-agnostic reusable building blocks. Should compile and be useful in any service without modification.

- **Infrastructure Wrappers**: `rabbit`, `http_client`, `secrets` — standardizes external infra usage.
- **Observability**: `logger`, `otel` — consistent logs/traces across services.
- **Vendor Integrations**: `ramp_provider_clients` (Bitso, BAM, Hercle, etc.) — treated as external infra.
- **Low-Level Utilities**: `floats`, `random`, `sanitize`.

---

## 3. Communication Flow Example — "Create Deposit"

1. **Transport** (`internal/servers/rest`): `DepositController` receives JSON, validates via `validators/body.go`, converts to domain struct, calls `DepositService.Create()`.
2. **Service** (`internal/services/deposit_infos`): Checks business rules, may call `RampProviderClient` in `/pkg`, calls `DepositRepo.Create()`.
3. **Repository** (`internal/repos/deposit_infos`): Executes `INSERT INTO`, returns entity or DB error.

---

## 4. Key Architectural Decisions

- **Interface-First**: All Services and Repos defined by interfaces (`service_interface.go`, `repo_interface.go`), enabling mock generation (`*_mock.go`).
- **Error Handling**: Errors are wrapped with context as they bubble up Repo → Service → Controller, using `pkg/errors`.
- **Migrations**: Versioned Go files in `internal/db/migrations`.

---

## 5. Database Migrations (MigrateX)

We use **MigrateX**, our internal migration engine. Migrations are written in Go for compiler safety.

### Setup
```bash
GOPRIVATE=github.com/IBEXDWM/* go install github.com/IBEXDWM/migratex/mgx@latest
```

### Creating a Migration
```bash
cd internal/db/migrations
mgx n create_users_table
```
This generates a file prefixed with a unix timestamp, e.g., `1741943860_create_users_table.go`.

### Migration File Structure
```go
package migrations

import (
    "github.com/IBEXDWM/migratex"
    "gorm.io/gorm"
)

func init() {
    migratex.Register(&_1741943860_create_users_table{})
}

type _1741943860_create_users_table struct{}

func (m *_1741943860_create_users_table) Up(db *gorm.DB) error {
    type User struct {
        ID    uint   `gorm:"primaryKey"`
        Email string `gorm:"unique"`
    }
    err := db.AutoMigrate(&User{})
    if err != nil {
        return fmt.Errorf("[create table] %w", err)
    }
    return nil
}

func (m *_1741943860_create_users_table) Down(db *gorm.DB) error {
    return nil
}

func (m *_1741943860_create_users_table) ID() string {
    return "1741943860_create_users_table"
}
```

### How it Works
1. `init()` registers the migration with the global migratex registry.
2. On startup, `migratex.ApplyPendingMigrations` is called.
3. It checks the `migrations` table, sorts pending migrations by timestamp, and executes them in a DB transaction. Any failure rolls back the entire batch.

### Best Practices
- Write `IF NOT EXISTS` logic for idempotency.
- Define structs inside the migration file — **never** use `internal/domains` structs. Domain structs change; migrations must be static snapshots.

---

## 6. Development & Automation (Makefile)

### `make deps`
- Configures Git to use SSH for private org repos (`github.com/IBEXDWM/*`).
- Fetches `migratex`.
- Runs `go mod vendor` for reproducible builds.
- **Run after every dependency change and regularly on each service.**

### Code Generation

| Command | Tool | Purpose |
|---|---|---|
| `make mocks` | mockery | Generates `*_mock.go` from interface files |
| `make proto` | protoc | Compiles `.proto` → `.pb.go` for gRPC |
| `make swagger-spec` | swag | Generates `swagger.yaml` from Go comments |
| `make swagger-serve` | UI server | Visualizes API docs locally |

### Quality Assurance

| Command | Tool | Notes |
|---|---|---|
| `make lint` | golangci-lint | Build pipeline fails on any lint errors |
| `make tests` | ginkgo | `-r --cover --skip-package ./tests/` |

---

## 7. Logger Package (`/pkg/logger`)

### Overview
Wrapper around `logrus`. Provides structured JSON logging with automatic Datadog integration (trace/span IDs).

### Initialization
```go
log, err := logger.NewLogger("my-service", envConfig)
if err != nil {
    panic(err)
}
```

### Log Levels & Alerting

| Level | Method | Alerting Behavior |
|---|---|---|
| Panic | `Panic()`, `Panicf()` | CRITICAL: Pages on-call (PagerDuty) + Slack |
| Fatal | `Fatal()`, `Fatalf()` | CRITICAL: Pages on-call (PagerDuty) + Slack |
| Error | `Error()`, `Errorf()` | CRITICAL: Pages on-call (PagerDuty) + Slack |
| Warn | `Warn()`, `Warnf()` | WARNING: Slack notification (no page) |
| Info | `Info()`, `Infof()` | No alert — stored for historical analysis |
| Debug | `Debug()`, `Debugf()` | No alert |
| Trace | `Trace()`, `Tracef()` | No alert |

> **Warning:** Do NOT use `Error`, `Fatal`, or `Panic` for expected validation errors ("user not found", "invalid input"). Use `Info` or `Warn` for business logic failures that don't indicate system malfunction.

### Usage

```go
// Basic
log.Info("Service started successfully")
log.Errorf("Database connection failed: %v", err) // triggers PagerDuty

// With context (preferred inside request handlers)
s.log.WithCtx(ctx).Infof("Fetching user %s", id)

// With depth (for helper functions)
s.log.WithDepth(4).Info("Log from helper")
```

---

## 8. Golang Standards

- Functions take **at most 2 parameters** and return **at most 2 variables**.
- First parameter is always `ctx context.Context`; second (if any) can be a pointer to a struct named `FuncNameInput` or `FuncNameIn`.
- Return values: first is the result (standard type, pointer, or interface); second is always `error`. Returning an error is mandatory.
- If the output is a struct (not a domain), name it `FuncNameOutput` or `FuncNameOut`.
- **80 chars max per line.**
- Do not declare structs inside other structs, function calls, or return statements.

---

## 9. Error Management

We use a custom error wrapper in `/pkg/errors` — never raw Go errors.

### A. Sentinel Errors (package-level)
```go
var (
    ErrForbidden   = errs.Newf(codes.PermissionDenied, "forbidden")
    ErrInvalidID   = errs.Newf(codes.InvalidArgument, "invalid ID format")
    ErrUserBlocked = errs.Newf(codes.FailedPrecondition, "user is blocked")
)
```

### B. Wrapping External Library Errors (leaf nodes)
```go
ipsStr, err := json.Marshal(in.IPs)
if err != nil {
    return nil, errs.Wrapf(err, codes.Internal, "[json.Marshal]")
}
```

### C. Bubbling Up Internal Errors
```go
dInfos, err := s.services.DepositInfos.GetByID(ctx, uuid)
if err != nil {
    // Do NOT use errs.Wrapf — error already has a code
    return nil, fmt.Errorf("[dInfoServGetByID] %w", err)
}
```

### Returning Errors to Clients

**gRPC:**
```go
return nil, s.RenderError(ctx, fmt.Errorf("[dInfoServGetByID] %w", err))
```

**REST/Gin:**
```go
httpErrs.ErrResp(c, fmt.Errorf("[acc.SetAccountInContext] %w", err))
return
```

### Error Code → HTTP/gRPC Mapping

| Internal Code | Log Level | gRPC Code | HTTP Status |
|---|---|---|---|
| Unknown | ERROR | Unknown | 400 |
| Canceled | INFO | Canceled | 499 |
| InvalidArgument | INFO | InvalidArgument | 400 |
| DeadlineExceeded | WARN | DeadlineExceeded | 504 |
| NotFound | INFO | NotFound | 404 |
| AlreadyExists | INFO | AlreadyExists | 409 |
| PermissionDenied | INFO | PermissionDenied | 403 |
| ResourceExhausted | WARN | ResourceExhausted | 429 |
| FailedPrecondition | INFO | FailedPrecondition | 400 |
| Aborted | WARN | Aborted | 409 |
| OutOfRange | INFO | OutOfRange | 400 |
| Unimplemented | ERROR | Unimplemented | 501 |
| Internal | ERROR | Internal | 500 |
| Unavailable | ERROR | Unavailable | 503 |
| DataLoss | ERROR | DataLoss | 500 |
| Unauthenticated | INFO | Unauthenticated | 401 |

> `Internal` and `Unknown` errors return a generic `"internal error"` message to the client — the full stack trace stays in private logs only.

### Stack Trace Example

**System error (500):**
```
ERRO[0012] RenderError: [DepositHandler.Create] [DepositService.Validate] [AccountRepo.GetByID] connection refused
```
Client receives: `{"error": "internal error"}`

**User error (409):**
```
INFO[0045] RenderError: [RegisterHandler.Create] [AuthService.Register] [UserRepo.Create] user already exists
```
Client receives: `{"error": "user already exists"}`

---

## 10. REST API Endpoint Standards

### Handler Signature
Every handler is a function that accepts its dependencies and returns a `gin.HandlerFunc`. Dependencies are injected as interfaces.

```go
func Create(
    authService authServ.Service,
    orgService  orgServ.Service,
) gin.HandlerFunc {
    return func(c *gin.Context) { ... }
}
```

### Input Structs
- Named `*Input` (e.g., `CreateInput`, `GetOrganizationsInput`).
- Body inputs use `json:`, `binding:`, `conform:`, and `validate:` tags.
- Query param inputs use `form:` tags.

```go
// Body input
type CreateInput struct {
    AdminEmail string `json:"adminEmail" binding:"required" conform:"email,trim,lower" validate:"email"`
    AdminName  string `json:"adminName"  binding:"required" conform:"name,trim"`
}

// Query params input
type ListInput struct {
    Page  int    `form:"page,default=0"`
    Limit int    `form:"limit,default=10"`
    Email string `form:"email"`
}
```

### Validation
- **Never** call `c.ShouldBindJSON` or `c.ShouldBindQuery` directly.
- Always use the validator wrappers:

```go
// For request body
var payload CreateInput
if err := validators.ValidateBody(c, &payload); err != nil {
    httpErrs.ErrResp(c, err)
    return
}

// For query parameters
var params ListInput
if err := validators.ValidateQueryParams(c, &params); err != nil {
    httpErrs.ErrResp(c, fmt.Errorf("valid: %w", err))
    return
}
```

### Error Responses
- Always use `httpErrs.ErrResp(c, err)` — never `c.JSON` with an error value.
- Always `return` immediately after `ErrResp`.

```go
org, err := orgService.GetByID(ctx, id)
if err != nil {
    httpErrs.ErrResp(c, fmt.Errorf("[orgServ.GetByID] %w", err))
    return
}
```

### Success Responses
```go
c.JSON(http.StatusOK, output)
// or for creates:
c.JSON(http.StatusCreated, output)
```

### Output Structs
- Map from domain objects inside the handler — never return raw domain structs.
- Named `*Output` or a named collection type.

### Context
```go
// Prefer c.Request.Context() over casting c directly
ctx := c.Request.Context()
```

### Routes Registration
- Every controller package has a `routes.go` file with a `SetRoutes` function.
- Middleware (JWT, permissions) is applied at the group level.

```go
func SetRoutes(
    r           *gin.Engine,
    fundService fundServ.Service,
    authService authServ.Service,
) {
    g := r.Group("/funds")
    g.Use(middlewares.JWTVerify(authService))
    g.POST("",
        middlewares.HasPermission(authService, perms.CreateFunds),
        Create(fundService),
    )
}
```

---

## 11. Unit Test Standards

### Framework
- **Ginkgo v2** (`github.com/onsi/ginkgo/v2`) — BDD test runner.
- **Gomega** (`github.com/onsi/gomega`) — assertion library.
- **testify/mock** (`github.com/stretchr/testify/mock`) — for generated mocks.

### Package Naming
Test files must use the external test package pattern:
```go
package organizations_test  // NOT package organizations
```

### Suite File (required per package)
Every test package needs a `*_suite_test.go` file:
```go
package organizations_test

import (
    "testing"
    "github.com/gin-gonic/gin"
    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    "IBEXHub-V2/internal/server/validators"
)

func TestOrganizations(t *testing.T) {
    _ = validators.InitValidator()
    gin.SetMode(gin.TestMode)
    RegisterFailHandler(Fail)
    RunSpecs(t, "Organizations Suite")
}
```

### Test Structure
Use `Describe` → `Context`/`When` → `It`. Variables are declared at the top of `Describe`, initialized in `BeforeEach`.

```go
var _ = Describe("GetAll", func() {
    var (
        c          *gin.Context
        orgService *orgServ.MockService
    )

    BeforeEach(func() {
        c, _ = gin.CreateTestContext(httptest.NewRecorder())
        c.Request = httptest.NewRequest(http.MethodGet, "/orgs", http.NoBody)
        orgService = &orgServ.MockService{}
    })

    Context("when the service returns an error", func() {
        BeforeEach(func() {
            orgService.On("GetAll", mock.Anything, mock.Anything).
                Return(nil, orgDom.ErrNotFound)
        })

        It("returns 404", func() {
            GetAll(orgService)(c)
            Expect(c.Writer.Status()).To(Equal(http.StatusNotFound))
        })
    })
})
```

### Mocks
- **Never** use real services or repos in unit tests — always use generated mocks.
- Mocks are generated via `make mocks` and live at `*_mock.go` (e.g., `service_mock.go`).
- Use `mock.Anything` for context args, `mock.MatchedBy` for precise input assertions.

### Response Capture
Use the `bodyLogWriter` helper (defined in the suite file) to capture response bodies:

```go
type bodyLogWriter struct {
    gin.ResponseWriter
    body *bytes.Buffer
}

func (w bodyLogWriter) Write(b []byte) (int, error) {
    w.body.Write(b)
    return w.ResponseWriter.Write(b)
}
```

### JSON Helpers (suite file)
```go
func writeJSON(payload interface{}) (io.ReadCloser, error) {
    b, err := json.Marshal(payload)
    if err != nil {
        return nil, err
    }
    return io.NopCloser(bytes.NewBuffer(b)), nil
}

func readJSON(s string, v interface{}) error {
    return json.Unmarshal([]byte(s), v)
}
```

### Assertions
- Use `Expect(x).To(Equal(y))` — never `t.Error`, `t.Fatal`, or `assert.*`.
- Use `Expect(err).ToNot(HaveOccurred())` for error checks.
- Use `Expect(c.Writer.Status()).To(Equal(http.StatusOK))` for HTTP status checks.

### Running Tests
```bash
make tests    # full unit test suite (skips /tests integration folder)
make lint     # must pass before merging
```
