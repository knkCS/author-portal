# Foundation & Backend Setup — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Set up the portal backend Go service with database schemas, odon JWT authentication, tenant resolution, guardian permissions, and the branding API — the foundation everything else builds on.

**Architecture:** A Go HTTP server (net/http + Connect-RPC, matching odon's patterns) with Ent ORM for the portal's own PostgreSQL database. Validates odon JWTs via JWKS discovery, resolves tenants from request hostname, and checks permissions via guardian-go SDK. Two entrypoints: `cmd/api` (HTTP server) and `cmd/sync` (worker, scaffolded only).

**Tech Stack:** Go 1.26, Ent ORM, PostgreSQL, Redis, Connect-RPC, lestrrat-go/jwx v2, guardian-go SDK, Docker Compose

**Plan scope:** This is plan 1 of 5 for the MVP. Subsequent plans:
- Plan 2: Frontend Shell & Auth Flow
- Plan 3: Author Profiles & Communication
- Plan 4: Manuscript Management
- Plan 5: Production Workflow & Basic ERP Sync

**Repos referenced:**
- odon: `/Users/jeskoiwanovski/repo/odon` — patterns for Connect-RPC, JWT, Ent
- guardian-go: `/Users/jeskoiwanovski/repo/guardian-go` — permission check SDK
- core CMS: `/Users/jeskoiwanovski/repo/core` — CMS API for future proxy integration

---

## File Structure

```
author-portal-backend/
├── cmd/
│   ├── api/
│   │   └── main.go                    # HTTP server entrypoint
│   └── sync/
│       └── main.go                    # Sync worker entrypoint (scaffold)
├── internal/
│   ├── server/
│   │   ├── server.go                  # HTTP server setup, middleware chain, mux
│   │   └── server_test.go
│   ├── middleware/
│   │   ├── auth.go                    # odon JWT validation via JWKS
│   │   ├── auth_test.go
│   │   ├── tenant.go                  # Tenant resolution from hostname
│   │   ├── tenant_test.go
│   │   └── context.go                 # Context helpers (UserID, TenantID, etc.)
│   ├── handler/
│   │   ├── branding/
│   │   │   ├── handler.go             # Branding CRUD (Connect-RPC)
│   │   │   └── handler_test.go
│   │   └── health/
│   │       └── handler.go             # Health check
│   ├── storage/
│   │   └── ent/
│   │       └── schema/
│   │           ├── tenant.go
│   │           ├── message.go
│   │           ├── notification.go
│   │           ├── notification_preference.go
│   │           ├── sync_job.go
│   │           └── entity_sync_config.go
│   ├── guardian/
│   │   ├── schema.go                  # Portal permission schema definition
│   │   └── schema_test.go
│   └── config/
│       └── config.go                  # App configuration (env vars)
├── api/
│   └── portal/
│       └── v1/
│           ├── branding.proto         # Branding service proto
│           └── branding_connect.go    # Generated Connect-RPC code
├── docker-compose.yml
├── Dockerfile
├── go.mod
├── go.sum
└── Makefile
```

---

### Task 1: Initialize Go Module & Project Structure

**Files:**
- Create: `author-portal-backend/go.mod`
- Create: `author-portal-backend/cmd/api/main.go`
- Create: `author-portal-backend/cmd/sync/main.go`
- Create: `author-portal-backend/Makefile`

- [ ] **Step 1: Create project directory and Go module**

```bash
mkdir -p /Users/jeskoiwanovski/repo/author-portal-backend
cd /Users/jeskoiwanovski/repo/author-portal-backend
go mod init github.com/knkCS/author-portal-backend
```

- [ ] **Step 2: Create directory structure**

```bash
mkdir -p cmd/api cmd/sync
mkdir -p internal/server internal/middleware internal/handler/branding internal/handler/health
mkdir -p internal/storage/ent/schema internal/guardian internal/config
mkdir -p api/portal/v1
```

- [ ] **Step 3: Write API entrypoint**

Create `cmd/api/main.go`:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/knkCS/author-portal-backend/internal/config"
	"github.com/knkCS/author-portal-backend/internal/server"
)

func main() {
	cfg, err := config.Load()
	if err != nil {
		slog.Error("failed to load config", "error", err)
		os.Exit(1)
	}

	srv, err := server.New(cfg)
	if err != nil {
		slog.Error("failed to create server", "error", err)
		os.Exit(1)
	}

	httpServer := &http.Server{
		Addr:         fmt.Sprintf(":%d", cfg.Port),
		Handler:      srv.Handler(),
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
	}

	go func() {
		slog.Info("starting server", "port", cfg.Port)
		if err := httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			slog.Error("server error", "error", err)
			os.Exit(1)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := httpServer.Shutdown(ctx); err != nil {
		slog.Error("shutdown error", "error", err)
	}
	slog.Info("server stopped")
}
```

- [ ] **Step 4: Write sync worker scaffold**

Create `cmd/sync/main.go`:

```go
package main

import (
	"log/slog"
)

func main() {
	slog.Info("sync worker starting — not yet implemented")
}
```

- [ ] **Step 5: Write config loader**

Create `internal/config/config.go`:

```go
package config

import (
	"fmt"
	"os"
	"strconv"
)

type Config struct {
	Port        int
	DatabaseURL string
	RedisURL    string
	OdonJWKSURL string
	GuardianURL string
	CMSURL      string
}

func Load() (*Config, error) {
	port, _ := strconv.Atoi(getEnv("PORT", "8090"))
	dbURL := getEnv("DATABASE_URL", "")
	if dbURL == "" {
		return nil, fmt.Errorf("DATABASE_URL is required")
	}
	return &Config{
		Port:        port,
		DatabaseURL: dbURL,
		RedisURL:    getEnv("REDIS_URL", "redis://localhost:6379"),
		OdonJWKSURL: getEnv("ODON_JWKS_URL", "http://localhost:8080/.well-known/jwks.json"),
		GuardianURL: getEnv("GUARDIAN_URL", "http://localhost:8081"),
		CMSURL:      getEnv("CMS_URL", "http://localhost:8082"),
	}, nil
}

func getEnv(key, fallback string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return fallback
}
```

- [ ] **Step 6: Write Makefile**

Create `Makefile`:

```makefile
.PHONY: api sync generate test lint

api:
	go run ./cmd/api

sync:
	go run ./cmd/sync

generate:
	go generate ./...

test:
	go test ./... -race -v

lint:
	go vet ./...
```

- [ ] **Step 7: Commit**

```bash
git init && git branch -M main
git add .
git commit -m "feat: initialize Go project structure with api and sync entrypoints"
```

---

### Task 2: Docker Compose & Database Setup

**Files:**
- Create: `author-portal-backend/docker-compose.yml`
- Create: `author-portal-backend/Dockerfile`

- [ ] **Step 1: Write docker-compose.yml**

Create `docker-compose.yml`:

```yaml
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: portal
      POSTGRES_PASSWORD: portal
      POSTGRES_DB: author_portal
    ports:
      - "5433:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6380:6379"

  api:
    build:
      context: .
      target: api
    ports:
      - "8090:8090"
    environment:
      PORT: "8090"
      DATABASE_URL: "postgres://portal:portal@postgres:5432/author_portal?sslmode=disable"
      REDIS_URL: "redis://redis:6379"
      ODON_JWKS_URL: "${ODON_JWKS_URL:-http://host.docker.internal:8080/.well-known/jwks.json}"
      GUARDIAN_URL: "${GUARDIAN_URL:-http://host.docker.internal:8081}"
      CMS_URL: "${CMS_URL:-http://host.docker.internal:8082}"
    depends_on:
      - postgres
      - redis

volumes:
  pgdata:
```

- [ ] **Step 2: Write multi-stage Dockerfile**

Create `Dockerfile`:

```dockerfile
FROM golang:1.26-alpine AS build
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .

FROM build AS api
RUN go build -o /api ./cmd/api
ENTRYPOINT ["/api"]

FROM build AS sync
RUN go build -o /sync ./cmd/sync
ENTRYPOINT ["/sync"]
```

- [ ] **Step 3: Verify compose starts**

```bash
docker compose up -d postgres redis
docker compose ps
```

Expected: postgres and redis containers running.

- [ ] **Step 4: Commit**

```bash
git add docker-compose.yml Dockerfile
git commit -m "feat: add Docker Compose with PostgreSQL and Redis"
```

---

### Task 3: Ent Schemas

**Files:**
- Create: `internal/storage/ent/schema/tenant.go`
- Create: `internal/storage/ent/schema/message.go`
- Create: `internal/storage/ent/schema/notification.go`
- Create: `internal/storage/ent/schema/notification_preference.go`
- Create: `internal/storage/ent/schema/sync_job.go`
- Create: `internal/storage/ent/schema/entity_sync_config.go`
- Create: `internal/storage/ent/generate.go`

- [ ] **Step 1: Add Ent dependency**

```bash
go get entgo.io/ent@latest
```

- [ ] **Step 2: Write tenant schema**

Create `internal/storage/ent/schema/tenant.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type Tenant struct {
	ent.Schema
}

func (Tenant) Fields() []ent.Field {
	return []ent.Field{
		field.String("name").NotEmpty(),
		field.String("slug").Unique().NotEmpty(),
		field.String("custom_domain").Optional().Nillable(),
		field.String("odon_org_id").Unique().NotEmpty().Comment("odon organization ID"),
		field.JSON("branding", map[string]any{}).Optional().Comment("logo, colors, fonts"),
		field.Bool("active").Default(true),
		field.Time("created_at").Immutable().Default(timeNow),
		field.Time("updated_at").Default(timeNow).UpdateDefault(timeNow),
	}
}

func (Tenant) Edges() []ent.Edge {
	return []ent.Edge{
		edge.To("messages", Message.Type),
		edge.To("sync_configs", EntitySyncConfig.Type),
		edge.To("sync_jobs", SyncJob.Type),
	}
}

func (Tenant) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("custom_domain").Unique(),
	}
}
```

- [ ] **Step 3: Write message schema**

Create `internal/storage/ent/schema/message.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type Message struct {
	ent.Schema
}

func (Message) Fields() []ent.Field {
	return []ent.Field{
		field.String("tenant_id").NotEmpty(),
		field.String("thread_id").NotEmpty().Comment("groups messages in a conversation"),
		field.String("title_id").Optional().Nillable().Comment("CMS title content ID, if tied to a title"),
		field.String("sender_id").NotEmpty().Comment("odon user ID"),
		field.String("sender_name").NotEmpty(),
		field.String("body").NotEmpty(),
		field.String("parent_id").Optional().Nillable().Comment("for threaded replies"),
		field.Time("created_at").Immutable().Default(timeNow),
	}
}

func (Message) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("tenant", Tenant.Type).Ref("messages").Unique().Required(),
	}
}

func (Message) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("tenant_id", "thread_id"),
		index.Fields("tenant_id", "title_id"),
	}
}
```

- [ ] **Step 4: Write notification schema**

Create `internal/storage/ent/schema/notification.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type Notification struct {
	ent.Schema
}

func (Notification) Fields() []ent.Field {
	return []ent.Field{
		field.String("tenant_id").NotEmpty(),
		field.String("user_id").NotEmpty().Comment("odon user ID of recipient"),
		field.String("type").NotEmpty().Comment("e.g. message, approval_needed, status_change"),
		field.String("title").NotEmpty(),
		field.String("body").NotEmpty(),
		field.String("link").Optional().Nillable().Comment("deep link into portal"),
		field.Bool("read").Default(false),
		field.Bool("email_sent").Default(false),
		field.Time("created_at").Immutable().Default(timeNow),
		field.Time("read_at").Optional().Nillable(),
	}
}

func (Notification) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("tenant_id", "user_id", "read"),
		index.Fields("user_id", "created_at"),
	}
}
```

- [ ] **Step 5: Write notification preference schema**

Create `internal/storage/ent/schema/notification_preference.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type NotificationPreference struct {
	ent.Schema
}

func (NotificationPreference) Fields() []ent.Field {
	return []ent.Field{
		field.String("tenant_id").NotEmpty(),
		field.String("user_id").NotEmpty(),
		field.String("event_type").NotEmpty().Comment("e.g. message, approval_needed, status_change"),
		field.Bool("in_app").Default(true),
		field.Bool("email").Default(true),
		field.Time("updated_at").Default(timeNow).UpdateDefault(timeNow),
	}
}

func (NotificationPreference) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("tenant_id", "user_id", "event_type").Unique(),
	}
}
```

- [ ] **Step 6: Write sync job schema**

Create `internal/storage/ent/schema/sync_job.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type SyncJob struct {
	ent.Schema
}

func (SyncJob) Fields() []ent.Field {
	return []ent.Field{
		field.String("tenant_id").NotEmpty(),
		field.String("entity_type").NotEmpty().Comment("e.g. titles, contracts, royalties"),
		field.Enum("status").Values("pending", "running", "completed", "failed").Default("pending"),
		field.Int("records_synced").Default(0),
		field.String("error_message").Optional().Nillable(),
		field.Time("started_at").Optional().Nillable(),
		field.Time("completed_at").Optional().Nillable(),
		field.Time("created_at").Immutable().Default(timeNow),
	}
}

func (SyncJob) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("tenant", Tenant.Type).Ref("sync_jobs").Unique().Required(),
	}
}

func (SyncJob) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("tenant_id", "status"),
		index.Fields("tenant_id", "entity_type", "created_at"),
	}
}
```

- [ ] **Step 7: Write entity sync config schema**

Create `internal/storage/ent/schema/entity_sync_config.go`:

```go
package schema

import (
	"entgo.io/ent"
	"entgo.io/ent/schema/edge"
	"entgo.io/ent/schema/field"
	"entgo.io/ent/schema/index"
)

type EntitySyncConfig struct {
	ent.Schema
}

func (EntitySyncConfig) Fields() []ent.Field {
	return []ent.Field{
		field.String("tenant_id").NotEmpty(),
		field.String("entity").NotEmpty().Comment("e.g. production-milestone, author-profile"),
		field.Enum("source_of_truth").Values("erp", "cms", "bidirectional").Default("erp"),
		field.Enum("sync_direction").Values("erp_to_cms", "cms_to_erp", "both").Default("erp_to_cms"),
		field.Enum("conflict_rule").Values("erp_wins", "cms_wins", "latest_wins", "manual").Default("erp_wins"),
		field.Enum("sync_frequency").Values("realtime", "hourly", "daily", "manual").Default("daily"),
		field.Time("updated_at").Default(timeNow).UpdateDefault(timeNow),
	}
}

func (EntitySyncConfig) Edges() []ent.Edge {
	return []ent.Edge{
		edge.From("tenant", Tenant.Type).Ref("sync_configs").Unique().Required(),
	}
}

func (EntitySyncConfig) Indexes() []ent.Index {
	return []ent.Index{
		index.Fields("tenant_id", "entity").Unique(),
	}
}
```

- [ ] **Step 8: Add shared time helper**

Create `internal/storage/ent/schema/helpers.go`:

```go
package schema

import "time"

func timeNow() time.Time {
	return time.Now().UTC()
}
```

- [ ] **Step 9: Add generate directive**

Create `internal/storage/ent/generate.go`:

```go
package ent

//go:generate go run -mod=mod entgo.io/ent/cmd/ent generate ./schema
```

- [ ] **Step 10: Generate Ent code**

```bash
go generate ./internal/storage/ent/...
```

Expected: Generated files appear in `internal/storage/ent/` (client.go, tenant.go, message.go, etc.)

- [ ] **Step 11: Verify compilation**

```bash
go build ./...
```

Expected: No errors.

- [ ] **Step 12: Commit**

```bash
git add .
git commit -m "feat: add Ent schemas for tenants, messages, notifications, sync"
```

---

### Task 4: Auth Middleware (odon JWT Validation)

**Files:**
- Create: `internal/middleware/context.go`
- Create: `internal/middleware/auth.go`
- Create: `internal/middleware/auth_test.go`

- [ ] **Step 1: Add JWT dependencies**

```bash
go get github.com/lestrrat-go/jwx/v2@latest
```

- [ ] **Step 2: Write context helpers**

Create `internal/middleware/context.go`:

```go
package middleware

import "context"

type contextKey string

const (
	ctxUserID   contextKey = "user_id"
	ctxEmail    contextKey = "email"
	ctxTenantID contextKey = "tenant_id"
)

func UserIDFromContext(ctx context.Context) string {
	v, _ := ctx.Value(ctxUserID).(string)
	return v
}

func EmailFromContext(ctx context.Context) string {
	v, _ := ctx.Value(ctxEmail).(string)
	return v
}

func TenantIDFromContext(ctx context.Context) string {
	v, _ := ctx.Value(ctxTenantID).(string)
	return v
}

func contextWithUser(ctx context.Context, userID, email string) context.Context {
	ctx = context.WithValue(ctx, ctxUserID, userID)
	ctx = context.WithValue(ctx, ctxEmail, email)
	return ctx
}

func contextWithTenant(ctx context.Context, tenantID string) context.Context {
	return context.WithValue(ctx, ctxTenantID, tenantID)
}
```

- [ ] **Step 3: Write failing auth middleware test**

Create `internal/middleware/auth_test.go`:

```go
package middleware_test

import (
	"crypto/rand"
	"crypto/rsa"
	"net/http"
	"net/http/httptest"
	"testing"
	"time"

	"github.com/knkCS/author-portal-backend/internal/middleware"
	"github.com/lestrrat-go/jwx/v2/jwa"
	"github.com/lestrrat-go/jwx/v2/jwk"
	"github.com/lestrrat-go/jwx/v2/jwt"
)

func generateTestJWKS(t *testing.T) (jwk.Key, jwk.Set) {
	t.Helper()
	privKey, err := rsa.GenerateKey(rand.Reader, 2048)
	if err != nil {
		t.Fatal(err)
	}
	key, err := jwk.FromRaw(privKey)
	if err != nil {
		t.Fatal(err)
	}
	_ = key.Set(jwk.KeyIDKey, "test-key-id")
	_ = key.Set(jwk.AlgorithmKey, jwa.RS256())

	pubKey, err := key.PublicKey()
	if err != nil {
		t.Fatal(err)
	}
	set := jwk.NewSet()
	_ = set.AddKey(pubKey)

	return key, set
}

func signToken(t *testing.T, key jwk.Key, claims map[string]any) string {
	t.Helper()
	builder := jwt.NewBuilder().
		Issuer("odon-test").
		IssuedAt(time.Now()).
		Expiration(time.Now().Add(time.Hour))
	for k, v := range claims {
		builder = builder.Claim(k, v)
	}
	tok, err := builder.Build()
	if err != nil {
		t.Fatal(err)
	}
	signed, err := jwt.Sign(tok, jwt.WithKey(jwa.RS256(), key))
	if err != nil {
		t.Fatal(err)
	}
	return string(signed)
}

func TestAuthMiddleware_ValidToken(t *testing.T) {
	privKey, pubSet := generateTestJWKS(t)

	token := signToken(t, privKey, map[string]any{
		"sub":   "user_123",
		"email": "author@example.com",
	})

	var capturedUserID, capturedEmail string
	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		capturedUserID = middleware.UserIDFromContext(r.Context())
		capturedEmail = middleware.EmailFromContext(r.Context())
		w.WriteHeader(http.StatusOK)
	})

	mw := middleware.NewAuthMiddleware(middleware.AuthMiddlewareConfig{
		KeySet: pubSet,
	})
	handler := mw.Wrap(inner)

	req := httptest.NewRequest("GET", "/", nil)
	req.Header.Set("Authorization", "Bearer "+token)
	rr := httptest.NewRecorder()
	handler.ServeHTTP(rr, req)

	if rr.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rr.Code)
	}
	if capturedUserID != "user_123" {
		t.Fatalf("expected user_123, got %q", capturedUserID)
	}
	if capturedEmail != "author@example.com" {
		t.Fatalf("expected author@example.com, got %q", capturedEmail)
	}
}

func TestAuthMiddleware_MissingToken(t *testing.T) {
	_, pubSet := generateTestJWKS(t)

	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	mw := middleware.NewAuthMiddleware(middleware.AuthMiddlewareConfig{
		KeySet: pubSet,
	})
	handler := mw.Wrap(inner)

	req := httptest.NewRequest("GET", "/", nil)
	rr := httptest.NewRecorder()
	handler.ServeHTTP(rr, req)

	if rr.Code != http.StatusUnauthorized {
		t.Fatalf("expected 401, got %d", rr.Code)
	}
}

func TestAuthMiddleware_ExpiredToken(t *testing.T) {
	privKey, pubSet := generateTestJWKS(t)

	builder := jwt.NewBuilder().
		Subject("user_123").
		Claim("email", "author@example.com").
		IssuedAt(time.Now().Add(-2 * time.Hour)).
		Expiration(time.Now().Add(-1 * time.Hour))
	tok, _ := builder.Build()
	signed, _ := jwt.Sign(tok, jwt.WithKey(jwa.RS256(), privKey))

	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	mw := middleware.NewAuthMiddleware(middleware.AuthMiddlewareConfig{
		KeySet: pubSet,
	})
	handler := mw.Wrap(inner)

	req := httptest.NewRequest("GET", "/", nil)
	req.Header.Set("Authorization", "Bearer "+string(signed))
	rr := httptest.NewRecorder()
	handler.ServeHTTP(rr, req)

	if rr.Code != http.StatusUnauthorized {
		t.Fatalf("expected 401, got %d", rr.Code)
	}
}
```

- [ ] **Step 4: Run tests to verify they fail**

```bash
go test ./internal/middleware/... -v
```

Expected: Compilation error — `middleware.NewAuthMiddleware` not defined.

- [ ] **Step 5: Implement auth middleware**

Create `internal/middleware/auth.go`:

```go
package middleware

import (
	"context"
	"log/slog"
	"net/http"
	"strings"
	"time"

	"github.com/lestrrat-go/jwx/v2/jwk"
	"github.com/lestrrat-go/jwx/v2/jws"
	"github.com/lestrrat-go/jwx/v2/jwt"
)

type AuthMiddlewareConfig struct {
	// KeySet is used directly in tests. In production, use JWKSURL instead.
	KeySet jwk.Set
	// JWKSURL is the odon JWKS endpoint for auto-refreshing keys.
	JWKSURL string
}

type AuthMiddleware struct {
	keySet jwk.Set
	cache  *jwk.Cache
}

func NewAuthMiddleware(cfg AuthMiddlewareConfig) *AuthMiddleware {
	m := &AuthMiddleware{}
	if cfg.KeySet != nil {
		m.keySet = cfg.KeySet
		return m
	}
	if cfg.JWKSURL != "" {
		cache := jwk.NewCache(context.Background())
		_ = cache.Register(cfg.JWKSURL, jwk.WithMinRefreshInterval(5*time.Minute))
		m.cache = cache
	}
	return m
}

func (m *AuthMiddleware) getKeySet(ctx context.Context) (jwk.Set, error) {
	if m.keySet != nil {
		return m.keySet, nil
	}
	if m.cache != nil {
		return m.cache.Lookup(ctx, "")
	}
	return nil, fmt.Errorf("no key set configured")
}

func (m *AuthMiddleware) Wrap(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		token := extractBearerToken(r)
		if token == "" {
			http.Error(w, "missing authorization token", http.StatusUnauthorized)
			return
		}

		keySet, err := m.getKeySet(r.Context())
		if err != nil {
			slog.Error("failed to get JWKS", "error", err)
			http.Error(w, "internal error", http.StatusInternalServerError)
			return
		}

		parsed, err := jwt.Parse(
			[]byte(token),
			jwt.WithKeySet(keySet, jws.WithInferAlgorithmFromKey(true)),
			jwt.WithValidate(true),
		)
		if err != nil {
			http.Error(w, "invalid token", http.StatusUnauthorized)
			return
		}

		userID, _ := parsed.Get("sub")
		email, _ := parsed.Get("email")

		ctx := contextWithUser(
			r.Context(),
			toString(userID),
			toString(email),
		)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func extractBearerToken(r *http.Request) string {
	auth := r.Header.Get("Authorization")
	if !strings.HasPrefix(auth, "Bearer ") {
		return ""
	}
	return strings.TrimPrefix(auth, "Bearer ")
}

func toString(v any) string {
	if s, ok := v.(string); ok {
		return s
	}
	return ""
}
```

Add missing import to auth.go:

```go
import (
	"context"
	"fmt"
	// ... rest of imports
)
```

- [ ] **Step 6: Handle JWKSURL in getKeySet**

The `cache.Lookup` needs the URL. Fix `getKeySet`:

```go
func (m *AuthMiddleware) getKeySet(ctx context.Context) (jwk.Set, error) {
	if m.keySet != nil {
		return m.keySet, nil
	}
	if m.cache != nil {
		return m.cache.Lookup(ctx, m.jwksURL)
	}
	return nil, fmt.Errorf("no key set configured")
}
```

Add `jwksURL` field to `AuthMiddleware` struct and set it in `NewAuthMiddleware`:

```go
type AuthMiddleware struct {
	keySet  jwk.Set
	cache   *jwk.Cache
	jwksURL string
}
```

In `NewAuthMiddleware`, set `m.jwksURL = cfg.JWKSURL` before the cache setup.

- [ ] **Step 7: Run tests to verify they pass**

```bash
go test ./internal/middleware/... -v -race
```

Expected: All 3 tests PASS.

- [ ] **Step 8: Commit**

```bash
git add internal/middleware/
git commit -m "feat: add JWT auth middleware with odon JWKS validation"
```

---

### Task 5: Tenant Resolution Middleware

**Files:**
- Create: `internal/middleware/tenant.go`
- Create: `internal/middleware/tenant_test.go`

- [ ] **Step 1: Write failing tenant middleware test**

Create `internal/middleware/tenant_test.go`:

```go
package middleware_test

import (
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/knkCS/author-portal-backend/internal/middleware"
)

type mockTenantResolver struct {
	tenants map[string]string // hostname → tenant ID
}

func (m *mockTenantResolver) ResolveByHost(host string) (string, error) {
	id, ok := m.tenants[host]
	if !ok {
		return "", fmt.Errorf("tenant not found for host %q", host)
	}
	return id, nil
}

func TestTenantMiddleware_ResolvesFromHost(t *testing.T) {
	resolver := &mockTenantResolver{
		tenants: map[string]string{
			"portal.acme-books.com": "tenant_acme",
		},
	}

	var capturedTenantID string
	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		capturedTenantID = middleware.TenantIDFromContext(r.Context())
		w.WriteHeader(http.StatusOK)
	})

	mw := middleware.NewTenantMiddleware(resolver)
	handler := mw.Wrap(inner)

	req := httptest.NewRequest("GET", "/", nil)
	req.Host = "portal.acme-books.com"
	rr := httptest.NewRecorder()
	handler.ServeHTTP(rr, req)

	if rr.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rr.Code)
	}
	if capturedTenantID != "tenant_acme" {
		t.Fatalf("expected tenant_acme, got %q", capturedTenantID)
	}
}

func TestTenantMiddleware_UnknownHost(t *testing.T) {
	resolver := &mockTenantResolver{tenants: map[string]string{}}

	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	mw := middleware.NewTenantMiddleware(resolver)
	handler := mw.Wrap(inner)

	req := httptest.NewRequest("GET", "/", nil)
	req.Host = "unknown.example.com"
	rr := httptest.NewRecorder()
	handler.ServeHTTP(rr, req)

	if rr.Code != http.StatusNotFound {
		t.Fatalf("expected 404, got %d", rr.Code)
	}
}

func TestTenantMiddleware_FallbackHeader(t *testing.T) {
	resolver := &mockTenantResolver{
		tenants: map[string]string{},
	}

	var capturedTenantID string
	inner := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		capturedTenantID = middleware.TenantIDFromContext(r.Context())
		w.WriteHeader(http.StatusOK)
	})

	mw := middleware.NewTenantMiddleware(resolver)
	handler := mw.Wrap(inner)

	req := httptest.NewRequest("GET", "/", nil)
	req.Host = "localhost:8090"
	req.Header.Set("X-Tenant-ID", "tenant_dev")
	rr := httptest.NewRecorder()
	handler.ServeHTTP(rr, req)

	if rr.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rr.Code)
	}
	if capturedTenantID != "tenant_dev" {
		t.Fatalf("expected tenant_dev, got %q", capturedTenantID)
	}
}
```

Add `"fmt"` to imports in the test file.

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/middleware/... -v -run TestTenant
```

Expected: Compilation error — `middleware.NewTenantMiddleware` not defined.

- [ ] **Step 3: Implement tenant middleware**

Create `internal/middleware/tenant.go`:

```go
package middleware

import (
	"net/http"
	"strings"
)

type TenantResolver interface {
	ResolveByHost(host string) (tenantID string, err error)
}

type TenantMiddleware struct {
	resolver TenantResolver
}

func NewTenantMiddleware(resolver TenantResolver) *TenantMiddleware {
	return &TenantMiddleware{resolver: resolver}
}

func (m *TenantMiddleware) Wrap(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Try X-Tenant-ID header first (for local development)
		if tenantID := r.Header.Get("X-Tenant-ID"); tenantID != "" {
			ctx := contextWithTenant(r.Context(), tenantID)
			next.ServeHTTP(w, r.WithContext(ctx))
			return
		}

		// Resolve from hostname
		host := stripPort(r.Host)
		tenantID, err := m.resolver.ResolveByHost(host)
		if err != nil {
			http.Error(w, "unknown tenant", http.StatusNotFound)
			return
		}

		ctx := contextWithTenant(r.Context(), tenantID)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func stripPort(host string) string {
	if i := strings.LastIndex(host, ":"); i != -1 {
		return host[:i]
	}
	return host
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/middleware/... -v -race
```

Expected: All 6 tests PASS (3 auth + 3 tenant).

- [ ] **Step 5: Commit**

```bash
git add internal/middleware/tenant.go internal/middleware/tenant_test.go
git commit -m "feat: add tenant resolution middleware with hostname and header fallback"
```

---

### Task 6: Guardian Permission Schema

**Files:**
- Create: `internal/guardian/schema.go`
- Create: `internal/guardian/schema_test.go`

- [ ] **Step 1: Add guardian-go dependency**

```bash
go get github.com/knkCS/guardian-go@latest
```

Note: If the module is not published to a registry, use a replace directive in go.mod:

```bash
go mod edit -replace github.com/knkCS/guardian-go=/Users/jeskoiwanovski/repo/guardian-go
```

- [ ] **Step 2: Write failing schema test**

Create `internal/guardian/schema_test.go`:

```go
package guardian_test

import (
	"testing"

	portalguardian "github.com/knkCS/author-portal-backend/internal/guardian"
)

func TestPortalSchema_HasExpectedTypes(t *testing.T) {
	schema := portalguardian.PortalSchema()

	expectedTypes := []string{"publisher", "imprint", "title", "manuscript"}
	for _, typ := range expectedTypes {
		found := false
		for _, def := range schema.ResourceTypes {
			if def.Name == typ {
				found = true
				break
			}
		}
		if !found {
			t.Errorf("expected resource type %q in schema", typ)
		}
	}
}

func TestPortalSchema_TitleHasAuthorRelation(t *testing.T) {
	schema := portalguardian.PortalSchema()

	for _, def := range schema.ResourceTypes {
		if def.Name == "title" {
			for _, rel := range def.Relations {
				if rel.Name == "author" {
					return // found
				}
			}
			t.Fatal("title resource type missing 'author' relation")
		}
	}
	t.Fatal("title resource type not found")
}

func TestPortalSchema_TitleAuthorImpliesCanView(t *testing.T) {
	schema := portalguardian.PortalSchema()

	for _, def := range schema.ResourceTypes {
		if def.Name == "title" {
			for _, rel := range def.Relations {
				if rel.Name == "can_view" {
					for _, implied := range rel.Union {
						if implied == "author" {
							return // found
						}
					}
					t.Fatal("can_view does not include author in union")
				}
			}
			t.Fatal("title missing can_view relation")
		}
	}
	t.Fatal("title resource type not found")
}
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
go test ./internal/guardian/... -v
```

Expected: Compilation error — `portalguardian.PortalSchema` not defined.

- [ ] **Step 4: Implement portal permission schema**

Create `internal/guardian/schema.go`:

```go
package guardian

import guardian "github.com/knkCS/guardian-go"

type Relation struct {
	Name  string
	Union []string // relations that imply this one
}

type ResourceType struct {
	Name      string
	Relations []Relation
}

type Schema struct {
	ResourceTypes []ResourceType
}

// PortalSchema defines the permission model for the author portal.
//
// Hierarchy: publisher → imprint → title → manuscript
//
// Key relationships:
//   - author on title: can view their own manuscripts, royalties, contracts, production status
//   - editor on title: can view/edit manuscripts, manage production, assign tasks
//   - admin on publisher: full access to all titles and settings
//   - production on title: can manage production workflow for a title
func PortalSchema() Schema {
	return Schema{
		ResourceTypes: []ResourceType{
			{
				Name: "publisher",
				Relations: []Relation{
					{Name: "admin"},
					{Name: "member"},
					{Name: "can_manage", Union: []string{"admin"}},
					{Name: "can_view", Union: []string{"admin", "member"}},
				},
			},
			{
				Name: "imprint",
				Relations: []Relation{
					{Name: "publisher"}, // parent reference
					{Name: "manager"},
					{Name: "member"},
					{Name: "can_manage", Union: []string{"manager"}},
					{Name: "can_view", Union: []string{"manager", "member"}},
				},
			},
			{
				Name: "title",
				Relations: []Relation{
					{Name: "imprint"}, // parent reference
					{Name: "author"},
					{Name: "editor"},
					{Name: "production"},
					{Name: "can_edit", Union: []string{"editor"}},
					{Name: "can_view", Union: []string{"author", "editor", "production"}},
					{Name: "can_approve", Union: []string{"author"}},
					{Name: "can_manage_production", Union: []string{"editor", "production"}},
				},
			},
			{
				Name: "manuscript",
				Relations: []Relation{
					{Name: "title"}, // parent reference
					{Name: "author"},
					{Name: "editor"},
					{Name: "can_edit", Union: []string{"author", "editor"}},
					{Name: "can_view", Union: []string{"author", "editor"}},
					{Name: "can_comment", Union: []string{"author", "editor"}},
				},
			},
		},
	}
}

// WriteToGuardian registers the portal schema with the guardian service.
func WriteToGuardian(client *guardian.Client, schema Schema) error {
	// Convert portal schema to guardian SchemaDefinition.
	// Implementation depends on exact guardian-go SDK types.
	// This will be wired up when guardian-go types are finalized.
	_ = client
	_ = schema
	return nil
}
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
go test ./internal/guardian/... -v -race
```

Expected: All 3 tests PASS.

- [ ] **Step 6: Commit**

```bash
git add internal/guardian/
git commit -m "feat: define guardian permission schema for author portal"
```

---

### Task 7: Health Check & Server Wiring

**Files:**
- Create: `internal/handler/health/handler.go`
- Create: `internal/server/server.go`
- Create: `internal/server/server_test.go`

- [ ] **Step 1: Write health handler**

Create `internal/handler/health/handler.go`:

```go
package health

import (
	"encoding/json"
	"net/http"
)

func Handler() http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
	})
}
```

- [ ] **Step 2: Write failing server test**

Create `internal/server/server_test.go`:

```go
package server_test

import (
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/knkCS/author-portal-backend/internal/config"
	"github.com/knkCS/author-portal-backend/internal/server"
)

func TestHealthEndpoint(t *testing.T) {
	cfg := &config.Config{
		Port:        8090,
		DatabaseURL: "postgres://portal:portal@localhost:5433/author_portal?sslmode=disable",
		OdonJWKSURL: "http://localhost:8080/.well-known/jwks.json",
		GuardianURL: "http://localhost:8081",
		CMSURL:      "http://localhost:8082",
	}

	srv, err := server.New(cfg)
	if err != nil {
		t.Fatal(err)
	}

	req := httptest.NewRequest("GET", "/healthz", nil)
	rr := httptest.NewRecorder()
	srv.Handler().ServeHTTP(rr, req)

	if rr.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rr.Code)
	}

	var body map[string]string
	if err := json.NewDecoder(rr.Body).Decode(&body); err != nil {
		t.Fatal(err)
	}
	if body["status"] != "ok" {
		t.Fatalf("expected status ok, got %q", body["status"])
	}
}
```

- [ ] **Step 3: Run test to verify it fails**

```bash
go test ./internal/server/... -v
```

Expected: Compilation error — `server.New` not defined.

- [ ] **Step 4: Implement server**

Create `internal/server/server.go`:

```go
package server

import (
	"net/http"

	"github.com/knkCS/author-portal-backend/internal/config"
	"github.com/knkCS/author-portal-backend/internal/handler/health"
	"github.com/knkCS/author-portal-backend/internal/middleware"
)

type Server struct {
	mux  *http.ServeMux
	auth *middleware.AuthMiddleware
}

func New(cfg *config.Config) (*Server, error) {
	auth := middleware.NewAuthMiddleware(middleware.AuthMiddlewareConfig{
		JWKSURL: cfg.OdonJWKSURL,
	})

	s := &Server{
		mux:  http.NewServeMux(),
		auth: auth,
	}
	s.routes()
	return s, nil
}

func (s *Server) Handler() http.Handler {
	return s.mux
}

func (s *Server) routes() {
	// Public endpoints (no auth required)
	s.mux.Handle("GET /healthz", health.Handler())

	// Authenticated endpoints will be added in subsequent tasks
	// s.mux.Handle("GET /api/v1/branding", s.auth.Wrap(...))
}
```

- [ ] **Step 5: Run test to verify it passes**

```bash
go test ./internal/server/... -v -race
```

Expected: PASS.

- [ ] **Step 6: Verify full build and test suite**

```bash
go build ./... && go test ./... -race -v
```

Expected: All tests pass, all packages build.

- [ ] **Step 7: Commit**

```bash
git add internal/handler/health/ internal/server/
git commit -m "feat: add health endpoint and server wiring"
```

---

### Task 8: Branding API

**Files:**
- Create: `internal/handler/branding/handler.go`
- Create: `internal/handler/branding/handler_test.go`
- Modify: `internal/server/server.go`

- [ ] **Step 1: Write failing branding handler test**

Create `internal/handler/branding/handler_test.go`:

```go
package branding_test

import (
	"bytes"
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/knkCS/author-portal-backend/internal/handler/branding"
)

type mockStore struct {
	data map[string]branding.Branding
}

func (m *mockStore) Get(ctx context.Context, tenantID string) (*branding.Branding, error) {
	b, ok := m.data[tenantID]
	if !ok {
		return &branding.Branding{}, nil
	}
	return &b, nil
}

func (m *mockStore) Update(ctx context.Context, tenantID string, b branding.Branding) error {
	m.data[tenantID] = b
	return nil
}

func TestGetBranding(t *testing.T) {
	store := &mockStore{
		data: map[string]branding.Branding{
			"tenant_acme": {
				Logo:         "https://acme.com/logo.png",
				PrimaryColor: "#1a2b3c",
				AccentColor:  "#ff5500",
				FontFamily:   "Inter",
				PublisherName: "Acme Books",
			},
		},
	}

	h := branding.NewHandler(store)

	req := httptest.NewRequest("GET", "/api/v1/branding", nil)
	req = req.WithContext(withTenantID(req.Context(), "tenant_acme"))
	rr := httptest.NewRecorder()
	h.Get(rr, req)

	if rr.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rr.Code)
	}

	var got branding.Branding
	json.NewDecoder(rr.Body).Decode(&got)
	if got.PublisherName != "Acme Books" {
		t.Fatalf("expected Acme Books, got %q", got.PublisherName)
	}
	if got.PrimaryColor != "#1a2b3c" {
		t.Fatalf("expected #1a2b3c, got %q", got.PrimaryColor)
	}
}

func TestUpdateBranding(t *testing.T) {
	store := &mockStore{data: map[string]branding.Branding{}}

	h := branding.NewHandler(store)

	body, _ := json.Marshal(branding.Branding{
		Logo:          "https://newpub.com/logo.svg",
		PrimaryColor:  "#000000",
		AccentColor:   "#ffffff",
		FontFamily:    "Georgia",
		PublisherName: "New Publisher",
	})

	req := httptest.NewRequest("PUT", "/api/v1/branding", bytes.NewReader(body))
	req.Header.Set("Content-Type", "application/json")
	req = req.WithContext(withTenantID(req.Context(), "tenant_new"))
	rr := httptest.NewRecorder()
	h.Update(rr, req)

	if rr.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rr.Code)
	}

	saved := store.data["tenant_new"]
	if saved.PublisherName != "New Publisher" {
		t.Fatalf("expected New Publisher, got %q", saved.PublisherName)
	}
}

// Test helper: inject tenant ID into context using the same key as middleware
type tenantKey string

func withTenantID(ctx context.Context, id string) context.Context {
	// Uses the middleware package's context function indirectly.
	// For tests, we inject via the same key.
	return context.WithValue(ctx, tenantKey("tenant_id"), id)
}
```

Note: The test needs access to the middleware context key. Update the test to import and use `middleware.TenantIDFromContext`. The handler should call `middleware.TenantIDFromContext(r.Context())`.

Replace the `withTenantID` helper:

```go
import "github.com/knkCS/author-portal-backend/internal/middleware"

// Use middleware's exported function to set context.
// Since middleware doesn't export the setter, we need a test-only approach.
// The handler will use middleware.TenantIDFromContext.
// For testing the handler in isolation, we pass tenant ID via a header
// and have the handler accept it both ways.
```

Simplify: the handler test should use a simpler approach. The handler reads tenant ID from context. For tests, we need middleware to be in the chain or use a test helper. Let's have the handler accept a `TenantIDFunc` or simply use the middleware context:

Replace the test to wrap requests through a minimal middleware:

```go
package branding_test

import (
	"bytes"
	"context"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"

	"github.com/knkCS/author-portal-backend/internal/handler/branding"
	"github.com/knkCS/author-portal-backend/internal/middleware"
)

type mockStore struct {
	data map[string]branding.Branding
}

func (m *mockStore) Get(_ context.Context, tenantID string) (*branding.Branding, error) {
	b, ok := m.data[tenantID]
	if !ok {
		return &branding.Branding{}, nil
	}
	return &b, nil
}

func (m *mockStore) Update(_ context.Context, tenantID string, b branding.Branding) error {
	m.data[tenantID] = b
	return nil
}

// wrapWithTenant adds X-Tenant-ID header so tenant middleware resolves it.
func addTenantHeader(req *http.Request, tenantID string) *http.Request {
	req.Header.Set("X-Tenant-ID", tenantID)
	return req
}

func TestGetBranding(t *testing.T) {
	store := &mockStore{
		data: map[string]branding.Branding{
			"tenant_acme": {
				Logo:          "https://acme.com/logo.png",
				PrimaryColor:  "#1a2b3c",
				AccentColor:   "#ff5500",
				FontFamily:    "Inter",
				PublisherName: "Acme Books",
			},
		},
	}

	h := branding.NewHandler(store)
	tenantMW := middleware.NewTenantMiddleware(&headerOnlyResolver{})
	handler := tenantMW.Wrap(http.HandlerFunc(h.Get))

	req := addTenantHeader(httptest.NewRequest("GET", "/api/v1/branding", nil), "tenant_acme")
	rr := httptest.NewRecorder()
	handler.ServeHTTP(rr, req)

	if rr.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rr.Code)
	}

	var got branding.Branding
	json.NewDecoder(rr.Body).Decode(&got)
	if got.PublisherName != "Acme Books" {
		t.Fatalf("expected Acme Books, got %q", got.PublisherName)
	}
}

func TestUpdateBranding(t *testing.T) {
	store := &mockStore{data: map[string]branding.Branding{}}

	h := branding.NewHandler(store)
	tenantMW := middleware.NewTenantMiddleware(&headerOnlyResolver{})
	handler := tenantMW.Wrap(http.HandlerFunc(h.Update))

	body, _ := json.Marshal(branding.Branding{
		Logo:          "https://newpub.com/logo.svg",
		PrimaryColor:  "#000000",
		AccentColor:   "#ffffff",
		FontFamily:    "Georgia",
		PublisherName: "New Publisher",
	})

	req := addTenantHeader(
		httptest.NewRequest("PUT", "/api/v1/branding", bytes.NewReader(body)),
		"tenant_new",
	)
	req.Header.Set("Content-Type", "application/json")
	rr := httptest.NewRecorder()
	handler.ServeHTTP(rr, req)

	if rr.Code != http.StatusOK {
		t.Fatalf("expected 200, got %d", rr.Code)
	}

	saved := store.data["tenant_new"]
	if saved.PublisherName != "New Publisher" {
		t.Fatalf("expected New Publisher, got %q", saved.PublisherName)
	}
}

// headerOnlyResolver always fails hostname lookup, forcing X-Tenant-ID header fallback.
type headerOnlyResolver struct{}

func (h *headerOnlyResolver) ResolveByHost(host string) (string, error) {
	return "", fmt.Errorf("not found")
}
```

Add `"fmt"` to imports.

- [ ] **Step 2: Run tests to verify they fail**

```bash
go test ./internal/handler/branding/... -v
```

Expected: Compilation error — `branding.NewHandler` not defined.

- [ ] **Step 3: Implement branding handler**

Create `internal/handler/branding/handler.go`:

```go
package branding

import (
	"context"
	"encoding/json"
	"net/http"

	"github.com/knkCS/author-portal-backend/internal/middleware"
)

type Branding struct {
	Logo          string `json:"logo"`
	PrimaryColor  string `json:"primaryColor"`
	AccentColor   string `json:"accentColor"`
	FontFamily    string `json:"fontFamily"`
	PublisherName string `json:"publisherName"`
}

type Store interface {
	Get(ctx context.Context, tenantID string) (*Branding, error)
	Update(ctx context.Context, tenantID string, b Branding) error
}

type Handler struct {
	store Store
}

func NewHandler(store Store) *Handler {
	return &Handler{store: store}
}

func (h *Handler) Get(w http.ResponseWriter, r *http.Request) {
	tenantID := middleware.TenantIDFromContext(r.Context())
	if tenantID == "" {
		http.Error(w, "tenant not resolved", http.StatusBadRequest)
		return
	}

	branding, err := h.store.Get(r.Context(), tenantID)
	if err != nil {
		http.Error(w, "failed to get branding", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(branding)
}

func (h *Handler) Update(w http.ResponseWriter, r *http.Request) {
	tenantID := middleware.TenantIDFromContext(r.Context())
	if tenantID == "" {
		http.Error(w, "tenant not resolved", http.StatusBadRequest)
		return
	}

	var b Branding
	if err := json.NewDecoder(r.Body).Decode(&b); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	if err := h.store.Update(r.Context(), tenantID, b); err != nil {
		http.Error(w, "failed to update branding", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(b)
}
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
go test ./internal/handler/branding/... -v -race
```

Expected: All 2 tests PASS.

- [ ] **Step 5: Wire branding routes into server**

Modify `internal/server/server.go` — add branding routes:

```go
package server

import (
	"net/http"

	"github.com/knkCS/author-portal-backend/internal/config"
	"github.com/knkCS/author-portal-backend/internal/handler/branding"
	"github.com/knkCS/author-portal-backend/internal/handler/health"
	"github.com/knkCS/author-portal-backend/internal/middleware"
)

type Server struct {
	mux     *http.ServeMux
	auth    *middleware.AuthMiddleware
	tenant  *middleware.TenantMiddleware
	branding *branding.Handler
}

func New(cfg *config.Config, opts ...Option) (*Server, error) {
	auth := middleware.NewAuthMiddleware(middleware.AuthMiddlewareConfig{
		JWKSURL: cfg.OdonJWKSURL,
	})

	s := &Server{
		mux:  http.NewServeMux(),
		auth: auth,
	}

	for _, opt := range opts {
		opt(s)
	}

	s.routes()
	return s, nil
}

type Option func(*Server)

func WithTenantResolver(resolver middleware.TenantResolver) Option {
	return func(s *Server) {
		s.tenant = middleware.NewTenantMiddleware(resolver)
	}
}

func WithBrandingStore(store branding.Store) Option {
	return func(s *Server) {
		s.branding = branding.NewHandler(store)
	}
}

func (s *Server) Handler() http.Handler {
	return s.mux
}

func (s *Server) routes() {
	// Public
	s.mux.Handle("GET /healthz", health.Handler())

	// Tenant-scoped, public (branding is loaded before auth for the login page)
	if s.branding != nil && s.tenant != nil {
		s.mux.Handle("GET /api/v1/branding",
			s.tenant.Wrap(http.HandlerFunc(s.branding.Get)))
	}

	// Tenant-scoped, authenticated
	if s.branding != nil && s.tenant != nil {
		s.mux.Handle("PUT /api/v1/branding",
			s.tenant.Wrap(s.auth.Wrap(http.HandlerFunc(s.branding.Update))))
	}
}
```

- [ ] **Step 6: Run full test suite**

```bash
go test ./... -race -v
```

Expected: All tests pass.

- [ ] **Step 7: Commit**

```bash
git add internal/handler/branding/ internal/server/
git commit -m "feat: add branding API with get and update endpoints"
```

---

### Task 9: Tenant Database Store (Wire Ent to Branding)

**Files:**
- Create: `internal/storage/tenant_store.go`
- Create: `internal/storage/tenant_store_test.go`

- [ ] **Step 1: Write failing store test**

Create `internal/storage/tenant_store_test.go`:

```go
package storage_test

import (
	"context"
	"testing"

	"github.com/knkCS/author-portal-backend/internal/handler/branding"
	"github.com/knkCS/author-portal-backend/internal/storage"

	_ "github.com/mattn/go-sqlite3" // test DB
)

func TestTenantStore_GetAndUpdateBranding(t *testing.T) {
	// Use an in-memory Ent client for testing.
	// This requires the generated Ent client with SQLite support.
	// For now, test with a mock that validates the interface.
	var _ branding.Store = (*storage.TenantStore)(nil)
}
```

This is a compile-time interface check. Full integration tests require the Ent client with a test database — we'll add those once the Ent client is generated and wired.

- [ ] **Step 2: Implement tenant store**

Create `internal/storage/tenant_store.go`:

```go
package storage

import (
	"context"
	"encoding/json"

	"github.com/knkCS/author-portal-backend/internal/handler/branding"
	"github.com/knkCS/author-portal-backend/internal/storage/ent"
	"github.com/knkCS/author-portal-backend/internal/storage/ent/tenant"
)

type TenantStore struct {
	client *ent.Client
}

func NewTenantStore(client *ent.Client) *TenantStore {
	return &TenantStore{client: client}
}

// Get retrieves branding for a tenant. Returns empty branding if tenant not found.
func (s *TenantStore) Get(ctx context.Context, tenantID string) (*branding.Branding, error) {
	t, err := s.client.Tenant.Query().
		Where(tenant.OdonOrgID(tenantID)).
		Only(ctx)
	if err != nil {
		return &branding.Branding{}, nil
	}

	b := &branding.Branding{}
	if t.Branding != nil {
		raw, _ := json.Marshal(t.Branding)
		json.Unmarshal(raw, b)
	}
	return b, nil
}

// Update saves branding for a tenant.
func (s *TenantStore) Update(ctx context.Context, tenantID string, b branding.Branding) error {
	brandingMap := map[string]any{
		"logo":          b.Logo,
		"primaryColor":  b.PrimaryColor,
		"accentColor":   b.AccentColor,
		"fontFamily":    b.FontFamily,
		"publisherName": b.PublisherName,
	}

	return s.client.Tenant.Update().
		Where(tenant.OdonOrgID(tenantID)).
		SetBranding(brandingMap).
		Exec(ctx)
}

// ResolveByHost implements middleware.TenantResolver.
func (s *TenantStore) ResolveByHost(ctx context.Context, host string) (string, error) {
	t, err := s.client.Tenant.Query().
		Where(tenant.CustomDomain(host)).
		Only(ctx)
	if err != nil {
		// Try slug-based subdomain: {slug}.portal.example.com
		return "", err
	}
	return t.OdonOrgID, nil
}
```

- [ ] **Step 3: Verify compilation**

```bash
go build ./...
```

Note: This may fail if Ent hasn't been generated yet. Run `go generate ./internal/storage/ent/...` first if needed.

- [ ] **Step 4: Commit**

```bash
git add internal/storage/tenant_store.go internal/storage/tenant_store_test.go
git commit -m "feat: add tenant store bridging Ent schema to branding API"
```

---

### Task 10: Integration Smoke Test

**Files:**
- Modify: `cmd/api/main.go` — wire store and server together

- [ ] **Step 1: Update main.go to wire everything**

Update `cmd/api/main.go` to connect to the database and wire stores:

```go
package main

import (
	"context"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/knkCS/author-portal-backend/internal/config"
	"github.com/knkCS/author-portal-backend/internal/server"
	"github.com/knkCS/author-portal-backend/internal/storage"
	"github.com/knkCS/author-portal-backend/internal/storage/ent"

	_ "github.com/lib/pq"
)

func main() {
	cfg, err := config.Load()
	if err != nil {
		slog.Error("failed to load config", "error", err)
		os.Exit(1)
	}

	client, err := ent.Open("postgres", cfg.DatabaseURL)
	if err != nil {
		slog.Error("failed to connect to database", "error", err)
		os.Exit(1)
	}
	defer client.Close()

	if err := client.Schema.Create(context.Background()); err != nil {
		slog.Error("failed to run migrations", "error", err)
		os.Exit(1)
	}

	tenantStore := storage.NewTenantStore(client)

	srv, err := server.New(cfg,
		server.WithTenantResolver(tenantStore),
		server.WithBrandingStore(tenantStore),
	)
	if err != nil {
		slog.Error("failed to create server", "error", err)
		os.Exit(1)
	}

	httpServer := &http.Server{
		Addr:         fmt.Sprintf(":%d", cfg.Port),
		Handler:      srv.Handler(),
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
	}

	go func() {
		slog.Info("starting server", "port", cfg.Port)
		if err := httpServer.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			slog.Error("server error", "error", err)
			os.Exit(1)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := httpServer.Shutdown(ctx); err != nil {
		slog.Error("shutdown error", "error", err)
	}
	slog.Info("server stopped")
}
```

- [ ] **Step 2: Add postgres driver dependency**

```bash
go get github.com/lib/pq@latest
```

- [ ] **Step 3: Update TenantStore.ResolveByHost to match interface**

The `middleware.TenantResolver` interface has `ResolveByHost(host string)` without context. Update `TenantStore`:

```go
func (s *TenantStore) ResolveByHost(host string) (string, error) {
	t, err := s.client.Tenant.Query().
		Where(tenant.CustomDomain(host)).
		Only(context.Background())
	if err != nil {
		return "", err
	}
	return t.OdonOrgID, nil
}
```

- [ ] **Step 4: Verify full build**

```bash
go build ./...
```

Expected: Compiles cleanly.

- [ ] **Step 5: Smoke test with Docker Compose**

```bash
docker compose up -d postgres redis
DATABASE_URL="postgres://portal:portal@localhost:5433/author_portal?sslmode=disable" go run ./cmd/api &
sleep 2
curl -s http://localhost:8090/healthz
kill %1
```

Expected: `{"status":"ok"}`

- [ ] **Step 6: Commit**

```bash
git add cmd/api/main.go
git commit -m "feat: wire database, tenant store, and branding into API server"
```

- [ ] **Step 7: Push to GitHub**

```bash
gh repo create knkCS/author-portal-backend --private --source=. --push --description "Author portal API gateway and sync service"
```

---

## Summary

After completing all 10 tasks, you have:

- **Go project** with two entrypoints (`cmd/api`, `cmd/sync`)
- **6 Ent schemas** (tenant, message, notification, notification_preference, sync_job, entity_sync_config)
- **JWT auth middleware** validating odon tokens via JWKS
- **Tenant resolution middleware** from hostname or header
- **Guardian permission schema** defining publisher → imprint → title → manuscript hierarchy
- **Branding API** (GET public, PUT authenticated) backed by PostgreSQL
- **Health endpoint** at `/healthz`
- **Docker Compose** with PostgreSQL and Redis
- **Tests** for auth, tenant resolution, branding, and server wiring

**Next plan:** Plan 2 — Frontend Shell & Auth Flow (React + Vite + anker + odon OAuth2 PKCE)
