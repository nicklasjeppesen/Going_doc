# Application layers

- [Introduction](#introduction)
- [Folder layout](#folder-layout)
- [The layers](#the-layers)
- [Import direction](#import-direction)
- [Binding interfaces in the container](#binding-interfaces-in-the-container)
- [Moving an area into the layers](#moving-an-area-into-the-layers)


## Introduction

A small Going app gets far with controllers and ActiveRecord models. When an area grows business rules and side effects (activity logs, notifications, the next repeat of a task), it pays to split it into layers. This page describes the layout Kontoret uses for its todo area. It follows 37signals' "Vanilla Rails is plenty" (rich models with concerns) plus DDD repositories and commands.

The layers are optional and can be adopted one area at a time. Areas that haven't moved yet keep using a plain service.


## Folder layout

```
internal/app/
├── db/                  ORM rows and their relationships, no behaviour
├── mixins/              concerns shared by models (Fileable, Commentable)
├── model/               rich models: a db row + mixins + behaviour, and the domain events
│   ├── todo.go
│   └── todo_events.go
├── domain/
│   └── todo/            the area's read types, pure rules, the Repository interface and the commands
├── infrastructure/      raw SQL behind the repositories, files on disk
├── events/
│   └── listeners/       what happens after an event, all registered in listeners.Register
├── providers/
│   └── container/       the DI container: binds each interface to its implementation
└── http/controller/     actions that read the request, build a command and render
```


## The layers

### `db`: rows and relationships

Going ORM rows (`db.Todo`, `db.TodoList`, …) with their relationships (`Checklist()`, `Files()`). They know how a row is stored and nothing else. Simple create, read, update and delete go through them. See [Database / ORM](db.md) and [Relationships](relationships.md).

### `mixins`: shared concerns

Behaviour several models share, such as `Fileable` (files as a delegated type) and `Commentable`. Recording domain events needs no mixin: every ActiveRecord row has `Record`/`ReleaseEvents` built in. See [Concerns / Mixins](concerns.md).

### `model`: row + mixins + behaviour + events

A rich model embeds a row and its mixins and adds the business methods:

```go
type Todo struct {
    *db.Todo
    mixins.Fileable[*db.Todo]
    mixins.Commentable[*db.Todo]
}
```

A method (`Complete`, `Trash`, `Rename`, …) changes the row in memory, records a domain event with the row's built-in `Record`, and returns false (recording nothing) when there was nothing to change. It never saves and never publishes. The events sit next to the model (`model/todo_events.go`), see [Where to put the events](events.md#where-to-put-the-events).

### `domain/<area>`: commands and the Repository interface

The area's package holds:

- **The `Repository` interface**: what the commands and controllers read and save with (`Find`, `Save`, `Add`, the queries other pages show). It lists only methods that are used, so a test can fake it.
- **Commands**: one small struct per task with an `Execute`, such as `Create`, `Complete`, `Trash` and `Move`. A command loads the model through the repository, lets the model do the work, saves, and publishes `t.ReleaseEvents()`. See [Recording events on the model](events.md#recording-events-on-the-model).
- **Read types and pure rules** that need no database.

The domain only knows interfaces (`Repository`, `events.Publisher` from `super/events`), never the SQL behind them.

### `infrastructure`: raw SQL and disk

The implementations of the domain's interfaces. `infrastructure.TodoRepository` implements `todo.Repository` with `database/sql` for what the ORM can't do (joins, counts, upserts, bulk copies), and `infrastructure.Uploads` implements the file domain's `Storage` on disk.

### `events/listeners`: side effects

Each listener reacts to one event and does one thing: writes the activity log, sends a push, creates the next repeat. They are all wired in one function, so there is one place to read:

```go
// Register wires every listener to the dispatcher.
func Register(dispatcher *events.Dispatcher, todos todo.Repository) {
    registerTodo(dispatcher, todos)
    registerFile(dispatcher, todos)
}
```

Sync listeners run before the response (what the page has to show right away), async ones after it (push). See [Sync and async listeners](events.md#sync-and-async-listeners).

### Controllers

A controller only handles HTTP. It reads the request, builds a command with what its `Loader` received, and renders or redirects:

```go
func (c *TodoDoneController) Update(r Request, id int) Result {
    if err := (todo.Toggle{Todos: c.todos, Events: c.events}).Execute(r.R.Context(), int64(id)); err != nil {
        return NotFoundOr(err)
    }
    return Response.Back()
}
```


## Import direction

Imports only go one way, so there are no cycles:

```
events ◀── model (db, mixins) ◀── domain/<area> ◀── infrastructure ◀── events/listeners ◀── providers/container ◀── controllers
```

The domain never imports `infrastructure`, and a model never imports the domain.


## Binding interfaces in the container

The container registers **interfaces**, never concrete types. A controller asks for the interface in its `Loader`, so the implementation can be swapped, for example with a fake in a test. See [DI Container](container.md).

```go
// The implementations fulfil the interfaces (checked when the package builds).
var (
    _ todo.Repository  = (*infrastructure.TodoRepository)(nil)
    _ events.Publisher = (*events.Dispatcher)(nil)
    _ file.Storage     = (*infrastructure.Uploads)(nil)
)

func GetContainer() *customrouter.Container {
    container := customrouter.NewContainer()

    // The domain's repositories → the infrastructure's SQL.
    container.Register(func() todo.Repository { return infrastructure.DefaultTodoRepository() })

    // File contents → the upload folder on disk.
    container.Register(func() file.Storage { return infrastructure.DefaultUploads() })

    // Domain events → the dispatcher with all listeners.
    container.Register(func() events.Publisher { return Events() })

    return container
}
```

`Events()` builds the dispatcher once (with `sync.Once`) and calls `listeners.Register` on it, so the whole app shares one dispatcher with every listener wired.

A controller with its dependencies:

```go
func (c *TodoDoneController) Loader(todos todo.Repository, events events.Publisher) *TodoDoneController {
    c.todos, c.events = todos, events
    return c
}
```

A new dependency is a new `container.Register(func() I { ... })` plus a new `Loader` parameter.


## Moving an area into the layers

1. Rows and relationships in `db`.
2. Behaviour and events in `model`.
3. `domain/<area>` with its types, `Repository` interface and commands.
4. SQL in `infrastructure.<Area>Repository`.
5. Listeners for the side effects, registered in `listeners.Register`.
6. Bind the interfaces in the container.
7. Switch the controllers to the commands.
