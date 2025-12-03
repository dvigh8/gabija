# Gabija Plugin Development Guide

**Version:** 0.1  
**Last Updated:** December 2024  
**Target Audience:** Plugin developers building extensions for Gabija

---

## Table of Contents

1. [Overview](#overview)
2. [Plugin Architecture](#plugin-architecture)
3. [gRPC Service Definitions](#grpc-service-definitions)
4. [Plugin SDK (Go)](#plugin-sdk-go)
5. [Development Workflow](#development-workflow)
6. [Code Examples](#code-examples)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)

---

## Overview

Gabija plugins extend the platform with new functionality while running in a secure WASM sandbox. Plugins communicate with the Gabija core via gRPC using Protocol Buffers.

### Key Concepts

- **WASM Sandbox**: Plugins run in isolated WebAssembly environment
- **gRPC Communication**: All plugin-core communication via Protocol Buffers
- **Capability-Based Security**: Plugins declare required permissions upfront
- **Plugin-Owned Tables**: Each plugin manages its own database tables
- **Event Bus**: Asynchronous inter-plugin communication

### What You Can Build

- Data management (tasks, recipes, inventory)
- External service integrations (APIs, webhooks)
- Automation and workflows
- Reports and analytics
- Custom UI components (via JSON APIs)

---

## Plugin Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Gabija Core (Go)                        │
│  ┌────────────────────────────────────────────────────┐ │
│  │           Plugin Manager                           │ │
│  │  - Lifecycle Management                            │ │
│  │  - HTTP Request Routing                            │ │
│  │  - Capability Enforcement                          │ │
│  │  - Event Bus Coordination                          │ │
│  └────────────────┬───────────────────────────────────┘ │
│                   │                                      │
│  ┌────────────────▼───────────────────────────────────┐ │
│  │         gRPC Services (Protocol Buffers)           │ │
│  │                                                     │ │
│  │  Core → Plugin:                                    │ │
│  │  - Setup/Teardown (lifecycle)                      │ │
│  │  - HandleHttpRequest (route handler)               │ │
│  │  - HandleEvent (event delivery)                    │ │
│  │                                                     │ │
│  │  Plugin → Core:                                    │ │
│  │  - Query/Exec (database access)                    │ │
│  │  - PublishEvent (event bus)                        │ │
│  │  - HttpRequest (external API calls)                │ │
│  │  - GetSecret/SetSecret (secrets management)        │ │
│  └────────────────┬───────────────────────────────────┘ │
└───────────────────┼───────────────────────────────────┬─┘
                    │ gRPC over WASI                    │
┌───────────────────▼───────────────────────────────────▼─┐
│              WASM Plugin Runtime                         │
│            (wasmtime-go)                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Todo Plugin │  │ Future Plugin│  │ Your Plugin  │  │
│  │  (.wasm)     │  │  (.wasm)     │  │  (.wasm)     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Plugin Components

1. **manifest.yaml** - Plugin metadata, capabilities, routes, migrations
2. **main.go** - Plugin implementation (compiled to WASM)
3. **migrations/** - SQL files for database schema
4. **proto/** - Protocol Buffer definitions (optional customization)

---

## gRPC Service Definitions

### Core → Plugin Services

These services are **implemented by your plugin** and **called by the core**.

```protobuf
// core_to_plugin.proto

syntax = "proto3";
package gabija.plugin;

// Service that every plugin must implement
service PluginService {
  // Lifecycle hooks
  rpc Setup(SetupRequest) returns (SetupResponse);
  rpc Teardown(Empty) returns (Empty);
  rpc HealthCheck(Empty) returns (HealthResponse);
  
  // HTTP request handling
  rpc HandleHttpRequest(HttpRequest) returns (HttpResponse);
  
  // Event handling
  rpc HandleEvent(Event) returns (Empty);
  
  // Configuration updates
  rpc OnConfigUpdate(ConfigUpdate) returns (Empty);
}

message SetupRequest {
  string plugin_id = 1;
  map<string, string> config = 2;
  string data_dir = 3;
}

message SetupResponse {
  bool success = 1;
  string error_message = 2;
}

message HttpRequest {
  string method = 1;           // GET, POST, PATCH, DELETE
  string path = 2;             // /api/tasks
  map<string, string> headers = 3;
  bytes body = 4;
  
  // Authentication context (provided by core)
  string user_id = 5;
  string household_id = 6;
  string plugin_id = 7;
}

message HttpResponse {
  int32 status_code = 1;
  map<string, string> headers = 2;
  bytes body = 3;
}

message Event {
  string type = 1;              // "household.member_added"
  string source_plugin = 2;     // plugin that published event
  google.protobuf.Timestamp timestamp = 3;
  map<string, string> data = 4; // event payload
}

message HealthResponse {
  bool healthy = 1;
  string message = 2;
}

message ConfigUpdate {
  map<string, string> config = 1;
}

message Empty {}
```

### Plugin → Core Services

These services are **provided by the core** and **called by your plugin**.

```protobuf
// plugin_to_core.proto

syntax = "proto3";
package gabija.core;

// Database access service
service CoreDataService {
  // Query plugin's own tables
  rpc Query(QueryRequest) returns (QueryResponse);
  
  // Execute write operations (INSERT, UPDATE, DELETE)
  rpc Exec(ExecRequest) returns (ExecResponse);
  
  // Execute multiple operations in a transaction
  rpc Transaction(TransactionRequest) returns (TransactionResponse);
}

// Event bus service
service CoreEventService {
  // Publish events to event bus
  rpc Publish(Event) returns (Empty);
  
  // Subscribe to events (server streaming)
  rpc Subscribe(EventFilter) returns (stream Event);
}

// External HTTP service (capability-controlled)
service CoreHttpService {
  // Make HTTP requests to external APIs
  rpc Request(HttpRequest) returns (HttpResponse);
}

// Secrets management service
service CoreSecretService {
  // Get secret (capability-controlled)
  rpc GetSecret(SecretRequest) returns (SecretResponse);
  
  // Store secret (encrypted at rest)
  rpc SetSecret(SecretRequest) returns (Empty);
}

// ─── Request/Response Messages ───

message QueryRequest {
  string sql = 1;                // Parameterized SQL
  repeated Value params = 2;     // Query parameters
  RequestContext context = 3;    // user_id, household_id
}

message QueryResponse {
  repeated Row rows = 1;
}

message Row {
  repeated Value columns = 1;
}

message Value {
  oneof kind {
    string string_value = 1;
    int64 int_value = 2;
    double double_value = 3;
    bool bool_value = 4;
    bytes bytes_value = 5;
  }
}

message ExecRequest {
  string sql = 1;
  repeated Value params = 2;
  RequestContext context = 3;
}

message ExecResponse {
  int64 rows_affected = 1;
  int64 last_insert_id = 2;
}

message TransactionRequest {
  repeated ExecRequest operations = 1;
  RequestContext context = 2;
}

message TransactionResponse {
  repeated ExecResponse results = 1;
}

message RequestContext {
  string user_id = 1;
  string household_id = 2;
  string plugin_id = 3;        // Automatically added by core
}

message EventFilter {
  repeated string event_patterns = 1;  // "task.*", "household.member_added"
}

message SecretRequest {
  string key = 1;
  string value = 2;  // Empty for Get, populated for Set
}

message SecretResponse {
  string value = 1;
  bool found = 2;
}
```

### Request Flow Diagrams

#### HTTP Request Flow

```
┌──────┐         ┌──────────┐         ┌────────────────┐         ┌────────┐
│Client│         │   Core   │         │ Plugin Manager │         │ Plugin │
└──┬───┘         └────┬─────┘         └───────┬────────┘         └───┬────┘
   │                  │                        │                      │
   │ GET /api/tasks   │                        │                      │
   ├─────────────────>│                        │                      │
   │                  │                        │                      │
   │                  │ 1. Authenticate user   │                      │
   │                  │ 2. Determine plugin    │                      │
   │                  │    (route → todo-list) │                      │
   │                  │                        │                      │
   │                  │ gRPC HandleHttpRequest │                      │
   │                  │    (with user context) │                      │
   │                  ├───────────────────────>│                      │
   │                  │                        │                      │
   │                  │                        │ 3. Check capabilities│
   │                  │                        ├─────────────────────>│
   │                  │                        │                      │
   │                  │                        │<─────────────────────┤
   │                  │                        │   (authorized)       │
   │                  │                        │                      │
   │                  │                        │ 4. Call plugin       │
   │                  │                        │    HandleHttpRequest │
   │                  │                        ├─────────────────────>│
   │                  │                        │                      │
   │                  │                        │                      │─┐
   │                  │                        │                      │ │ 5. Plugin
   │                  │                        │                      │ │    queries DB
   │                  │                        │                      │ │    via gRPC
   │                  │                        │                      │<┘
   │                  │                        │                      │
   │                  │                        │ HttpResponse (JSON)  │
   │                  │                        │<─────────────────────┤
   │                  │                        │                      │
   │                  │<───────────────────────┤                      │
   │                  │                        │                      │
   │ JSON Response    │                        │                      │
   │<─────────────────┤                        │                      │
   │                  │                        │                      │
```

#### Event Publication Flow

```
┌────────────┐         ┌──────────┐         ┌──────────┐
│Plugin A    │         │Event Bus │         │Plugin B  │
│(Publisher) │         │          │         │(Subscriber)│
└─────┬──────┘         └────┬─────┘         └────┬─────┘
      │                     │                     │
      │ PublishEvent        │                     │
      │ ("task.created")    │                     │
      ├────────────────────>│                     │
      │                     │                     │
      │                     │ 1. Store in DB      │
      │                     │ 2. Fan out to       │
      │                     │    subscribers      │
      │                     │                     │
      │                     │ HandleEvent         │
      │                     ├────────────────────>│
      │                     │ (async)             │
      │                     │                     │
      │                     │                     │─┐
      │                     │                     │ │ React to
      │                     │                     │ │ event
      │                     │                     │<┘
      │ ack                 │                     │
      │<────────────────────┤                     │
      │                     │                     │
```

---

## Plugin SDK (Go)

### Installation

```bash
go get github.com/gabija/plugin-sdk-go@latest
```

### Basic Plugin Structure

```go
package main

import (
    "context"
    "encoding/json"
    
    "github.com/gabija/plugin-sdk-go/plugin"
)

func main() {
    // Create plugin instance
    p := plugin.New("todo-list")
    
    // Register HTTP handlers
    p.HandleHTTP("GET", "/api/tasks", handleGetTasks)
    p.HandleHTTP("POST", "/api/tasks", handleCreateTask)
    p.HandleHTTP("PATCH", "/api/tasks/:id", handleUpdateTask)
    p.HandleHTTP("DELETE", "/api/tasks/:id", handleDeleteTask)
    
    // Subscribe to events
    p.OnEvent("household.member_added", onMemberAdded)
    p.OnEvent("household.member_removed", onMemberRemoved)
    
    // Lifecycle hooks
    p.OnSetup(setupPlugin)
    p.OnTeardown(teardownPlugin)
    
    // Run plugin (blocks)
    if err := p.Run(); err != nil {
        panic(err)
    }
}

// Setup hook - called once when plugin is enabled
func setupPlugin(ctx *plugin.Context) error {
    ctx.Logger.Info("Todo plugin starting up")
    
    // Initialize any resources, validate config, etc.
    return nil
}

// Teardown hook - called when plugin is disabled
func teardownPlugin(ctx *plugin.Context) error {
    ctx.Logger.Info("Todo plugin shutting down")
    
    // Cleanup resources
    return nil
}
```

### HTTP Handler Example

```go
// Task model
type Task struct {
    ID          int64     `json:"id"`
    HouseholdID string    `json:"household_id"`
    Title       string    `json:"title"`
    Description string    `json:"description"`
    Completed   bool      `json:"completed"`
    AssignedTo  string    `json:"assigned_to"` // user_id
    DueDate     time.Time `json:"due_date"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}

// GET /api/tasks - List all tasks for household
func handleGetTasks(ctx *plugin.Context) (*plugin.Response, error) {
    // Query is automatically scoped to ctx.HouseholdID
    rows, err := ctx.Query(`
        SELECT id, household_id, title, description, completed, 
               assigned_to, due_date, created_at, updated_at
        FROM tasks
        WHERE household_id = $1
        ORDER BY created_at DESC
    `, ctx.HouseholdID)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var tasks []Task
    for rows.Next() {
        var t Task
        err := rows.Scan(&t.ID, &t.HouseholdID, &t.Title, &t.Description,
            &t.Completed, &t.AssignedTo, &t.DueDate, &t.CreatedAt, &t.UpdatedAt)
        if err != nil {
            return nil, err
        }
        tasks = append(tasks, t)
    }
    
    return plugin.JSON(tasks), nil
}

// POST /api/tasks - Create new task
func handleCreateTask(ctx *plugin.Context) (*plugin.Response, error) {
    // Parse request body
    var req struct {
        Title       string    `json:"title"`
        Description string    `json:"description"`
        AssignedTo  string    `json:"assigned_to"`
        DueDate     time.Time `json:"due_date"`
    }
    
    if err := ctx.BindJSON(&req); err != nil {
        return plugin.ErrorBadRequest("Invalid JSON"), nil
    }
    
    // Validate
    if req.Title == "" {
        return plugin.ErrorBadRequest("Title is required"), nil
    }
    
    // Insert into database
    result, err := ctx.Exec(`
        INSERT INTO tasks (household_id, title, description, completed, 
                          assigned_to, due_date, created_at, updated_at)
        VALUES ($1, $2, $3, false, $4, $5, NOW(), NOW())
        RETURNING id
    `, ctx.HouseholdID, req.Title, req.Description, req.AssignedTo, req.DueDate)
    if err != nil {
        return nil, err
    }
    
    taskID := result.LastInsertID()
    
    // Publish event to event bus
    err = ctx.PublishEvent("task.created", map[string]interface{}{
        "task_id":      taskID,
        "title":        req.Title,
        "household_id": ctx.HouseholdID,
        "created_by":   ctx.UserID,
    })
    if err != nil {
        ctx.Logger.Warnf("Failed to publish task.created event: %v", err)
        // Don't fail the request if event publishing fails
    }
    
    return plugin.JSON(map[string]interface{}{
        "id": taskID,
    }), nil
}

// PATCH /api/tasks/:id - Update task
func handleUpdateTask(ctx *plugin.Context) (*plugin.Response, error) {
    taskID := ctx.Param("id")
    
    var req struct {
        Title       *string    `json:"title"`
        Description *string    `json:"description"`
        Completed   *bool      `json:"completed"`
        AssignedTo  *string    `json:"assigned_to"`
        DueDate     *time.Time `json:"due_date"`
    }
    
    if err := ctx.BindJSON(&req); err != nil {
        return plugin.ErrorBadRequest("Invalid JSON"), nil
    }
    
    // Build dynamic UPDATE query
    updates := []string{}
    params := []interface{}{ctx.HouseholdID, taskID}
    paramIdx := 3
    
    if req.Title != nil {
        updates = append(updates, fmt.Sprintf("title = $%d", paramIdx))
        params = append(params, *req.Title)
        paramIdx++
    }
    if req.Description != nil {
        updates = append(updates, fmt.Sprintf("description = $%d", paramIdx))
        params = append(params, *req.Description)
        paramIdx++
    }
    if req.Completed != nil {
        updates = append(updates, fmt.Sprintf("completed = $%d", paramIdx))
        params = append(params, *req.Completed)
        paramIdx++
    }
    if req.AssignedTo != nil {
        updates = append(updates, fmt.Sprintf("assigned_to = $%d", paramIdx))
        params = append(params, *req.AssignedTo)
        paramIdx++
    }
    if req.DueDate != nil {
        updates = append(updates, fmt.Sprintf("due_date = $%d", paramIdx))
        params = append(params, *req.DueDate)
        paramIdx++
    }
    
    if len(updates) == 0 {
        return plugin.ErrorBadRequest("No fields to update"), nil
    }
    
    updates = append(updates, "updated_at = NOW()")
    
    query := fmt.Sprintf(`
        UPDATE tasks
        SET %s
        WHERE household_id = $1 AND id = $2
    `, strings.Join(updates, ", "))
    
    result, err := ctx.Exec(query, params...)
    if err != nil {
        return nil, err
    }
    
    if result.RowsAffected() == 0 {
        return plugin.ErrorNotFound("Task not found"), nil
    }
    
    // Publish event if task was completed
    if req.Completed != nil && *req.Completed {
        ctx.PublishEvent("task.completed", map[string]interface{}{
            "task_id":      taskID,
            "household_id": ctx.HouseholdID,
            "completed_by": ctx.UserID,
        })
    }
    
    return plugin.JSON(map[string]interface{}{
        "success": true,
    }), nil
}
```

### Event Handler Example

```go
// Handle household.member_added event
func onMemberAdded(ctx *plugin.Context, event *plugin.Event) error {
    memberID := event.Data["member_id"]
    memberName := event.Data["member_name"]
    
    ctx.Logger.Infof("New member added: %s (%s)", memberName, memberID)
    
    // Create welcome task for new member
    _, err := ctx.Exec(`
        INSERT INTO tasks (household_id, title, description, completed, 
                          assigned_to, created_at, updated_at)
        VALUES ($1, $2, $3, false, $4, NOW(), NOW())
    `,
        ctx.HouseholdID,
        fmt.Sprintf("Welcome, %s!", memberName),
        "Complete this task to get started with Gabija!",
        memberID,
    )
    
    if err != nil {
        ctx.Logger.Errorf("Failed to create welcome task: %v", err)
        return err
    }
    
    return nil
}

// Handle household.member_removed event
func onMemberRemoved(ctx *plugin.Context, event *plugin.Event) error {
    memberID := event.Data["member_id"]
    
    // Reassign or delete tasks assigned to removed member
    _, err := ctx.Exec(`
        UPDATE tasks
        SET assigned_to = NULL, updated_at = NOW()
        WHERE household_id = $1 AND assigned_to = $2
    `, ctx.HouseholdID, memberID)
    
    return err
}
```

### External API Call Example

```go
// Make HTTP request to external API (requires capability)
func syncWithTodoist(ctx *plugin.Context) error {
    // Get API token from secrets
    token, err := ctx.GetSecret("todoist_api_token")
    if err != nil {
        return err
    }
    
    // Make HTTP request (capability-checked by core)
    resp, err := ctx.HTTPRequest(&plugin.HTTPRequest{
        Method: "GET",
        URL:    "https://api.todoist.com/rest/v2/tasks",
        Headers: map[string]string{
            "Authorization": fmt.Sprintf("Bearer %s", token),
        },
    })
    if err != nil {
        return err
    }
    
    if resp.StatusCode != 200 {
        return fmt.Errorf("todoist API returned %d", resp.StatusCode)
    }
    
    var tasks []TodoistTask
    if err := json.Unmarshal(resp.Body, &tasks); err != nil {
        return err
    }
    
    // Import tasks...
    for _, task := range tasks {
        // Insert into database
        ctx.Exec(`INSERT INTO tasks ...`)
    }
    
    return nil
}
```

---

## Development Workflow

### 1. Create Plugin

```bash
# Use CLI to scaffold plugin
$ gabija plugin new my-plugin

# Creates structure:
# my-plugin/
# ├── manifest.yaml
# ├── main.go
# ├── go.mod
# ├── go.sum
# ├── migrations/
# │   └── 001_initial.sql
# ├── README.md
# └── .github/
#     └── workflows/
#         └── release.yml
```

### 2. Implement Plugin

Edit `main.go` and add your plugin logic using the SDK.

### 3. Build Plugin

```bash
# Compile Go to WASM
$ gabija plugin build

# Output: dist/my-plugin.wasm
```

Under the hood, this runs:
```bash
tinygo build -o dist/my-plugin.wasm -target=wasi main.go
```

### 4. Test Locally

```bash
# Install plugin locally
$ gabija plugin install ./dist

# Enable plugin
$ gabija plugin enable my-plugin

# Test HTTP endpoints
$ curl http://localhost:3000/api/my-plugin/...

# View logs
$ gabija logs -f --plugin=my-plugin
```

### 5. Publish to GitHub

```bash
# Create GitHub repository
$ git init
$ git add .
$ git commit -m "Initial commit"
$ git remote add origin github.com/username/my-plugin
$ git push -u origin main

# Create release (triggers CI build)
$ git tag v1.0.0
$ git push origin v1.0.0
```

GitHub Actions workflow will:
1. Build WASM binary
2. Create GitHub release
3. Attach `my-plugin.wasm` and `manifest.yaml`

### 6. Users Install

```bash
$ gabija plugin install github.com/username/my-plugin
```

---

## Plugin Installation Flow

```
┌─────────────────────────────────────────────────────────┐
│ $ gabija plugin install github.com/user/my-plugin      │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 1. Fetch GitHub Release                                 │
│    - GET /repos/user/my-plugin/releases/latest          │
│    - Download my-plugin.wasm                            │
│    - Download manifest.yaml                             │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Validate Manifest                                    │
│    - Check schema                                       │
│    - Validate capabilities                              │
│    - Check dependencies                                 │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Extract to ~/.gabija/plugins/my-plugin/             │
│    - manifest.yaml                                      │
│    - my-plugin.wasm                                     │
│    - migrations/                                        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 4. Run Database Migrations                              │
│    - migrations/001_initial.sql                         │
│    - migrations/002_add_column.sql                      │
│    - Track in schema_migrations table                   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 5. Load WASM Module                                     │
│    - wasmtime.LoadModule(my-plugin.wasm)                │
│    - Allocate memory (64MB)                             │
│    - Set CPU limits                                     │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 6. Initialize gRPC Client                               │
│    - Connect to plugin's gRPC server                    │
│    - Register with Plugin Manager                       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 7. Call Setup() Lifecycle Hook                          │
│    - plugin.Setup(config)                               │
│    - Plugin initializes resources                       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ 8. Register Routes & Events                             │
│    - Register HTTP routes from manifest                 │
│    - Subscribe to events                                │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│ ✓ Plugin Active!                                        │
│   - Responding to HTTP requests                         │
│   - Receiving events                                    │
│   - Processing in WASM sandbox                          │
└─────────────────────────────────────────────────────────┘
```

---

## Event Bus Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Event Bus                             │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │          Publisher Interface                       │ │
│  │  - PublishEvent(type, data)                        │ │
│  │  - Validates event schema                          │ │
│  │  - Stores in events table                          │ │
│  └──────────────┬─────────────────────────────────────┘ │
│                 │                                        │
│  ┌──────────────▼─────────────────────────────────────┐ │
│  │         Event Storage (PostgreSQL)                 │ │
│  │  - Persistent event log                            │ │
│  │  - Audit trail                                     │ │
│  │  - Replay capability                               │ │
│  └──────────────┬─────────────────────────────────────┘ │
│                 │                                        │
│  ┌──────────────▼─────────────────────────────────────┐ │
│  │    Subscriber Registry                             │ │
│  │  - Maps event patterns to plugins                  │ │
│  │  - "task.*" → [todo-plugin, analytics-plugin]      │ │
│  │  - "household.member_added" → [todo-plugin]        │ │
│  └──────────────┬─────────────────────────────────────┘ │
│                 │                                        │
│  ┌──────────────▼─────────────────────────────────────┐ │
│  │         Delivery Engine                            │ │
│  │  - Fan-out to subscribers (async)                  │ │
│  │  - Retry on failure (3 attempts)                   │ │
│  │  - Dead letter queue for failed deliveries         │ │
│  │  - Rate limiting (100 events/min per plugin)       │ │
│  └──────────────┬─────────────────────────────────────┘ │
│                 │                                        │
└─────────────────┼────────────────────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
┌───────────────┐   ┌───────────────┐
│  Plugin A     │   │  Plugin B     │
│  .HandleEvent │   │  .HandleEvent │
└───────────────┘   └───────────────┘
```

### Event Patterns

Plugins subscribe to events using glob patterns:

- `task.*` - All task events (task.created, task.completed, etc.)
- `household.member_added` - Specific event
- `*.created` - All creation events
- `*` - All events (use sparingly!)

---

## Best Practices

### 1. Database Queries

**✅ DO:**
```go
// Use parameterized queries
rows, err := ctx.Query(`
    SELECT * FROM tasks WHERE household_id = $1 AND id = $2
`, ctx.HouseholdID, taskID)
```

**❌ DON'T:**
```go
// Never use string interpolation
query := fmt.Sprintf("SELECT * FROM tasks WHERE id = %s", taskID)
rows, err := ctx.Query(query) // SQL injection risk!
```

### 2. Error Handling

**✅ DO:**
```go
func handleGetTask(ctx *plugin.Context) (*plugin.Response, error) {
    taskID := ctx.Param("id")
    
    var task Task
    err := ctx.QueryRow(`
        SELECT * FROM tasks WHERE id = $1 AND household_id = $2
    `, taskID, ctx.HouseholdID).Scan(&task)
    
    if err == sql.ErrNoRows {
        return plugin.ErrorNotFound("Task not found"), nil
    }
    if err != nil {
        ctx.Logger.Errorf("Query failed: %v", err)
        return nil, err // Let framework handle 500
    }
    
    return plugin.JSON(task), nil
}
```

### 3. Event Publishing

**✅ DO:**
```go
// Publish events after successful operations
result, err := ctx.Exec(`INSERT INTO tasks ...`)
if err != nil {
    return nil, err
}

// Publish event (non-blocking)
ctx.PublishEvent("task.created", map[string]interface{}{
    "task_id": result.LastInsertID(),
})
```

**❌ DON'T:**
```go
// Don't fail the request if event publishing fails
err := ctx.PublishEvent("task.created", data)
if err != nil {
    return nil, err // TOO STRICT - event bus issues shouldn't break user requests
}
```

### 4. Logging

```go
// Use structured logging
ctx.Logger.Infof("Task created: id=%d, household=%s", taskID, ctx.HouseholdID)
ctx.Logger.Warnf("Rate limit approaching: %d/%d", count, limit)
ctx.Logger.Errorf("Database query failed: %v", err)
```

### 5. Transactions

```go
// Use transactions for multiple related operations
tx, err := ctx.BeginTx()
if err != nil {
    return nil, err
}
defer tx.Rollback() // Safe to call even after commit

// Multiple operations
_, err = tx.Exec(`INSERT INTO tasks ...`)
if err != nil {
    return nil, err
}

_, err = tx.Exec(`INSERT INTO task_assignments ...`)
if err != nil {
    return nil, err
}

// Commit transaction
if err := tx.Commit(); err != nil {
    return nil, err
}
```

### 6. Capability Declaration

**manifest.yaml:**
```yaml
capabilities:
  data:
    - read:tasks           # Only declare what you need
    - write:tasks
  events:
    - subscribe:household.member_added
    - publish:task.created
  http:
    - request:https://api.todoist.com/*  # Specific domain
```

**Principle of Least Privilege:** Only request capabilities you actually use.

---

## Troubleshooting

### Plugin Not Loading

**Symptoms:** Plugin doesn't appear in `gabija plugin list`

**Checks:**
1. Verify manifest.yaml syntax: `gabija plugin validate`
2. Check logs: `gabija logs --plugin=my-plugin`
3. Ensure .wasm file exists in plugin directory
4. Verify dependencies are met

### Database Errors

**Symptoms:** Queries fail with "table does not exist"

**Solution:**
1. Check migrations ran: `gabija plugin migrations my-plugin`
2. Verify table name prefix: `myplug_tasks` not just `tasks`
3. Check household_id is included in WHERE clauses

### Capability Errors

**Symptoms:** `PermissionDenied: plugin lacks data.read:tasks capability`

**Solution:**
1. Add capability to manifest.yaml
2. Reinstall plugin: `gabija plugin reinstall my-plugin`
3. Check audit logs: `gabija audit --plugin=my-plugin`

### Performance Issues

**Symptoms:** Slow HTTP responses

**Checks:**
1. Profile queries: `EXPLAIN ANALYZE` in PostgreSQL
2. Add indexes to frequently queried columns
3. Batch operations instead of N+1 queries
4. Use connection pooling (SDK handles this)

### Event Not Received

**Symptoms:** Event handler not called

**Checks:**
1. Verify event pattern in manifest: `subscribe:task.*`
2. Check event bus logs: `gabija logs --component=event-bus`
3. Ensure event type matches: `task.created` vs `tasks.created`
4. Check rate limits: max 100 events/min per plugin

---

## Next Steps

1. **Build your first plugin**: Follow the [Todo Plugin Tutorial](./tutorial-todo-plugin.md)
2. **Read the API Reference**: [Plugin SDK API Docs](./api-reference.md)
3. **Join the community**: [Discord](https://discord.gg/gabija) or [GitHub Discussions](https://github.com/gabija/gabija/discussions)
4. **Share your plugin**: Submit to [Plugin Registry](https://plugins.gabija.app)

---

**Questions?** File an issue on [GitHub](https://github.com/gabija/gabija/issues) or ask in [Discord](https://discord.gg/gabija).
