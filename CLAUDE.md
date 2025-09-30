## casboard: Single Static Binary Implementation - CLAUDE.md

# casboard - Complete Single Binary Project Specification v1.0.0

This document contains the complete specification for building casboard as a single static binary with everything embedded. All assets, templates, and migrations are compiled into the binary.

## Project Overview

**Name:** casboard  
**Organization:** casapps  
**License:** MIT  
**Version:** 1.0.0  
**Binary:** Single static executable with embedded resources  
**Port:** Auto-assigned from 64000-64999 range if not specified  

## Directory Structure

```
casboard/
├── cmd/
│   └── casboard/
│       └── main.go                 # Single entry point
├── internal/
│   ├── app/
│   │   └── app.go                  # Main application
│   ├── config/
│   │   └── config.go               # Configuration (DB-based)
│   ├── database/
│   │   ├── migrations.go           # Embedded migrations
│   │   ├── connection.go
│   │   └── queries.go
│   ├── handlers/
│   │   ├── api.go
│   │   ├── board.go
│   │   ├── post.go
│   │   ├── admin.go
│   │   ├── setup.go
│   │   └── websocket.go
│   ├── middleware/
│   │   └── middleware.go           # All middleware
│   ├── models/
│   │   └── models.go               # All models
│   ├── services/
│   │   └── services.go             # All services
│   ├── scheduler/
│   │   └── scheduler.go            # Built-in scheduler
│   ├── templates/
│   │   └── templates.go            # Embedded templates
│   ├── static/
│   │   └── static.go               # Embedded static files
│   └── embed/
│       └── embed.go                # Resource embedding
├── web/
│   ├── templates/                  # HTML templates (embedded)
│   │   ├── base.tmpl
│   │   ├── setup/
│   │   ├── board/
│   │   └── admin/
│   ├── static/                     # Static files (embedded)
│   │   ├── css/
│   │   ├── js/
│   │   └── images/
│   └── migrations/                  # SQL migrations (embedded)
│       ├── 001_initial.sql
│       ├── 002_boards.sql
│       └── ...
├── Makefile
├── go.mod
├── go.sum
├── README.md
├── LICENSE.md
└── Jenkinsfile
```

## Single Binary Implementation

### Main Entry Point

```go
// cmd/casboard/main.go
package main

import (
    "context"
    "database/sql"
    "embed"
    "flag"
    "fmt"
    "log"
    "math/rand"
    "net"
    "net/http"
    "os"
    "os/signal"
    "path/filepath"
    "runtime"
    "syscall"
    "time"
    
    _ "github.com/lib/pq"
    _ "modernc.org/sqlite" // SQLite for embedded option
    
    "casboard/internal/app"
    "casboard/internal/embed"
)

var (
    version = "1.0.0"
    build   = "unknown"
    
    //go:embed all:web/templates/*
    templateFS embed.FS
    
    //go:embed all:web/static/*
    staticFS embed.FS
    
    //go:embed all:web/migrations/*.sql
    migrationFS embed.FS
)

func main() {
    var (
        dataDir    = flag.String("data", defaultDataDir(), "Data directory")
        dbType     = flag.String("db", "auto", "Database type: postgres, sqlite, auto")
        dbDSN      = flag.String("dsn", "", "Database DSN (overrides auto-detection)")
        portFlag   = flag.Int("port", 0, "Server port (0 = auto-assign from 64000-64999)")
        migrate    = flag.Bool("migrate", false, "Run database migrations")
        resetPort  = flag.Bool("reset-port", false, "Reset to new random port")
        showVer    = flag.Bool("version", false, "Show version")
        help       = flag.Bool("help", false, "Show help")
    )
    flag.Parse()
    
    if *help {
        printHelp()
        os.Exit(0)
    }
    
    if *showVer {
        fmt.Printf("casboard v%s (build: %s, %s/%s)\n", version, build, runtime.GOOS, runtime.GOARCH)
        os.Exit(0)
    }
    
    // Ensure data directory exists
    if err := os.MkdirAll(*dataDir, 0755); err != nil {
        log.Fatal("Failed to create data directory:", err)
    }
    
    // Initialize embedded resources
    resources := &embed.Resources{
        Templates:  templateFS,
        Static:     staticFS,
        Migrations: migrationFS,
    }
    
    // Setup database
    db, err := setupDatabase(*dataDir, *dbType, *dbDSN)
    if err != nil {
        log.Fatal("Database setup failed:", err)
    }
    defer db.Close()
    
    // Run migrations
    if *migrate {
        if err := runEmbeddedMigrations(db, resources); err != nil {
            log.Fatal("Migration failed:", err)
        }
        log.Println("Migrations completed successfully")
        if !needsInitialSetup(db) {
            os.Exit(0)
        }
    }
    
    // Get or assign port
    port := getServerPort(db, *portFlag, *resetPort)
    
    // Check if initial setup is needed
    if needsInitialSetup(db) {
        log.Printf("Starting setup wizard on port %d...\n", port)
        runSetupWizard(db, port, resources)
        return
    }
    
    // Create application
    application, err := app.New(db, resources, port)
    if err != nil {
        log.Fatal("Failed to create application:", err)
    }
    
    // Start server
    srv := &http.Server{
        Addr:         fmt.Sprintf(":%d", port),
        Handler:      application.Router(),
        ReadTimeout:  15 * time.Second,
        WriteTimeout: 15 * time.Second,
        IdleTimeout:  60 * time.Second,
    }
    
    // Start built-in services
    go application.StartScheduler()
    go application.StartWebSocket()
    
    // Graceful shutdown
    go func() {
        sigint := make(chan os.Signal, 1)
        signal.Notify(sigint, os.Interrupt, syscall.SIGTERM)
        <-sigint
        
        log.Println("Shutting down...")
        ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
        defer cancel()
        
        application.Shutdown(ctx)
        srv.Shutdown(ctx)
    }()
    
    log.Printf("casboard v%s started on port %d", version, port)
    log.Printf("Access at http://localhost:%d", port)
    
    if err := srv.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatal("Server failed:", err)
    }
}

func defaultDataDir() string {
    switch runtime.GOOS {
    case "windows":
        return filepath.Join(os.Getenv("APPDATA"), "casboard")
    case "darwin":
        home, _ := os.UserHomeDir()
        return filepath.Join(home, "Library", "Application Support", "casboard")
    default:
        // Linux/Unix
        if xdgData := os.Getenv("XDG_DATA_HOME"); xdgData != "" {
            return filepath.Join(xdgData, "casboard")
        }
        home, _ := os.UserHomeDir()
        return filepath.Join(home, ".local", "share", "casboard")
    }
}

func setupDatabase(dataDir, dbType, dsn string) (*sql.DB, error) {
    if dsn != "" {
        // Use provided DSN
        return sql.Open(detectDBType(dsn), dsn)
    }
    
    switch dbType {
    case "sqlite", "auto":
        // Use embedded SQLite for simplicity
        dbPath := filepath.Join(dataDir, "casboard.db")
        return sql.Open("sqlite", dbPath+"?_pragma=foreign_keys(1)&_pragma=journal_mode(WAL)")
        
    case "postgres":
        // Try to connect to PostgreSQL
        dsn = fmt.Sprintf("host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
            getEnv("DB_HOST", "localhost"),
            getEnvInt("DB_PORT", 5432),
            getEnv("DB_USER", "casboard"),
            getEnv("DB_PASSWORD", ""),
            getEnv("DB_NAME", "casboard"),
            getEnv("DB_SSLMODE", "prefer"),
        )
        return sql.Open("postgres", dsn)
        
    default:
        return nil, fmt.Errorf("unsupported database type: %s", dbType)
    }
}

func getServerPort(db *sql.DB, requestedPort int, resetPort bool) int {
    // If specific port requested, use it
    if requestedPort > 0 && !resetPort {
        savePortToDatabase(db, requestedPort)
        return requestedPort
    }
    
    // Check if port is saved in database
    if !resetPort {
        var savedPort int
        err := db.QueryRow(`
            SELECT value::int FROM system_settings WHERE key = 'server_port'
        `).Scan(&savedPort)
        
        if err == nil && savedPort > 0 {
            // Verify port is available
            if isPortAvailable(savedPort) {
                return savedPort
            }
        }
    }
    
    // Find random available port in range 64000-64999
    port := findAvailablePort()
    savePortToDatabase(db, port)
    return port
}

func findAvailablePort() int {
    rand.Seed(time.Now().UnixNano())
    
    for attempts := 0; attempts < 100; attempts++ {
        port := 64000 + rand.Intn(1000)
        if isPortAvailable(port) {
            return port
        }
    }
    
    // Fallback: let OS assign
    listener, err := net.Listen("tcp", ":0")
    if err != nil {
        return 8080 // Last resort default
    }
    defer listener.Close()
    return listener.Addr().(*net.TCPAddr).Port
}

func isPortAvailable(port int) bool {
    listener, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
    if err != nil {
        return false
    }
    listener.Close()
    return true
}

func savePortToDatabase(db *sql.DB, port int) {
    db.Exec(`
        INSERT INTO system_settings (key, value, type, category)
        VALUES ('server_port', $1, 'integer', 'server')
        ON CONFLICT (key) DO UPDATE SET value = EXCLUDED.value
    `, fmt.Sprintf("%d", port))
}

func needsInitialSetup(db *sql.DB) bool {
    var initialized bool
    err := db.QueryRow(`
        SELECT COALESCE(
            (SELECT value::boolean FROM system_settings WHERE key = 'initialized'),
            false
        )
    `).Scan(&initialized)
    
    return err != nil || !initialized
}

func runEmbeddedMigrations(db *sql.DB, resources *embed.Resources) error {
    // Create migrations table if not exists
    _, err := db.Exec(`
        CREATE TABLE IF NOT EXISTS schema_migrations (
            version INTEGER PRIMARY KEY,
            applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    `)
    if err != nil {
        return err
    }
    
    // Read all migration files
    entries, err := migrationFS.ReadDir("web/migrations")
    if err != nil {
        return err
    }
    
    for _, entry := range entries {
        if entry.IsDir() {
            continue
        }
        
        // Parse version from filename (e.g., 001_initial.sql)
        var version int
        fmt.Sscanf(entry.Name(), "%03d_", &version)
        
        // Check if already applied
        var applied bool
        db.QueryRow("SELECT EXISTS(SELECT 1 FROM schema_migrations WHERE version = $1)", version).Scan(&applied)
        if applied {
            continue
        }
        
        // Read and execute migration
        content, err := migrationFS.ReadFile(filepath.Join("web/migrations", entry.Name()))
        if err != nil {
            return err
        }
        
        // Execute migration
        if _, err := db.Exec(string(content)); err != nil {
            return fmt.Errorf("migration %s failed: %w", entry.Name(), err)
        }
        
        // Record migration
        db.Exec("INSERT INTO schema_migrations (version) VALUES ($1)", version)
        log.Printf("Applied migration: %s", entry.Name())
    }
    
    return nil
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
    if value := os.Getenv(key); value != "" {
        var i int
        fmt.Sscanf(value, "%d", &i)
        if i > 0 {
            return i
        }
    }
    return defaultValue
}

func printHelp() {
    fmt.Printf(`casboard v%s - Modern Anonymous Image Board

Usage: casboard [options]

Options:
  -data string     Data directory (default: %s)
  -db string       Database type: postgres, sqlite, auto (default: auto)
  -dsn string      Database DSN (overrides auto-detection)
  -port int        Server port, 0 = auto-assign from 64000-64999 (default: 0)
  -migrate         Run database migrations
  -reset-port      Reset to new random port
  -version         Show version information
  -help            Show this help message

Environment Variables:
  DB_HOST          PostgreSQL host (default: localhost)
  DB_PORT          PostgreSQL port (default: 5432)
  DB_USER          PostgreSQL user (default: casboard)
  DB_PASSWORD      PostgreSQL password
  DB_NAME          PostgreSQL database (default: casboard)

Examples:
  # First run (auto-setup with SQLite)
  casboard

  # Use PostgreSQL
  casboard -db postgres

  # Specify port
  casboard -port 8080

  # Run migrations
  casboard -migrate

`, version, defaultDataDir())
}
```

### Embedded Resources

```go
// internal/embed/embed.go
package embed

import (
    "embed"
    "html/template"
    "io/fs"
    "net/http"
    "path/filepath"
    "strings"
)

type Resources struct {
    Templates  embed.FS
    Static     embed.FS
    Migrations embed.FS
}

// GetTemplate parses and returns templates
func (r *Resources) GetTemplate(patterns ...string) (*template.Template, error) {
    tmpl := template.New("").Funcs(templateFuncs())
    
    for _, pattern := range patterns {
        err := fs.WalkDir(r.Templates, "web/templates", func(path string, d fs.DirEntry, err error) error {
            if err != nil {
                return err
            }
            
            if d.IsDir() || !strings.HasSuffix(path, ".tmpl") {
                return nil
            }
            
            // Check if path matches pattern
            matched, _ := filepath.Match(pattern, path)
            if !matched && pattern != "*" {
                return nil
            }
            
            content, err := r.Templates.ReadFile(path)
            if err != nil {
                return err
            }
            
            name := strings.TrimPrefix(path, "web/templates/")
            _, err = tmpl.New(name).Parse(string(content))
            return err
        })
        
        if err != nil {
            return nil, err
        }
    }
    
    return tmpl, nil
}

// GetStaticHandler returns HTTP handler for static files
func (r *Resources) GetStaticHandler() http.Handler {
    sub, _ := fs.Sub(r.Static, "web/static")
    return http.FileServer(http.FS(sub))
}

// GetMigration reads a migration file
func (r *Resources) GetMigration(name string) (string, error) {
    content, err := r.Migrations.ReadFile(filepath.Join("web/migrations", name))
    if err != nil {
        return "", err
    }
    return string(content), nil
}

func templateFuncs() template.FuncMap {
    return template.FuncMap{
        "dict": func(values ...interface{}) (map[string]interface{}, error) {
            if len(values)%2 != 0 {
                return nil, fmt.Errorf("dict requires even number of arguments")
            }
            dict := make(map[string]interface{})
            for i := 0; i < len(values); i += 2 {
                key, ok := values[i].(string)
                if !ok {
                    return nil, fmt.Errorf("dict keys must be strings")
                }
                dict[key] = values[i+1]
            }
            return dict, nil
        },
        "safe": func(s string) template.HTML {
            return template.HTML(s)
        },
        "json": func(v interface{}) string {
            b, _ := json.Marshal(v)
            return string(b)
        },
        "now": func() time.Time {
            return time.Now()
        },
    }
}
```

### Application Core

```go
// internal/app/app.go
package app

import (
    "context"
    "database/sql"
    "fmt"
    "net/http"
    "sync"
    "time"
    
    "github.com/gorilla/mux"
    "github.com/gorilla/sessions"
    
    "casboard/internal/embed"
    "casboard/internal/handlers"
    "casboard/internal/middleware"
    "casboard/internal/scheduler"
    "casboard/internal/services"
)

type Application struct {
    db         *sql.DB
    router     *mux.Router
    resources  *embed.Resources
    services   *services.Container
    scheduler  *scheduler.Scheduler
    wsHub      *services.WebSocketHub
    store      sessions.Store
    port       int
    mu         sync.RWMutex
    shutdown   chan struct{}
}

func New(db *sql.DB, resources *embed.Resources, port int) (*Application, error) {
    app := &Application{
        db:        db,
        resources: resources,
        port:      port,
        shutdown:  make(chan struct{}),
    }
    
    // Initialize session store (uses database)
    app.store = NewDatabaseStore(db, []byte(app.getSessionKey()))
    
    // Initialize services
    app.services = &services.Container{
        DB:         db,
        Auth:       services.NewAuthService(db),
        Email:      services.NewEmailService(db),
        File:       services.NewFileService(db, app.getStoragePath()),
        Cache:      services.NewCacheService(db), // DB-based cache
        Spam:       services.NewSpamService(db),
        Moderation: services.NewModerationService(db),
    }
    
    // Initialize scheduler
    app.scheduler = scheduler.New(db, app.services)
    
    // Initialize WebSocket hub
    app.wsHub = services.NewWebSocketHub()
    
    // Setup routes
    app.setupRoutes()
    
    return app, nil
}

func (app *Application) setupRoutes() {
    r := mux.NewRouter()
    
    // Initialize middleware
    mw := middleware.New(app.db, app.store)
    
    // Global middleware
    r.Use(mw.Logger)
    r.Use(mw.Recovery) 
    r.Use(mw.ProxyHeaders)
    r.Use(mw.RateLimit)
    r.Use(mw.Session)
    
    // Initialize handlers
    h := handlers.New(app.db, app.resources, app.services, app.wsHub)
    
    // Static files (embedded)
    r.PathPrefix("/static/").Handler(
        http.StripPrefix("/static/", app.resources.GetStaticHandler()),
    )
    
    // Public routes
    r.HandleFunc("/", h.Home).Methods("GET")
    r.HandleFunc("/health", h.Health).Methods("GET")
    
    // Board routes
    r.HandleFunc("/{board:[a-z0-9]+}/", h.Board).Methods("GET")
    r.HandleFunc("/{board:[a-z0-9]+}/catalog", h.Catalog).Methods("GET")
    r.HandleFunc("/{board:[a-z0-9]+}/archive", h.Archive).Methods("GET")
    r.HandleFunc("/{board:[a-z0-9]+}/thread/{id:[0-9]+}", h.Thread).Methods("GET")
    
    // Posting routes (with CSRF)
    post := r.PathPrefix("").Subrouter()
    post.Use(mw.CSRF)
    post.HandleFunc("/{board:[a-z0-9]+}/post", h.CreateThread).Methods("POST")
    post.HandleFunc("/{board:[a-z0-9]+}/thread/{id:[0-9]+}/reply", h.CreateReply).Methods("POST")
    
    // API routes
    api := r.PathPrefix("/api/v1").Subrouter()
    api.Use(mw.JSON)
    
    api.HandleFunc("/boards", h.API.ListBoards).Methods("GET")
    api.HandleFunc("/boards/{board}", h.API.GetBoard).Methods("GET")
    api.HandleFunc("/boards/{board}/threads", h.API.ListThreads).Methods("GET")
    api.HandleFunc("/threads/{id}", h.API.GetThread).Methods("GET")
    api.HandleFunc("/posts/{id}", h.API.GetPost).Methods("GET")
    api.HandleFunc("/posts/search", h.API.SearchPosts).Methods("GET")
    
    // WebSocket
    api.HandleFunc("/ws", h.WebSocket.Handle)
    
    // Admin panel
    admin := r.PathPrefix("/admin").Subrouter()
    admin.Use(mw.RequireAdmin)
    admin.HandleFunc("/", h.Admin.Dashboard).Methods("GET")
    admin.HandleFunc("/boards", h.Admin.Boards).Methods("GET", "POST")
    admin.HandleFunc("/users", h.Admin.Users).Methods("GET")
    admin.HandleFunc("/settings", h.Admin.Settings).Methods("GET", "POST")
    
    // File serving (from data directory)
    r.HandleFunc("/src/{file}", h.ServeFile).Methods("GET")
    r.HandleFunc("/thumb/{file}", h.ServeThumbnail).Methods("GET")
    
    app.router = r
}

func (app *Application) Router() http.Handler {
    return app.router
}

func (app *Application) StartScheduler() {
    app.scheduler.Start()
}

func (app *Application) StartWebSocket() {
    go app.wsHub.Run()
}

func (app *Application) Shutdown(ctx context.Context) error {
    close(app.shutdown)
    app.scheduler.Stop()
    app.wsHub.Stop()
    return nil
}

func (app *Application) getSessionKey() string {
    var key string
    app.db.QueryRow(`
        SELECT value FROM system_settings WHERE key = 'session_key'
    `).Scan(&key)
    
    if key == "" {
        // Generate new key
        key = generateRandomKey(32)
        app.db.Exec(`
            INSERT INTO system_settings (key, value, type, category)
            VALUES ('session_key', $1, 'string', 'security')
        `, key)
    }
    
    return key
}

func (app *Application) getStoragePath() string {
    var path string
    app.db.QueryRow(`
        SELECT value FROM system_settings WHERE key = 'storage_path'
    `).Scan(&path)
    
    if path == "" {
        // Use default in data directory
        path = filepath.Join(defaultDataDir(), "storage")
        os.MkdirAll(path, 0755)
        app.db.Exec(`
            INSERT INTO system_settings (key, value, type, category)
            VALUES ('storage_path', $1, 'string', 'storage')
        `, path)
    }
    
    return path
}
```

### Database Migrations (Embedded)

```sql
-- web/migrations/001_initial.sql
-- Initial schema setup
CREATE TABLE IF NOT EXISTS system_settings (
    key VARCHAR(255) PRIMARY KEY,
    value TEXT,
    type VARCHAR(50) DEFAULT 'string',
    category VARCHAR(50),
    description TEXT,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert default settings
INSERT INTO system_settings (key, value, type, category, description) VALUES
('site_name', 'casboard', 'string', 'site', 'Site name'),
('site_tagline', 'Modern Anonymous Image Board', 'string', 'site', 'Site tagline'),
('initialized', 'false', 'boolean', 'system', 'System initialized'),
('server_port', '0', 'integer', 'server', 'Server port (0 = auto)');

-- web/migrations/002_users.sql
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username VARCHAR(32) UNIQUE,
    email VARCHAR(255) UNIQUE,
    password_hash VARCHAR(255),
    role VARCHAR(20) DEFAULT 'anon' CHECK (role IN ('anon','user','moderator','admin')),
    tripcode_secret VARCHAR(255),
    registration_ip_hash VARCHAR(64),
    last_login_ip_hash VARCHAR(64),
    post_count INTEGER DEFAULT 0,
    thread_count INTEGER DEFAULT 0,
    ban_count INTEGER DEFAULT 0,
    warning_count INTEGER DEFAULT 0,
    reputation_score INTEGER DEFAULT 0,
    email_verified INTEGER DEFAULT 0,
    two_factor_enabled INTEGER DEFAULT 0,
    two_factor_secret VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen_at TIMESTAMP,
    deleted_at TIMESTAMP
);

-- web/migrations/003_boards.sql
CREATE TABLE IF NOT EXISTS boards (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    slug VARCHAR(10) UNIQUE NOT NULL,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    position INTEGER DEFAULT 0,
    nsfw INTEGER DEFAULT 0,
    thread_limit INTEGER DEFAULT 150,
    bump_limit INTEGER DEFAULT 300,
    cooldown_threads INTEGER DEFAULT 300,
    cooldown_replies INTEGER DEFAULT 60,
    max_file_size INTEGER DEFAULT 10485760,
    image_required_op INTEGER DEFAULT 1,
    enable_spoilers INTEGER DEFAULT 1,
    enable_code_tags INTEGER DEFAULT 1,
    show_poster_ids INTEGER DEFAULT 0,
    show_country_flags INTEGER DEFAULT 0,
    unique_posts INTEGER DEFAULT 0,
    archive_days INTEGER DEFAULT 90,
    prune_days INTEGER DEFAULT 30,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert default boards
INSERT INTO boards (slug, name, description, position) VALUES
('/b/', 'Random', 'Anything goes', 1),
('/pol/', 'Politically Incorrect', 'Political discussion', 2),
('/v/', 'Video Games', 'Gaming discussion', 3),
('/a/', 'Anime & Manga', 'Japanese animation and comics', 4),
('/g/', 'Technology', 'Tech support and discussion', 5);

-- Continue with all other tables...
```

### Makefile for Single Binary

```makefile
# Makefile
BINARY_NAME=casboard
VERSION=1.0.0
BUILD_TIME=$(shell date +%FT%T%z)
BUILD_HASH=$(shell git rev-parse --short HEAD 2>/dev/null || echo "unknown")
LDFLAGS=-ldflags "-X main.version=${VERSION} -X main.build=${BUILD_HASH} -w -s"
STATIC_FLAGS=-tags osusergo,netgo,sqlite_omit_load_extension
CGO_FLAGS=CGO_ENABLED=0

.PHONY: all build clean test release

all: build

# Build single static binary with embedded resources
build:
	@echo "Building ${BINARY_NAME} static binary..."
	${CGO_FLAGS} go build ${STATIC_FLAGS} ${LDFLAGS} -o ${BINARY_NAME} cmd/casboard/main.go
	@echo "Build complete: ${BINARY_NAME} ($(shell du -h ${BINARY_NAME} | cut -f1))"

# Build for all platforms
build-all:
	@echo "Building for all platforms..."
	# Linux AMD64
	GOOS=linux GOARCH=amd64 ${CGO_FLAGS} go build ${STATIC_FLAGS} ${LDFLAGS} \
		-o ${BINARY_NAME}-linux-amd64 cmd/casboard/main.go
	
	# Linux ARM64
	GOOS=linux GOARCH=arm64 ${CGO_FLAGS} go build ${STATIC_FLAGS} ${LDFLAGS} \
		-o ${BINARY_NAME}-linux-arm64 cmd/casboard/main.go
	
	# Darwin AMD64
	GOOS=darwin GOARCH=amd64 ${CGO_FLAGS} go build ${STATIC_FLAGS} ${LDFLAGS} \
		-o ${BINARY_NAME}-darwin-amd64 cmd/casboard/main.go
	
	# Darwin ARM64
	GOOS=darwin GOARCH=arm64 ${CGO_FLAGS} go build ${STATIC_FLAGS} ${LDFLAGS} \
		-o ${BINARY_NAME}-darwin-arm64 cmd/casboard/main.go
	
	# Windows AMD64
	GOOS=windows GOARCH=amd64 ${CGO_FLAGS} go build ${STATIC_FLAGS} ${LDFLAGS} \
		-o ${BINARY_NAME}-windows-amd64.exe cmd/casboard/main.go
	
	# FreeBSD AMD64
	GOOS=freebsd GOARCH=amd64 ${CGO_FLAGS} go build ${STATIC_FLAGS} ${LDFLAGS} \
		-o ${BINARY_NAME}-freebsd-amd64 cmd/casboard/main.go
	
	@echo "Multi-platform build complete"

# Create release packages
release: build-all
	@echo "Creating release packages..."
	mkdir -p release
	
	# Create archives for each platform
	tar -czf release/${BINARY_NAME}-${VERSION}-linux-amd64.tar.gz ${BINARY_NAME}-linux-amd64
	tar -czf release/${BINARY_NAME}-${VERSION}-linux-arm64.tar.gz ${BINARY_NAME}-linux-arm64
	tar -czf release/${BINARY_NAME}-${VERSION}-darwin-amd64.tar.gz ${BINARY_NAME}-darwin-amd64
	tar -czf release/${BINARY_NAME}-${VERSION}-darwin-arm64.tar.gz ${BINARY_NAME}-darwin-arm64
	tar -czf release/${BINARY_NAME}-${VERSION}-freebsd-amd64.tar.gz ${BINARY_NAME}-freebsd-amd64
	zip release/${BINARY_NAME}-${VERSION}-windows-amd64.zip ${BINARY_NAME}-windows-amd64.exe
	
	# Generate checksums
	cd release && sha256sum * > checksums.txt
	
	@echo "Release packages created in release/"

# Test the binary
test:
	@echo "Running tests..."
	${CGO_FLAGS} go test ${STATIC_FLAGS} -v ./...

# Run locally
run: build
	./${BINARY_NAME}

# Install globally
install: build
	@echo "Installing ${BINARY_NAME}..."
	sudo cp ${BINARY_NAME} /usr/local/bin/
	@echo "Installed to /usr/local/bin/${BINARY_NAME}"

# Uninstall
uninstall:
	@echo "Uninstalling ${BINARY_NAME}..."
	sudo rm -f /usr/local/bin/${BINARY_NAME}
	rm -rf ~/.local/share/casboard
	@echo "Uninstalled"

# Clean build artifacts
clean:
	rm -f ${BINARY_NAME} ${BINARY_NAME}-*
	rm -rf release/
	go clean -cache

# Docker build (still single binary)
docker:
	@echo "Building Docker image..."
	docker build -t ghcr.io/casapps/${BINARY_NAME}:${VERSION} \
		--build-arg VERSION=${VERSION} \
		--build-arg BUILD=${BUILD_HASH} \
		-f Dockerfile.static .
	docker tag ghcr.io/casapps/${BINARY_NAME}:${VERSION} ghcr.io/casapps/${BINARY_NAME}:latest

# Size analysis
size: build
	@echo "Binary size analysis:"
	@ls -lh ${BINARY_NAME}
	@file ${BINARY_NAME}
	@echo ""
	@echo "Embedded resources:"
	@go list -f '{{.EmbedFiles}}' ./cmd/casboard
```

### Minimal Dockerfile for Static Binary

```dockerfile
# Dockerfile.static
# Build stage
FROM golang:1.21-alpine AS builder

ARG VERSION=dev
ARG BUILD=unknown

WORKDIR /build

# Copy source
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# Build static binary
RUN CGO_ENABLED=0 GOOS=linux go build \
    -tags osusergo,netgo,sqlite_omit_load_extension \
    -ldflags "-X main.version=${VERSION} -X main.build=${BUILD} -w -s" \
    -o casboard cmd/casboard/main.go

# Minimal runtime stage
FROM scratch

# Copy binary
COPY --from=builder /build/casboard /casboard

# Copy SSL certificates for HTTPS
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 64000-64999

ENTRYPOINT ["/casboard"]
```

### Systemd Service

```ini
# /etc/systemd/system/casboard.service
[Unit]
Description=casboard Image Board
After=network.target

[Service]
Type=simple
User=casboard
Group=casboard
WorkingDirectory=/var/lib/casboard
ExecStart=/usr/local/bin/casboard
Restart=always
RestartSec=10

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/casboard

# Resource limits
LimitNOFILE=65536
LimitNPROC=512

[Install]
WantedBy=multi-user.target
```

### Installation Script

```bash
#!/bin/bash
# install.sh

set -e

BINARY_URL="https://github.com/casapps/casboard/releases/latest/download/casboard-linux-amd64"
INSTALL_DIR="/usr/local/bin"
DATA_DIR="/var/lib/casboard"
USER="casboard"

echo "Installing casboard..."

# Create user if doesn't exist
if ! id "$USER" &>/dev/null; then
    sudo useradd -r -s /bin/false -m -d "$DATA_DIR" "$USER"
fi

# Download binary
echo "Downloading binary..."
sudo curl -L "$BINARY_URL" -o "$INSTALL_DIR/casboard"
sudo chmod +x "$INSTALL_DIR/casboard"

# Create data directory
sudo mkdir -p "$DATA_DIR"
sudo chown -R "$USER:$USER" "$DATA_DIR"

# Install systemd service
echo "Installing systemd service..."
sudo curl -L "https://raw.githubusercontent.com/casapps/casboard/main/casboard.service" \
    -o /etc/systemd/system/casboard.service

# Reload systemd
sudo systemctl daemon-reload
sudo systemctl enable casboard

echo "Installation complete!"
echo ""
echo "Start casboard with: sudo systemctl start casboard"
echo "Check status with: sudo systemctl status casboard"
echo "View logs with: sudo journalctl -u casboard -f"
echo ""
echo "Access casboard at: http://localhost:<port>"
echo "Port will be shown in logs on first start"
```

## Complete Features Summary

The single binary includes:

1. **Embedded Resources**
   - All templates compiled into binary
   - All static files (CSS, JS, images) embedded
   - All database migrations embedded
   - No external file dependencies

2. **Auto-Configuration**
   - Automatic port assignment (64000-64999)
   - Port persistence in database
   - Auto-detection of database type
   - Self-configuring storage paths

3. **Built-in Services**
   - Integrated scheduler (no separate process)
   - Integrated WebSocket server
   - Built-in session management
   - Database-backed cache (no Redis required)

4. **Database Support**
   - Embedded SQLite (default, no setup)
   - PostgreSQL (optional, for scale)
   - Automatic migration on startup
   - All configuration in database

5. **Zero Dependencies**
   - Single static binary
   - No runtime dependencies
   - No external services required
   - Works offline

6. **Cross-Platform**
   - Linux (AMD64, ARM64)
   - macOS (Intel, Apple Silicon)
   - Windows (AMD64)
   - FreeBSD (AMD64)

7. **Simple Deployment**
   - Single file to deploy
   - No installation required
   - Runs from anywhere
   - Automatic setup wizard

**Binary size: ~15-20MB with all resources embedded**

This creates a truly portable, self-contained image board system that can be run with a single command on any supported platform.
