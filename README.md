# Going Documentation

Documentation for the Going web framework - a fullstack Go framework.

## Getting Started

Going is built on top of Go's standard library and provides an opinionated, convention-over-configuration approach to building Go web applications.

## Documentation

| File | Description |
|------|-------------|
| [Authentication](going_docs/authentication.md) | JWT-based login, logout, and user sessions |
| [CLI Commands](going_docs/cli.md) | Code generation commands (`make:controller`, `make:model`, etc.) |
| [Collections](going_docs/collections.md) | Generic `Collection[T]` type |
| [Concerns / Mixins](going_docs/concerns.md) | Composition patterns and polymorphic associations |
| [DI Container](going_docs/container.md) | Dependency injection for controllers |
| [Database / ORM](going_docs/db.md) | ActiveRecord ORM, queries, pagination |
| [Jobs](going_docs/jobs.md) | Background job scheduler |
| [Middleware](going_docs/middleware.md) | HTTP middleware and middleware groups |
| [Migration](going_docs/migration.md) | Database migrations (SQLite, PostgreSQL) |
| [Relationships](going_docs/relationships.md) | BelongsTo, HasOne, HasMany, BelongsToMany, Polymorphic |
| [Request](going_docs/request.md) | Request handling and reflection-based parameter injection |
| [Response](going_docs/response.md) | Response builder, JSON, redirects, flash data, error responses |
| [Routing](going_docs/routing.md) | Router, route groups, DI, controller lifecycle |
| [Schema Builder](going_docs/schema.md) | Blueprint-based table definitions |
| [Security](going_docs/security.md) | JWT tokens, bcrypt password hashing, cookies |
| [WebSocket](going_docs/socket.md) | Hub-based WebSocket architecture with rooms |
| [Validation](going_docs/validation.md) | Input validation with custom error messages |
| [Views](going_docs/view.md) | Server-side templates and Inertia.js |

## Architecture

Going follows a Laravel/Rails-inspired MVC architecture:

1. **Entry point**: `app.NewApp()` creates the app, loads env, initializes the scheduler
2. **Routing**: `customrouter.NewMyRouter()` with `Group()`, `Middleware()`, `Get/Post/etc()`
3. **DI Container**: `customrouter.NewContainer()` with `Register()` and `UseContainer()`
4. **Controllers**: Structs with `Loader()` for DI, action methods with automatic parameter injection
5. **ORM**: `ActiveRecord[T]` models with chainable queries and relationships
6. **Validation**: `go-playground/validator` with `rules.json` for custom messages
7. **Response**: `Response.PrintJson()`, `Response.Redirect()`, `Fail.StatusNotFound()`
8. **Auth**: Cookie-based JWT with bcrypt password hashing
9. **WebSocket**: Hub-based architecture with rooms and event routing
10. **Background Jobs**: Ticker-based scheduler with overlap prevention
11. **CLI**: Cobra-based `make:` commands for code generation
12. **Views**: Dual view system - Inertia.js or server-side Go templates
