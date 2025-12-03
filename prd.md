# Gabija: Self-Hosted Family Management Platform
## Product Requirements Document (PRD)
**Part-Time Solo Developer Edition - 18 Month Roadmap**

---

**Document Version:** 0.1  
**Last Updated:** December 2024  
**Project Status:** Planning (Pre-Development)  
**Development Model:** Part-Time Hobby (10-12 hrs/week)  
**Timeline:** 18 months to public launch  
**Target Users:** Tech-savvy families who value data ownership and extensibility

---

## Executive Summary

Gabija is a self-hosted home management platform built with Go and React that consolidates family operations into a single, privacy-first application with unlimited extensibility through a gRPC/Protocol Buffers plugin architecture with WASM sandboxing.

**Core Value Proposition:**  
Unlike existing family coordination apps (Cozi, FamilyWall), Gabija enables plugins to work together—your budget affects meal planning, travel dates pause shopping lists, and family schedules adjust chore assignments automatically.

**Development Philosophy:**  
Built as a sustainable hobby project by a solo staff engineer working part-time. Success is measured by personal utility and learning, not time-to-market pressure.

---

## Vision & Goals

### Long-Term Vision (3-5 years)
Gabija becomes the "Home Assistant for family life"—a platform where the self-hosting community builds extensions that transform how households operate.

### 18-Month Goals
1. **Personal Utility:** Build a tool you use daily and genuinely improves household management
2. **Technical Learning:** Master Go backend development, gRPC/Protocol Buffers, WASM sandboxing for plugins, and community platform building
3. **Community Validation:** 100-500 active users, 3-5 community plugins
4. **Sustainable Hobby:** $500-1,000 MRR covering costs and providing hobby income

### Success Metrics
- **Month 6:** You and household use Gabija daily for finance tracking
- **Month 12:** 20-50 beta users, plugin SDK documented
- **Month 18:** 100-200 active users, 2-3 community plugins, paid tier launched

---

## Market Context

### Target Users

**Primary:** Tech-Savvy Families
- Self-host services (Plex, Home Assistant, Nextcloud)
- Value data ownership and privacy
- Frustrated by limitations of existing apps
- Willing to tinker for better functionality
- 2-6 person households

**Secondary:** Privacy-Conscious Power Users
- Don't currently self-host but want to
- Fed up with ad-supported family apps
- Need features existing apps won't add (finance integration, custom workflows)

### Market Size
- r/selfhosted: 500K+ members
- Home Assistant users: 500K+ active installations
- **Realistic addressable market:** 10K-50K households globally

### Competitive Landscape

**Direct Competitors:**
- **Cozi (15M users):** Simple family coordination, not extensible, cloud-only
- **FamilyWall (2M users):** Similar to Cozi, slightly more features
- **Grocy (10K+ stars):** Groceries/household inventory, focused on solo use

**Adjacent Tools:**
- **Firefly III:** Self-hosted finance (individual-focused)
- **Mealie:** Recipe management (no family coordination)
- **Monica:** Personal CRM (not household management)

### Competitive Advantages
1. **Plugin interoperability:** Only platform where household data works together via event bus
2. **gRPC + WASM security:** Plugins run in WASM sandbox with gRPC communication—strong isolation without sacrificing performance
3. **Privacy-first:** Self-hosted, no data mining, AGPL licensed
4. **Technical credibility:** Built by staff engineer with real expertise

---

## Product Overview

### Core Platform Features

**User Management**
- Households with multi-member support
- Role-based permissions (admin, member, child)
- Simple email/password authentication
- Invite system for adding family members

**Data Architecture**
- PostgreSQL for structured data
- Event bus for plugin communication
- RBAC for fine-grained access control
- Audit logs for sensitive operations

**Frontend**
- Responsive React web application
- Works on mobile browsers (no native apps in v1)
- Tailwind CSS for styling
- Dark mode support

### Phase 1: Foundation (Months 1-4)

**Deliverables:**
- Core platform deployed to internet
- User authentication and household management
- Basic UI framework (header, navigation, settings)
- PostgreSQL database with migrations
- REST API foundation

**Features:**
- Create account with email/password
- Create household and invite members
- Basic profile management
- Simple dashboard (placeholder for plugins)

**Technical Stack:**
- Backend: Go 1.21+, Gorilla Mux (routing), sqlx (database)
- Database: PostgreSQL 15+
- Frontend: React 18, React Router, Tailwind CSS
- Deployment: Render/Railway ($15-20/month)

**Time Estimate:** 200 hours (4 months @ 12 hrs/week)

**Success Criteria:**
- You can log in from phone and laptop
- Partner can create account and join household
- No security vulnerabilities in auth flow
- Deploys to production with zero manual steps

### Phase 2: First Plugin - Finance Tracker (Months 5-9)

**Overview:**  
Build the best self-hosted family finance tracker by integrating SimpleFIN for automatic bank syncing.

**Core Features:**

**Account Management**
- Connect bank accounts via SimpleFIN
- Manual account creation (cash, credit cards not on SimpleFIN)
- Account balance tracking
- Transaction sync (automatic for SimpleFIN accounts)

**Budgeting**
- Create household budgets with categories
- Per-person spending views within shared budgets
- Monthly budget vs. actual tracking
- Budget alerts (threshold notifications)

**Transaction Management**
- Automatic categorization (with manual override)
- Expense splitting for shared purchases
- Transaction search and filtering
- Notes and receipt references

**Reporting**
- Spending trends over time (line charts)
- Category breakdown (pie charts)
- Per-person contribution views
- Export to CSV

**Not Included in v1:**
- ❌ Investment tracking (defer to Months 15+)
- ❌ Bill payment integration (manual entry only)
- ❌ Multi-currency support (USD only initially)
- ❌ Tax reporting (out of scope)
- ❌ Loan/mortgage tracking (defer)

**User Stories:**

1. **Account Setup**
   - As a user, I can connect my bank account via SimpleFIN so transactions sync automatically
   - As a user, I can manually add cash and credit accounts not supported by SimpleFIN

2. **Shared Budgeting**
   - As a household admin, I can create monthly budgets for categories (groceries, dining, gas)
   - As a household member, I can see my personal spending within shared budgets
   - As a user, I receive notification when category spending exceeds 80% of budget

3. **Expense Splitting**
   - As a user, I can mark a transaction as "split" and specify who owes what
   - As a partner, I can see transactions where I owe money to other household members

4. **Financial Visibility**
   - As a user, I can view spending trends over the past 6 months
   - As a user, I can see category breakdown to understand where money goes
   - As a household admin, I can see each member's contribution to shared expenses

**Technical Implementation:**

**SimpleFIN Integration**
- Sign up for SimpleFIN developer account ($500-1000/year for API access)
- OAuth 2.0 flow for account connection
- Webhook handling for real-time transaction updates
- Fallback polling for accounts without webhook support

**Database Schema:**
```sql
-- Core tables
accounts (id, household_id, name, type, balance, simplefin_id)
transactions (id, account_id, date, amount, description, category_id, user_id)
categories (id, household_id, name, budget_amount, color)
budget_periods (id, household_id, start_date, end_date)
splits (id, transaction_id, user_id, amount, settled)
```

**API Endpoints:**
```
POST   /api/finance/accounts/simplefin/connect
GET    /api/finance/accounts
POST   /api/finance/accounts (manual account creation)
GET    /api/finance/transactions
PATCH  /api/finance/transactions/:id (update category, add note)
POST   /api/finance/transactions/:id/split
GET    /api/finance/budgets/:period
GET    /api/finance/reports/spending-trends
```

**Time Estimate:** 250 hours (5 months @ 12 hrs/week)

**Success Criteria:**
- You stop using Mint/YNAB/spreadsheets and use Gabija exclusively
- Partner voluntarily checks the budget weekly
- SimpleFIN syncs transactions within 24 hours
- Budget alerts actually help prevent overspending

### Phase 3: Plugin Architecture with gRPC + WASM (Months 10-14)

**Overview:**  
Enable community developers to extend Gabija by building a gRPC-based plugin SDK with WASM runtime for secure plugin execution. Plugins communicate with core via Protocol Buffers over gRPC, running in isolated WASM sandboxes.

**Core Features:**

**Plugin Manager & WASM Runtime**
- Plugin Manager coordinates all plugin lifecycle operations
- Load and execute plugins as WASM modules in isolated sandbox
- Each plugin runs in separate WASM instance with own memory space
- gRPC server in Plugin Manager exposes core services to plugins
- gRPC client in Plugin Manager calls plugin service handlers
- Capability-based security model (plugins declare and request specific permissions)
- Memory limits per plugin (64MB default, configurable)
- CPU time limits to prevent infinite loops
- Plugin lifecycle management (install, enable, disable, uninstall, update)

**Plugin Manifest System**
```yaml
# plugin.yaml example
id: todo-list
name: Simple Todo List
version: 1.0.0
author: Gabija Team
description: Shared household task management
runtime: wasm32-wasi

# Capabilities - enforced by core at runtime
capabilities:
  data:
    - read:tasks
    - write:tasks
  events:
    - subscribe:household.member_added
    - publish:task.created
    - publish:task.completed
  http:
    - request:https://api.todoist.com/*  # External API if needed

# Dependencies
dependencies:
  core: ">=1.0.0"
  plugins: []

# Data models this plugin owns (tables prefixed with plugin ID)
data_models:
  - name: tasks
    schema: migrations/001_create_tasks.sql
  - name: task_assignments
    schema: migrations/001_create_tasks.sql

# HTTP routes (proxied by core to plugin via gRPC)
routes:
  - path: /api/tasks
    method: GET
    handler: handle_get_tasks
  - path: /api/tasks
    method: POST
    handler: handle_create_task
  - path: /api/tasks/:id
    method: PATCH
    handler: handle_update_task

# Event handlers
events:
  subscribes:
    - household.member_added
  publishes:
    - task.created
    - task.completed

# Lifecycle hooks
lifecycle:
  setup: on_setup
  teardown: on_teardown
  config_update: on_config_update

# Database migrations (run automatically on install/update)
migrations:
  - version: 001
    file: migrations/001_create_tasks.sql
  - version: 002
    file: migrations/002_add_categories.sql
```

**Event Bus**
- **MVP (Self-Hosted):** In-memory pub/sub system for inter-plugin communication
- **Managed Service:** Redis Pub/Sub for persistence and multi-server support
- Event schema validation via Protocol Buffers
- Event history stored in PostgreSQL for debugging and audit trail
- Rate limiting to prevent event spam (100 events/minute per plugin)
- Async delivery—plugins don't block on event handling

**Plugin SDK**
- **Go SDK** (Phase 3) - Primary and only supported language initially
- TypeScript/Rust SDKs (Phase 4+) - Compile to WASM, call gRPC services
- Plugin template repository with working example
- Local development environment with mock core services
- Testing utilities for unit and integration tests

**Developer Experience (Phase 3 - Must-Have)**
- CLI tool: `gabija plugin new <name>` - scaffolds plugin structure with manifest, main.go, proto files
- `gabija plugin build` - compiles Go to WASM using TinyGo
- `gabija plugin install github.com/user/plugin` - installs from GitHub releases
- `gabija plugin list/enable/disable/update` - management commands
- Comprehensive documentation with step-by-step guide and examples

**Future (Phase 4 - Nice-to-Have)**
- `gabija plugin dev` - hot reload during development
- `gabija plugin test` - integrated testing framework
- `gabija plugin validate` - lint manifest and check capabilities
- Plugin debugging tools (structured logging, performance profiling)

**Example Plugins to Build:**

1. **Simple Todo List** (Phase 3 - reference implementation)
   - Demonstrates CRUD operations via gRPC Data Service
   - Shows how to handle HTTP routes through Plugin Manager
   - Example of event publishing (task.created, task.completed)
   - Subscribes to household.member_added to initialize default tasks
   - Database migrations for tasks and task_assignments tables
   - Complete with tests and documentation
   - ~60-80 hours to build + document thoroughly

**Future Official Plugins:**

2. **Meal Planning Plugin** (Phase 4)
   - Recipe database with import from URLs
   - Weekly meal planning calendar
   - Auto-generate shopping lists from meal plans
   - Listens to finance.budget_exceeded events (suggests cheaper meals)
   - Integration with grocery APIs
   - ~120-150 hours to build

3. **Refactor Finance into Plugin** (Phase 4)
   - Move existing finance tracker code into plugin architecture
   - Proves system can handle complex, production plugin
   - No feature changes, just architectural refactor
   - Validates plugin system is production-ready
   - ~40-60 hours

**Technical Implementation:**

**Plugin Communication via gRPC + Protocol Buffers**

The plugin system uses bidirectional gRPC communication:
- **Core → Plugin**: Lifecycle hooks, HTTP request handling, event delivery
- **Plugin → Core**: Data queries, event publishing, HTTP requests to external APIs

**Core Architecture:**
```go
type PluginManager struct {
    plugins        map[string]*LoadedPlugin
    wasmRuntime    *WasmRuntime
    grpcServer     *grpc.Server
    eventBus       *EventBus
    db             *sql.DB
    capabilityEnforcer *CapabilityEnforcer
}

type LoadedPlugin struct {
    ID             string
    Manifest       *PluginManifest
    WasmInstance   *wasmtime.Instance
    GrpcClient     pb.PluginServiceClient
    Enabled        bool
    Version        string
}
```

**Key Components:**
1. **Plugin Manager** - Orchestrates plugin lifecycle, routing, and capability enforcement
2. **WASM Runtime** - Loads and executes WASM modules with memory/CPU limits
3. **gRPC Services** - Defined in Protocol Buffers, handle all plugin communication
4. **Event Bus** - Pub/sub for inter-plugin communication (in-memory or Redis)
5. **Capability Enforcer** - Validates every operation against plugin's declared capabilities

**Plugin Installation Flow:**
```bash
$ gabija plugin install github.com/gabija/todo-plugin

# Steps:
# 1. Fetch manifest.yaml and plugin.wasm from GitHub release
# 2. Validate manifest schema and capabilities
# 3. Extract to ~/.gabija/plugins/todo-plugin/
# 4. Run database migrations
# 5. Load WASM module into runtime
# 6. Initialize gRPC client to plugin
# 7. Call Setup() lifecycle hook
# 8. Register routes and event subscriptions
# ✓ Plugin is live!
```

**Security Model:**
- Plugins run in WASM sandbox (no direct filesystem/network access)
- All operations go through gRPC APIs with capability checks
- Plugin-owned tables (prefixed: `todo_tasks`, `todo_categories`)
- Automatic household scoping on all queries

**Detailed implementation examples, gRPC service definitions, and Plugin SDK usage are documented in `/docs/plugin-development-guide.md`**

**Plugin Distribution (Phase 3)**

**GitHub Releases Distribution:**
- Plugin developers publish to GitHub repositories
- Create GitHub releases with version tags (v1.0.0, v1.1.0)
- Attach compiled `.wasm` binary and `manifest.yaml` as release assets
- Users install via: `gabija plugin install github.com/username/plugin-name`
- System fetches latest release or specific version

**Plugin Repository Structure:**
```
github.com/username/todo-plugin/
├── manifest.yaml
├── main.go
├── go.mod
├── migrations/
│   ├── 001_create_tasks.sql
│   └── 002_add_categories.sql
├── README.md
└── .github/workflows/
    └── release.yml    # Auto-builds .wasm on tag push
```

**Phase 4+: Custom Plugin Registry**
- Hosted registry at plugins.gabija.app
- Web UI for browsing plugins with screenshots
- Ratings, reviews, download counts
- Security scanning and verification
- Dependency resolution
- One-click installation from web UI

**Time Estimate:** 250 hours (5 months @ 12 hrs/week)

**Breakdown:**
- Plugin Manager infrastructure: 80 hours
- gRPC/Protocol Buffers service definitions and implementation: 40 hours
- WASM runtime integration with wasmtime-go: 50 hours
- Capability enforcement system: 30 hours
- Event bus implementation (in-memory with Redis interface): 20 hours
- Plugin SDK (Go) with examples: 40 hours
- CLI tools (new/build/install/list/enable/disable): 30 hours
- Todo plugin reference implementation: 60 hours
- Documentation (PRD update, plugin guide, API reference): 40 hours

**Success Criteria:**
- You build the todo plugin from scratch using your own SDK
- Todo plugin demonstrates: CRUD operations, event pub/sub, database migrations, capability usage
- Another developer (friend) can follow docs and create a simple plugin in a weekend
- Plugin installation works smoothly from GitHub releases
- WASM sandboxing prevents unauthorized data access (test with intentionally malicious plugin)
- Finance tracker (still in core) can publish events that todo plugin receives
- Capability system successfully blocks plugins from accessing unauthorized tables/APIs
- Hot reload not required for Phase 3 (documented as Phase 4 nice-to-have)

### Phase 4: Community Beta (Months 15-18)

**Overview:**  
Prepare for public launch, attract first 100 users, iterate based on feedback.

**Deliverables:**

**Self-Hosting Support**
- Docker image on Docker Hub
- Docker Compose configuration
- Environment variable documentation
- Backup/restore instructions
- Migration guide from SQLite to PostgreSQL (if requested)

**Documentation**
- User guide (setup, basic usage, troubleshooting)
- Admin guide (deployment, configuration, updates)
- Plugin developer guide (API reference, examples, best practices)
- FAQ and common issues

**Quality & Polish**
- Fix bugs reported by beta users
- Performance optimization (target <300ms API response times)
- Mobile web UI improvements
- Accessibility audit (keyboard navigation, screen readers)

**Community Infrastructure**
- GitHub Discussions for support/feedback
- Discord server for real-time chat
- Contribution guidelines for open source contributors
- Public roadmap showing planned features

**Marketing & Launch**
- Launch blog post explaining vision
- r/selfhosted launch thread
- Show HN post on Hacker News
- Product Hunt launch (low expectations)
- Twitter/X presence with progress updates

**Paid Tier Introduction**

**Free Tier (Community Edition)**
- Core platform
- All official plugins
- Single household (up to 6 members)
- Community plugin marketplace
- 5GB file storage

**Pro Tier ($8/month or $80/year)**
- Multiple households
- Priority support (email response within 48 hours)
- Early access to new features
- 50GB file storage
- Custom plugin installation (upload your own)

**Self-Hosted (Free - AGPL License)**
- No limits
- No support (community forum only)
- Must comply with AGPL (share modifications)

**Time Estimate:** 150 hours (4 months @ 10 hrs/week)

**Success Criteria:**
- 50-100 active users (any hosting method)
- 10+ beta testers provide constructive feedback
- 2-3 community plugins appear
- 5-10 paying customers ($40-80 MRR)
- Still enjoying working on the project

---

## Technical Architecture

### System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Client Layer                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ React Web App│  │Mobile Browser│  │  CLI Tool    │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼──────────────────┼──────────────────┼──────────┘
          │                  │                  │
          └──────────────────┴──────────────────┘
                             │
                    ┌────────▼────────┐
                    │   Nginx/Caddy   │
                    │  (SSL, Routing) │
                    └────────┬────────┘
                             │
          ┌──────────────────┴──────────────────┐
          │         Go HTTP Server               │
          │  ┌──────────────────────────────┐   │
          │  │      REST API Layer          │   │
          │  │  (Auth, RBAC, Rate Limiting) │   │
          │  └──────────┬───────────────────┘   │
          │             │                        │
          │  ┌──────────▼───────────────────┐   │
          │  │    Business Logic Layer      │   │
          │  │  ┌─────────────────────────┐ │   │
          │  │  │   Plugin Manager        │ │   │
          │  │  │  - Lifecycle            │ │   │
          │  │  │  - HTTP Routing         │ │   │
          │  │  │  - Capability Enforcer  │ │   │
          │  │  └────────┬────────────────┘ │   │
          │  │           │ gRPC Services    │   │
          │  │  ┌────────▼────────────────┐ │   │
          │  │  │   Event Bus             │ │   │
          │  │  │  (In-Memory / Redis)    │ │   │
          │  │  └────────┬────────────────┘ │   │
          │  └───────────┼───────────────────┘   │
          │              │ gRPC over WASI        │
          │  ┌───────────▼───────────────────┐   │
          │  │   WASM Plugin Runtime         │   │
          │  │     (wasmtime-go)             │   │
          │  │  ┌──────┐  ┌──────┐  ┌─────┐ │   │
          │  │  │ Todo │  │Future│  │More │ │   │
          │  │  │Plugin│  │Plugin│  │ ... │ │   │
          │  │  │.wasm │  │.wasm │  │     │ │   │
          │  │  └──────┘  └──────┘  └─────┘ │   │
          │  │      ↕ gRPC (Protocol Buffers)│   │
          │  └───────────────────────────────┘   │
          └──────────────┬──────────────────────┘
                         │
          ┌──────────────▼──────────────────────┐
          │       Data Layer                    │
          │  ┌──────────────┐  ┌─────────────┐ │
          │  │  PostgreSQL  │  │File Storage │ │
          │  │  (Neon DB)   │  │   (local)   │ │
          │  │              │  │             │ │
          │  │ Core tables: │  └─────────────┘ │
          │  │ - users      │                  │
          │  │ - households │  ┌─────────────┐ │
          │  │ - plugins    │  │   Redis     │ │
          │  │ - events     │  │ (Event Bus) │ │
          │  │              │  │ (Managed)   │ │
          │  │ Plugin tables:  └─────────────┘ │
          │  │ - todo_tasks │                  │
          │  │ - todo_*     │                  │
          │  └──────────────┘                  │
          └─────────────────────────────────────┘
```

**Key Components:**
- **Plugin Manager**: Orchestrates plugin lifecycle, routes HTTP requests to plugins, enforces capabilities
- **Event Bus**: In-memory (self-hosted) or Redis (managed) for plugin communication
- **WASM Runtime**: Sandboxed execution environment for plugins
- **gRPC Layer**: Protocol Buffers-based communication between core and plugins
- **Database**: Core tables + plugin-owned tables (prefixed by plugin ID)

### Technology Stack

**Backend**
- **Language:** Go 1.21+
- **Web Framework:** Chi (routing) - better performance than Gorilla Mux
- **Database:** PostgreSQL 15+ (Neon) with pgx driver
- **WASM Runtime:** wasmtime-go for plugin execution
- **gRPC:** google.golang.org/grpc for plugin communication
- **Protocol Buffers:** protoc-gen-go for code generation from .proto files
- **Event Bus:** In-memory (self-hosted) / Redis (managed service)
- **Auth:** JWT with secure httpOnly cookies
- **API:** RESTful JSON for clients, gRPC for plugins
- **Migrations:** golang-migrate for core + plugin schema management

**Frontend**
- **Framework:** React 18 with TypeScript
- **Routing:** React Router v6
- **State Management:** React Context + hooks (no Redux initially)
- **Styling:** Tailwind CSS
- **Charts:** recharts
- **Forms:** React Hook Form
- **HTTP Client:** fetch API with wrapper

**Database Schema (Core)**
```sql
-- Users & Auth
users (id, email, password_hash, created_at, updated_at)
sessions (id, user_id, token_hash, expires_at)

-- Households
households (id, name, created_by, created_at)
household_members (household_id, user_id, role, joined_at)

-- Plugins (Core Registry)
plugins (
  id, 
  identifier,           -- "todo-list", "finance-tracker"
  name, 
  version, 
  manifest_json,        -- full manifest
  wasm_path,            -- path to .wasm file
  installed_at,
  updated_at
)
plugin_permissions (plugin_id, capability)    -- expanded from manifest
plugin_installations (
  household_id, 
  plugin_id, 
  enabled, 
  config_json,
  installed_at
)

-- Plugin-Owned Tables (prefixed by plugin_id)
-- Example for todo-list plugin:
todo_tasks (
  id,
  household_id,
  title,
  description,
  completed,
  assigned_to,        -- user_id
  due_date,
  created_at,
  updated_at
)
todo_categories (id, household_id, name, color)

-- Events (Event Bus Persistence)
events (
  id, 
  type,               -- "task.created", "household.member_added"
  source_plugin,      -- plugin that published
  payload_json, 
  created_at,
  processed_at        -- null if pending
)
event_subscriptions (
  plugin_id, 
  event_pattern       -- "task.*", "household.member_added"
)

-- Migrations Tracking
schema_migrations (
  plugin_id,          -- "core" for core migrations
  version,
  applied_at
)

-- Audit
audit_logs (
  id, 
  user_id, 
  plugin_id,          -- which plugin performed action
  action, 
  resource_type, 
  resource_id, 
  metadata_json, 
  created_at
)
```

**Infrastructure**
- **Hosting:** Render or Railway (Phase 1-3), then Docker for self-hosting
- **SSL:** Automatic HTTPS via platform (Render/Railway) or Caddy
- **File Storage:** Local filesystem initially, S3-compatible later
- **Backups:** pg_dump scheduled daily, retained 30 days
- **Monitoring:** Simple logging to stdout, Sentry for error tracking (optional)

### Security Model

**Authentication**
- Password hashing: bcrypt with cost factor 12
- Session tokens: cryptographically random, 32 bytes
- Token storage: httpOnly, secure, sameSite cookies
- Session expiry: 7 days, refresh on activity

**Authorization**
- Household-level RBAC (admin, member, viewer)
- Plugin permissions checked before data access
- All sensitive operations logged to audit table

**Plugin Security**

**WASM Sandboxing:**
- Plugins run in WASM sandbox with NO direct system access
- No filesystem access (except via WASI with restricted paths)
- No network access (must use CoreHttpService with capability checks)
- No access to other plugins' memory or data
- Memory limit: 64MB per plugin (configurable)
- CPU time limit: 1M instructions per request (prevents infinite loops)

**Capability-Based Permissions:**
- Plugins declare required capabilities in manifest.yaml
- Core enforces capabilities at runtime before every operation
- Capabilities are granular: `data.read:tasks`, `http.request:https://api.example.com/*`
- Users review and approve capabilities during plugin installation
- Plugins cannot escalate privileges at runtime

**Database Access Control:**
- Plugins can ONLY access tables they own (prefixed with plugin_id: `todo_tasks`)
- All queries automatically scoped to current household_id via RequestContext
- Row-level security enforced by Plugin Manager
- No raw SQL injection risk (parameterized queries via gRPC)
- Plugin cannot query core tables or other plugins' tables

**gRPC Transport Security:**
- In-process gRPC (Unix domain sockets or shared memory)
- No network exposure of plugin gRPC services
- TLS optional for multi-server deployments (Phase 4+)

**Audit Logging:**
- All plugin operations logged to audit_logs table
- Failed capability checks logged with full details
- Users can review plugin activity in settings UI

**Example Capability Enforcement:**
```go
// Plugin tries to access data
func (pm *PluginManager) handleDataQuery(
    ctx context.Context, 
    pluginID string, 
    req *pb.QueryRequest,
) (*pb.QueryResponse, error) {
    // Extract table name from SQL
    table := pm.parser.ExtractTable(req.Sql)
    
    // Check capability
    if !pm.hasCapability(pluginID, "data.read", table) {
        pm.auditLogger.LogDenied(pluginID, "data.read", table)
        return nil, status.Error(codes.PermissionDenied, 
            fmt.Sprintf("plugin lacks data.read:%s capability", table))
    }
    
    // Enforce household scoping
    scopedSQL := pm.addHouseholdFilter(req.Sql, req.Context.HouseholdId)
    
    // Execute query
    return pm.db.Query(scopedSQL, req.Params)
}
```

**Security Comparison vs. Neovim/Home Assistant:**

| Security Feature | Neovim | Home Assistant | Gabija |
|-----------------|---------|----------------|--------|
| Sandboxing | ❌ None | ❌ None | ✅ WASM |
| Capability System | ❌ No | ❌ No | ✅ Yes |
| Database Isolation | N/A | ❌ Shared | ✅ Prefixed tables |
| Network Restrictions | ❌ No | ❌ No | ✅ Capability-based |
| Audit Logging | ❌ No | ⚠️ Limited | ✅ Comprehensive |

**Trust Model:**
- **Phase 3:** Trust-based with manual vetting of official plugins
- **Phase 4:** Signature verification for community plugins
- **Future:** Automated security scanning (static analysis of WASM bytecode)

**Data Protection**
- HTTPS everywhere (no plain HTTP in production)
- Database passwords in environment variables
- No sensitive data in logs
- SQL injection prevention via parameterized queries
- XSS prevention via React's default escaping

### Performance Targets

**API Response Times**
- Simple reads (user profile, account list): <100ms p95
- Complex queries (transaction history with filters): <300ms p95
- SimpleFIN webhook processing: <500ms p95
- Plugin event publishing: <50ms p95

**Frontend Performance**
- Initial page load: <2s on 3G connection
- Time to interactive: <3s
- Navigation between pages: <200ms

**Scalability**
- Single instance should handle 1,000 active households
- Database queries optimized with indexes
- Plugin WASM instances cached in memory
- Static assets served with long cache headers

---

## User Experience

### Key User Flows

**1. First-Time Setup**
```
1. Visit kojin.app (or self-hosted URL)
2. Click "Get Started"
3. Enter email, password, household name
4. Receive email verification link
5. Click link, account activated
6. See dashboard with "Add Your First Plugin" prompt
7. Click "Enable Finance Plugin"
8. Walk through SimpleFIN connection flow
9. See transactions syncing in real-time
```

**2. Daily Finance Check**
```
1. Open kojin.app on phone (mobile browser)
2. Auto-login via saved session
3. See dashboard with "Budget Status" widget
   - "You've spent $450 of $600 on groceries this month"
   - "Warning: Dining budget at 85%"
4. Tap "View Transactions"
5. See today's transactions auto-synced from banks
6. Tap transaction to categorize or split
7. Save and return to dashboard
```

**3. Inviting Family Member**
```
1. Go to Settings → Household Members
2. Click "Invite Member"
3. Enter partner's email
4. Choose role (Member or Admin)
5. Partner receives email with invite link
6. Partner clicks link, creates account
7. Partner automatically added to household
8. Partner sees same finance data (with appropriate permissions)
```

### UI/UX Principles

**Simplicity**
- Mobile-first design (most users will check on phones)
- Minimal navigation depth (3 clicks to any feature)
- Progressive disclosure (show simple view, hide advanced options)
- Clear CTAs (one primary action per page)

**Familiarity**
- Standard patterns (hamburger menu, tabs, modals)
- Conventional icons ($ for finance, 🍳 for recipes)
- Similar to tools users already know (Mint, YNAB, Notion)

**Speed**
- Optimistic UI updates (assume API calls succeed)
- Skeleton screens while loading
- Instant navigation with client-side routing
- Background sync (no "waiting for data" spinners)

**Accessibility**
- Keyboard navigation for all actions
- Screen reader support (semantic HTML, ARIA labels)
- High contrast mode support
- Touch targets minimum 44×44px

---

## Go-to-Market Strategy

### Phase 1: Personal Use (Months 1-9)
**Goal:** Build something you actually use

- No marketing, no users, just development
- Share progress on Twitter/X occasionally
- Build in public, document learnings in blog posts
- Goal: Validate you'll stick with the project

### Phase 2: Friends & Family Beta (Months 10-14)
**Goal:** 10-20 trusted testers

**Outreach:**
- Ask 5 friends to try the finance plugin
- Post in private Discord/Slack communities
- Offer to help with setup in exchange for feedback

**Feedback Loop:**
- Weekly check-ins with active testers
- Quick bug fixes (within 48 hours)
- Feature requests logged but not immediately built

### Phase 3: Public Beta (Months 15-18)
**Goal:** 100-200 active users

**Launch Channels:**
1. **r/selfhosted** (primary audience)
   - Title: "Gabija: Self-hosted family management with extensible plugins (AGPL-3.0)"
   - Emphasize: Privacy, data ownership, WASM security model
   - Offer: Docker image + setup help

2. **Hacker News Show HN**
   - Title: "Show HN: Gabija – Home Assistant for family life"
   - Emphasize: Technical novelty (WASM plugins, event bus)
   - Include: GitHub repo, live demo instance

3. **Product Hunt** (lower priority)
   - Position as "Notion for home management"
   - Emphasize: Extensibility, community plugins

4. **Indie Hackers / Reddit r/SideProject**
   - Share journey as solo dev
   - Technical deep-dives on plugin architecture

**Content Marketing:**
- Blog post: "Why I built Gabija instead of using Cozi"
- Blog post: "Building a secure plugin system with WASM and gRPC" ⭐ Technical deep-dive
- Blog post: "gRPC for plugin architectures: Lessons learned"
- Blog post: "Self-hosting for families: A practical guide"
- YouTube video: "Gabija setup in 10 minutes"
- YouTube video: "Building your first Gabija plugin" (live coding todo plugin)

### Pricing Strategy

**Free Tier (Loss Leader)**
- Purpose: Grow user base, get feedback
- Limit: Single household, 5GB storage
- Expectation: 80% of users stay here

**Pro Tier ($8/month or $80/year)**
- Purpose: Sustainable revenue for hosting costs
- Target: Power users who want multiple households or storage
- Expectation: 10-20% conversion rate

**Self-Hosted (Free Forever)**
- Purpose: Community evangelism, developer adoption
- Benefit: These users contribute plugins, report bugs, spread word
- Monetization: Some will eventually want managed hosting

**Enterprise (Future - $500-2000/year)**
- Purpose: B2B revenue (property managers, co-housing communities)
- Includes: White-label, SSO, priority support
- Not a focus until 1,000+ users

### Revenue Projections (Conservative)

**Month 12:**
- 50 total users
- 0 paying (no paid tier yet)
- $0 MRR
- Costs: $20/month hosting

**Month 18:**
- 200 total users
- 20 paying Pro ($8/month)
- $160 MRR
- Costs: $50/month hosting + $100/month SimpleFIN
- Net: ~$10 MRR (barely profitable)

**Month 24:**
- 1,000 total users
- 100 paying Pro
- $800 MRR
- Costs: $200/month hosting + $300/month SimpleFIN
- Net: ~$300 MRR (hobby income!)

**Note:** These are OPTIMISTIC. Real growth may be slower. That's okay—this is a hobby, not a startup.

---

## Risk Assessment

### Technical Risks

**High Risk:**
1. **WASM + gRPC performance overhead**
   - Risk: Crossing WASM boundary + gRPC serialization adds latency
   - Mitigation: 
     - Batch operations where possible (fetch multiple rows in one call)
     - Use streaming gRPC for events (avoid polling)
     - Profile early in Phase 3, optimize hot paths
     - In-process gRPC (Unix sockets) for minimal overhead
   - Fallback: If too slow, consider in-process plugins (lose sandboxing benefit)
   - **Target:** <50ms overhead per gRPC call
   - **Test by Month 10**

2. **WASM toolchain maturity for Go**
   - Risk: TinyGo (Go→WASM compiler) has limitations (limited stdlib, no reflect)
   - Mitigation: 
     - Test TinyGo thoroughly in Phase 3 with todo plugin
     - Document TinyGo limitations clearly for plugin developers
     - Provide working examples and templates that work around limitations
   - Fallback: Support Rust plugins (more mature WASM tooling) in Phase 4
   - **Test by Month 10**

3. **SimpleFIN dependency** (unchanged)
   - Mitigation: Abstract behind interface, support manual entry
   - Fallback: Build Plaid integration or scraping as alternative

**Medium Risk:**
1. **Plugin migration complexity**
   - Risk: Database migrations across plugin versions can fail or corrupt data
   - Mitigation:
     - Require migration testing in plugin dev workflow
     - Provide rollback mechanism (track migration versions)
     - Test migrations with realistic data volumes
     - Validate migration syntax before applying
   - Fallback: Manual migration with SQL scripts and documentation

2. **Event bus scalability**
   - Risk: In-memory event bus doesn't scale to multi-server deployments
   - Mitigation: 
     - Design event bus interface abstraction from day one
     - Easy swap to Redis Pub/Sub in Phase 4
     - Test with high event volumes (1000+ events/min)
   - Fallback: Redis Pub/Sub for managed service (already planned)

3. **Plugin ecosystem cold start** (unchanged)
   - Mitigation: Build 2-3 official plugins to demonstrate value
   - Fallback: Platform still useful with just official plugins

**Low Risk:**
1. **Go/React learning curve** (unchanged)
   - Mitigation: You're a staff engineer, these are mainstream tech
   - Fallback: N/A - this risk is minimal

2. **gRPC learning curve**
   - Risk: Minimal - gRPC is well-documented and widely adopted
   - Mitigation: Extensive examples, protobuf is straightforward
   - Fallback: N/A - you're a staff engineer

**Mitigated Risks (vs WASM-only approach):**
- ✅ Language support: gRPC enables multi-language plugins in future
- ✅ Debugging: Can debug Go plugins before compiling to WASM
- ✅ Community familiarity: gRPC more common than WASM host functions
- ✅ Protocol evolution: Protobuf supports backward compatibility

### Business Risks

**High Risk:**
1. **Nobody wants this**
   - Mitigation: Build for yourself first, validate with friends
   - Fallback: Still learned valuable skills, built cool portfolio piece

2. **Can't compete with free alternatives**
   - Mitigation: Focus on interoperability as unique differentiator
   - Fallback: Niche down (just finance plugin, become best-in-class)

**Medium Risk:**
1. **Burnout after 6-12 months**
   - Mitigation: Hobby pace, no artificial deadlines, take breaks
   - Fallback: Open source it, let community continue

2. **Competitor launches similar product**
   - Mitigation: WASM security model is genuinely hard to replicate
   - Fallback: Differentiate on UX, community, or niche features

**Low Risk:**
1. **Hosting costs too high**
   - Mitigation: Push self-hosting, users pay own costs
   - Fallback: Increase Pro tier price or add usage limits

### Mitigation Strategies

**Technical:**
- Keep dependencies minimal (fewer things to maintain)
- Write tests for critical paths (auth, finance sync)
- Document architecture decisions (ADRs) for future reference
- Have rollback plan for every deployment

**Business:**
- Launch self-hosted option early (reduces hosting costs)
- Build community before launching paid tiers (trust first)
- Focus on retention over acquisition (10 happy users > 100 lukewarm)
- Set clear boundaries (no 24/7 support expectations)

**Personal:**
- Track hours weekly, adjust pace if exceeding 12 hrs/week
- Take full weeks off when needed (this is a marathon)
- Celebrate milestones (deployed to prod! first external user!)
- Have exit criteria (if not fun by month 9, pause project)

---

## Development Workflow

### Task Management
- **GitHub Projects** for roadmap and sprint planning
- **Issues** for bugs and feature requests
- **Milestones** for each phase (1-4)
- **Labels:** bug, feature, documentation, plugin, good-first-issue

### Branching Strategy
- `main` branch: production-ready code
- `develop` branch: integration branch
- Feature branches: `feature/finance-simplefin-integration`
- Hotfix branches: `hotfix/auth-token-expiry`

### Code Quality
- **Tests:** Aim for 60%+ coverage on business logic
- **Linting:** golangci-lint for Go, ESLint for TypeScript
- **Formatting:** gofmt, Prettier (auto-format on save)
- **PR Reviews:** Self-review using GitHub PR template

### Deployment
- **CI/CD:** GitHub Actions
  - Run tests on every PR
  - Deploy to staging on merge to `develop`
  - Deploy to production on merge to `main`
- **Database Migrations:** Run automatically on deployment
- **Rollback Plan:** Keep previous 3 releases as Git tags

### Monitoring
- **Logs:** Structured JSON logging to stdout
- **Errors:** Sentry (free tier) for exception tracking
- **Metrics:** Simple `/health` endpoint, no complex monitoring in v1
- **Uptime:** UptimeRobot (free tier) for 5-minute checks

---

## Success Metrics & KPIs

### Product Metrics

**Activation (Month 1)**
- Account created → Household created → Finance plugin enabled → Bank connected
- Target: 70% of signups complete activation within 7 days

**Engagement (Month 1-18)**
- Daily Active Users / Monthly Active Users (DAU/MAU ratio)
- Target: 20%+ (indicates users check finances 6+ days/month)
- Median session duration
- Target: 3+ minutes (users are doing real work, not bouncing)

**Retention (Month 3+)**
- Day 7 retention: 40%+ (user returns after first week)
- Day 30 retention: 20%+ (user still active after a month)
- Month 3 retention: 10%+ (user becomes habitual)

**Plugin Ecosystem (Month 12+)**
- Community plugins published: 3+ by month 18
- Plugin installation rate: 30%+ of users enable at least one community plugin
- Plugin developer satisfaction: Survey score 7+/10

### Business Metrics

**Revenue (Month 15+)**
- Free to Pro conversion: 10-20% (industry standard for dev tools)
- Monthly churn: <5% (users stay because they need it)
- Customer Lifetime Value (LTV): $400+ (5+ years at $8/month)

**Community (Month 9+)**
- GitHub stars: 100+ by month 18 (indicates interest)
- Discord members: 50+ active contributors
- Blog post views: 1,000+ monthly (SEO working)

### Personal Metrics (Most Important!)

**Enjoyment**
- Still excited to work on project: Yes/No (check monthly)
- Learning goals met: Rate 1-10 every 3 months
- Work-life balance maintained: Averaging <12 hrs/week

**Sustainability**
- Weeks with zero commits: <4 per quarter (staying consistent)
- Burnout risk: Self-assessment 1-10 (if >7, take break)
- Financial stress: Costs covered by revenue or savings

---

## Future Roadmap (Post-Month 18)

### Potential Plugins (Community or Official)

**Home Management**
- Shared calendar with family events
- Chore assignment with gamification
- Home maintenance tracker (HVAC filters, lawn care)
- Pet care (vet appointments, feeding schedules)

**Food & Kitchen**
- Meal planning with recipe database
- Pantry/freezer inventory with expiration tracking
- Grocery shopping lists with aisle sorting
- Nutrition tracking per family member

**Travel & Events**
- Trip planning with itineraries
- Expense splitting for travel
- Packing lists
- Travel document storage (passports, visas)

**Education**
- School calendar integration
- Homework tracking
- Grade monitoring
- Extracurricular activity scheduling

**Health**
- Shared medical records
- Medication reminders
- Doctor appointment tracking
- Insurance information repository

### Platform Enhancements

**Year 2 (Months 19-30)**
- Native mobile apps (React Native or Capacitor)
- OpenID Connect / SSO support
- Plugin marketplace with ratings/reviews (hosted at plugins.gabija.app)
- Multi-language plugin SDKs (Rust, TypeScript via AssemblyScript)
- Advanced RBAC (custom roles)
- Multi-language support (i18n)
- Hot reload for plugins (wasmtime supports this)
- Plugin debugging tools (integrated with SDK)

**Year 3 (Months 31-42)**
- API for third-party integrations
- Webhooks for external services
- White-label option for enterprises
- Plugin monetization (paid plugins)
- Advanced analytics dashboard

### Exit Scenarios

**Scenario 1: Sustainable Hobby (Most Likely)**
- 1,000-5,000 users
- $1-3K MRR
- Covers costs + provides hobby income
- Work 5-10 hours/week on maintenance + features
- Outcome: Happy with this indefinitely

**Scenario 2: Acqui-Hire**
- 10K+ users, strong plugin ecosystem
- Larger company (Home Assistant, Notion) acquires for talent/tech
- Outcome: Join their team, Gabija lives on as open source

**Scenario 3: Full-Time Startup (Unlikely)**
- 50K+ users, $20K+ MRR
- Investor interest or bootstrapped profitability
- Quit day job, hire team, scale up
- Outcome: Becomes primary focus

**Scenario 4: Graceful Shutdown**
- Project not gaining traction after 18 months
- No longer enjoying it
- Open source fully, hand off to community
- Outcome: Great learning experience, move on

---

## Appendix

### Glossary

**Terms**
- **Household:** Group of users sharing data (family, roommates, etc.)
- **Plugin:** Extendable module that adds features to Gabija, runs in WASM sandbox
- **Event Bus:** Pub/sub system allowing plugins to communicate asynchronously
- **WASM:** WebAssembly, a binary format for secure, sandboxed plugin execution
- **gRPC:** High-performance RPC framework using Protocol Buffers for plugin-to-core communication
- **Protocol Buffers:** Language-neutral serialization format for structured data (protobuf)
- **Capability:** Permission that a plugin must declare in manifest to access resources (data, HTTP, events)
- **Plugin Manager:** Core service that orchestrates plugin lifecycle, routing, and security
- **SimpleFIN:** Third-party service for bank account aggregation
- **RBAC:** Role-Based Access Control for permissions
- **TinyGo:** Go compiler that targets WebAssembly
- **wasmtime:** Fast and secure WebAssembly runtime

### References

**Technical Inspiration**
- Home Assistant: Plugin architecture, community-first approach
- Obsidian: Local-first, extensible with community plugins
- VSCode: WASM plugin sandbox, extension marketplace

**Business Models**
- Home Assistant (Nabu Casa): Open source + managed hosting
- Ghost: Self-hosted + Pro managed service
- GitLab: Open core + enterprise features

### License

**AGPL-3.0 License**
- Core platform is free and open source
- Modifications must be shared (prevents proprietary forks)
- Commercial license available for embedding without AGPL restrictions

---

## Document History

**v0.1 - December 2024**
- Initial PRD for part-time solo development
- 18-month roadmap with realistic time estimates
- Focus on finance plugin as first killer feature
- gRPC + WASM plugin architecture as differentiator
- Go-only SDK for Phase 3 (MVP)
- Todo plugin as Phase 3 reference implementation

---

**END OF DOCUMENT**

This PRD is a living document and will be updated as the project evolves. For the latest version, see: [https://github.com/dvigh8/gabija.git](Gabija)