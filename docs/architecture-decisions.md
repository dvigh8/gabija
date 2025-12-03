# Architecture Decisions for Gabija Plugin System

**Version:** 0.1  
**Date:** December 2024  
**Status:** Planning Phase

---

## Decision Summary

This document captures the key architectural decisions made for the Gabija plugin system, transitioning from a pure WASM approach to a gRPC + WASM hybrid architecture.

---

## Decision 1: gRPC + WASM Instead of WASM-Only

### Context
Initially considered pure WASM with host functions for plugin communication. After researching Neovim and Home Assistant plugin systems, we identified opportunities to improve developer experience while maintaining security.

### Decision
**Use gRPC with Protocol Buffers for plugin-to-core communication, with plugins running in WASM sandbox.**

### Rationale
- **Better developer experience**: gRPC is widely understood, good tooling
- **Language-agnostic future**: Can support Rust, TypeScript, C++ plugins later
- **Easier debugging**: Standard gRPC debugging tools work
- **Protocol evolution**: Protobuf supports backward compatibility
- **Security maintained**: WASM still provides sandboxing

### Trade-offs
- **Performance overhead**: gRPC serialization + WASM boundary crossing (~50ms target)
- **Complexity**: Need to manage both gRPC and WASM runtimes
- **Binary size**: Larger than pure WASM with host functions

---

## Decision 2: Go-Only SDK for Phase 3 (MVP)

### Context
Could support multiple languages from day one (Go, Rust, TypeScript/AssemblyScript).

### Decision
**Go-only SDK for Phase 3, multi-language support in Phase 4+**

### Rationale
- **Faster MVP**: Focus on one SDK, get it right
- **TinyGo validation**: Test Go→WASM toolchain thoroughly first
- **Lower support burden**: One language during beta
- **Staff engineer familiar with Go**

### Future
- Phase 4: Add Rust SDK (more mature WASM tooling)
- Phase 4+: TypeScript via AssemblyScript

---

## Decision 3: API-Only Frontend (Phase 3)

### Context
Plugins need to provide UI. Options:
- API-only (plugin returns JSON, core renders)
- Dynamic React components (npm packages)
- Server-side rendering (HTML from plugin)

### Decision
**API-only for Phase 3**

### Rationale
- **Simpler**: No dynamic code loading
- **Secure**: No eval() or dynamic imports
- **Fast**: Direct JSON rendering
- **Acceptable scope**: Core can bundle plugin UIs for Phase 3

### Trade-offs
- **Less modular**: Core needs updates for new plugin UIs
- **Transition needed**: Move to dynamic components in Phase 4

---

## Decision 4: Plugin-Owned Tables with Prefixing

### Context
How should plugins store data?
- Direct database access
- API Gateway pattern
- Plugin-owned tables
- Shared tables with namespacing

### Decision
**Plugin-owned tables with plugin_id prefix (e.g., `todo_tasks`, `todo_categories`)**

### Rationale
- **Clear ownership**: Easy to see which plugin owns which tables
- **Easy uninstall**: DROP tables with prefix
- **Isolation**: Plugins can't query other plugins' tables
- **Simple migrations**: Each plugin manages its own schema

### Implementation
```sql
-- Todo plugin creates:
CREATE TABLE todo_tasks (...);
CREATE TABLE todo_categories (...);

-- Finance plugin creates:
CREATE TABLE finance_transactions (...);
CREATE TABLE finance_accounts (...);
```

---

## Decision 5: In-Memory Event Bus (MVP), Redis (Managed)

### Context
Event bus needed for inter-plugin communication. Options:
- In-memory (simple, not persistent)
- PostgreSQL LISTEN/NOTIFY
- Redis Pub/Sub
- Kafka/NATS (overkill)

### Decision
**In-memory for self-hosted (Phase 3), Redis for managed service (Phase 4)**

### Rationale
- **Self-hosters prefer simple**: No extra dependencies
- **Managed service scales**: Redis for multi-server
- **Interface abstraction**: Easy to swap implementations
- **Events still logged**: PostgreSQL stores events for audit

---

## Decision 6: Automatic Migrations (Developer Responsibility)

### Context
How to handle plugin schema updates?

### Decision
**Automatic migrations on plugin update, developer provides migration files, never a user problem**

### Rationale
- **Good UX**: Users don't run manual SQL scripts
- **Follows Home Assistant pattern**: Proven approach
- **Safety**: Migrations validated before applying
- **Rollback**: Track migration versions for rollback

### Implementation
```
plugins/todo-plugin/migrations/
├── 001_create_tasks.sql
├── 002_add_categories.sql
└── 003_add_due_date.sql
```

---

## Decision 7: Phase 3 Scope: Plugin System + Todo Plugin

### Context
Phase 3 could:
- Build plugin system only
- Build plugin system + refactor finance into plugin
- Build plugin system + simple new plugin

### Decision
**Build plugin system + simple todo plugin, keep finance in core**

### Rationale
- **Lower risk**: Don't touch working finance feature
- **Proves system**: Todo plugin validates architecture
- **Reference implementation**: Developers can learn from todo plugin
- **Refactor later**: Move finance to plugin in Phase 4 when system stable

---

## Decision 8: Context-Based Auth in gRPC Metadata

### Context
How should plugins know which user is making a request?

### Decision
**Core includes user_id and household_id in every gRPC request via RequestContext**

### Rationale
- **Secure**: Plugin can't forge identity
- **Simple**: Plugin just reads context
- **Standard pattern**: gRPC metadata is designed for this
- **Automatic scoping**: All queries scoped to household

### Implementation
```go
message RequestContext {
  string user_id = 1;
  string household_id = 2;
  string plugin_id = 3;  // Auto-added by core
}
```

---

## Decision 9: Must-Have Dev Tools Only (Phase 3)

### Context
Developer tooling scope for Phase 3.

### Decision
**Must-have tools only, nice-to-have documented for Phase 4**

### Must-Have (Phase 3)
- `gabija plugin new <name>` - scaffolding
- `gabija plugin build` - compile to WASM
- `gabija plugin install github.com/user/plugin`
- `gabija plugin list/enable/disable/update`

### Nice-to-Have (Phase 4)
- `gabija plugin dev` - hot reload
- `gabija plugin test` - testing framework
- `gabija plugin validate` - linting
- Debugging tools

### Rationale
- **Ship faster**: Focus on core functionality
- **Learn from users**: See which tools are most needed
- **Sustainable pace**: 250 hours is already ambitious

---

## Decision 10: GitHub Releases for Plugin Distribution (Phase 3)

### Context
Plugin distribution options:
- GitHub Releases (decentralized)
- Custom registry (centralized)
- npm/crates.io-style registry

### Decision
**GitHub Releases for Phase 3, custom registry for Phase 4+**

### Rationale
- **Zero infrastructure**: Use GitHub's CDN
- **Familiar to developers**: Same as Neovim plugins
- **Easy CI/CD**: GitHub Actions builds and releases
- **Decentralized**: No single point of failure

### Implementation
```bash
# Users install from GitHub
$ gabija plugin install github.com/gabija/todo-plugin

# Fetches from:
# https://github.com/gabija/todo-plugin/releases/latest
```

---

## Key Architecture Diagrams

All diagrams included in:
- `/docs/plugin-development-guide.md` - Complete guide with flow diagrams
- `/prd.md` - System architecture diagram updated

---

## Next Steps

1. **Begin Phase 1** (Months 1-4): Core platform foundation
2. **Begin Phase 2** (Months 5-9): Finance tracker
3. **Begin Phase 3** (Months 10-14): Plugin system implementation
   - Implement Plugin Manager
   - Implement gRPC services
   - Integrate wasmtime-go
   - Build capability enforcement
   - Build event bus
   - Create Plugin SDK (Go)
   - Build CLI tools
   - Build todo plugin reference implementation
   - Write comprehensive documentation

---

## References

- [Neovim Plugin System Analysis](../prd.md#appendix-neovim-analysis)
- [Home Assistant Integration System Analysis](../prd.md#appendix-homeassistant-analysis)
- [Plugin Development Guide](./plugin-development-guide.md)
- [gRPC Documentation](https://grpc.io/docs/languages/go/)
- [wasmtime-go](https://github.com/bytecodealliance/wasmtime-go)
- [TinyGo](https://tinygo.org/)
