# Events

- [Introduction](#introduction)
- [Where the package lives](#where-the-package-lives)
- [Creating a new event](#creating-a-new-event)
- [Where to put the events](#where-to-put-the-events)
- [When to publish an event](#when-to-publish-an-event)
- [Registering listeners](#registering-listeners)
- [Sync and async listeners](#sync-and-async-listeners)
- [Publishing from a controller](#publishing-from-a-controller)
- [Publishing from a service](#publishing-from-a-service)
- [Recording events on the model](#recording-events-on-the-model)
- [Publishing events](#publishing-events)
- [Dispatcher options](#dispatcher-options)
- [Error handling](#error-handling)
- [Context and cancellation](#context-and-cancellation)
- [Graceful shutdown](#graceful-shutdown)
- [WebSocket integration](#websocket-integration)
- [Runnable example](#runnable-example)
- [Testing](#testing)
- [API reference](#api-reference)


## Introduction

Events are the DDD counterpart to jobs. Where a job is something you schedule to run on a timer, an event is something the domain publishes after it already happened, and it carries the facts about it.

A domain event is a plain struct. It never changes state by itself and it never reaches into the database, it only says *what happened*. What should happen as a result is decided by a listener, which lives outside of the domain. That is what keeps an aggregate from knowing about its own side effects: creating a user publishes `UserRegistered`, and whether that sends a mail, a WebSocket message or nothing at all is not the concern of the code that stored the user.

The dispatcher is the piece that connects the two. In a http app the controller publishes the event as the last step of a successful action:

```
controller                          dispatcher                            listeners
----------                          ----------                            ---------
PublishDetached(paymentReceived) ──▶ sync listeners, errors returned ──▶ ConfirmOrder()
                                    └──▶ queue ──▶ worker ─────────────▶ NotifyCustomer()
```

Events are routed by name. Every event reports an `EventName()`, and every listener is registered under that name, so two events can never reach each other's listeners.


## Where the package lives

The dispatcher is part of Going, in the package `super/events` (from v0.1.13):

```go
import "github.com/nicklasjeppesen/going_internal/super/events"
```

It only depends on the standard library. `super/db` imports it (every model can record events, see [Recording events on the model](#recording-events-on-the-model)), never the other way around.

> Apps made before v0.1.13 have their own copy of the package in `app/events/`, taken from the `DomainListnerExample` app. Switch the imports to `super/events` and delete the copy, and drop the `Events` mixin, since every model now records events itself.


## Creating a new event

Events can be placed in `app/events/domain/`, one file per aggregate (see [Where to put the events](#where-to-put-the-events) for the other option):

```go
package domain

// UserRegistered is published after a user has been stored.
type UserRegistered struct {
    ID    int64
    Name  string
    Email string
}

// EventName returns the routing key of the event.
func (UserRegistered) EventName() string {
    return "user.registered"
}

// UserDeleted is published after a user has been removed.
type UserDeleted struct {
    ID int64
}

func (UserDeleted) EventName() string {
    return "user.deleted"
}
```

Keep the event a value type and give `EventName()` a value receiver, because the dispatcher takes the routing key from the zero value of the type (see [Typed listeners](#typed-listeners)).

A few rules worth keeping:

- Only carry facts that a listener needs. An event is not a database model, it is the reason something happened.
- Do not put behaviour on the event. Saving, mailing or rendering does not belong here.
- Treat the name as public API. Listeners are written against it, so renaming it silently unsubscribes them.

You can also embed `events.Base` to inherit a fallback name, and override it with your own `EventName()`:

```go
type OrderShipped struct {
    events.Base
    OrderID int64
}

func (OrderShipped) EventName() string {
    return "order.shipped"
}
```


## Where to put the events

There are two places that work, and which one fits depends on who creates the event:

| | `app/events/domain/` | Next to the model, e.g. `app/model/todo_events.go` |
|---|---|---|
| Who creates the event | a controller or a service, after it saved a plain ORM model | the model's own methods, see [Recording events on the model](#recording-events-on-the-model) |
| Imported by | controllers, services, listeners | the model itself, commands, listeners |
| Example | `DomainListnerExample` (`domain.UserRegistered`) | Kontoret (`model.TodoCompleted`, `model.FilesAttached`) |

The recommendation: **put the events where they are created.** When a rich model records its own events, keep them next to that model, one `<aggregate>_events.go` file per model. The method and the event it records change together, and the model package doesn't have to import a separate events package for its own facts. When there is no rich model and the application layer creates the events, `app/events/domain/` is the natural home.

Don't mix the two for one aggregate. A listener should find all events of a todo in one file.


## When to publish an event

An event describes a *business action*, not a technical change. The most common mistake is an event that fires every time a row changes, which makes every listener run on every update, whether it cares or not. Three rules prevent that:

1. **No generic events.** There is no `user.updated`. There is `user.email-changed`, `user.password-reset`, `order.confirmed`, each published by the one action that causes it.
2. **Only publish when something changed.** An action that sets the email to the value it already had publishes nothing.
3. **Only publish when the action succeeded.** A listener must never react to something the client was told failed. Publish as the last step of the action, see [Publishing from a controller](#publishing-from-a-controller).

Events are published by the application layer, the controllers (or a use case they call), and never by the models. An ActiveRecord model is infrastructure: it knows how a row is stored, not why it changed, so an ORM hook that publishes on every `Save()` is exactly the generic event rule 1 forbids.

A rich model may still *record* an event from a business method such as `Complete()`, as long as the code that saved it publishes afterwards. That is not an ORM hook, see [Recording events on the model](#recording-events-on-the-model).

```go
// One endpoint per action, so the event says why the user changed
func (c *UserController) ChangeEmail(r RequestBody[ChangeEmailRequest]) Result {
    // ... validate and load the user

    if user.Email == r.Body.Email {
        return Response.PrintJson(map[string]any{"changed": false}, 200) // rule 2: no event
    }

    var oldEmail = user.Email
    user.Email = r.Body.Email
    if err := user.Update(); err != nil {
        return Response.PrintJson(map[string]any{"error": err.Error()}, 500) // rule 3: no event
    }

    // Last step, only reached when the update succeeded
    if err := c.dispatcher.PublishDetached(r.R.Context(), domain.UserEmailChanged{
        ID: user.Id, OldEmail: oldEmail, NewEmail: user.Email,
    }); err != nil {
        return Response.PrintJson(map[string]any{"error": err.Error()}, 500)
    }

    return Response.PrintJson(map[string]any{"changed": true}, 200)
}
```

Renaming the user or changing the age goes through other actions, publishes nothing, and no listener runs.


## Registering listeners

Listeners are recommended to be placed in `app/events/listeners/`, and are all wired up in one place so the wiring is easy to read and easy to test:

```go
package listeners

import (
    "context"

    "github.com/nicklasjeppesen/going_internal/super/events"
    "myapp/app/events/domain"
)

// RegisterEvents wires every listener of the app to the dispatcher.
//
// param dispatcher: *events.Dispatcher:
//
// Exemple:
func RegisterEvents(dispatcher *events.Dispatcher) {
    events.On(dispatcher, ConfirmOrder(dispatcher))                 // sync
    events.On(dispatcher, SendWelcomeMessage, events.Async())       // worker pool
    events.On(dispatcher, LogUserDeleted, events.Async())
}
```

Registering the same event twice is not an error. The handlers are called in the order they were registered, and a handler that fails never stops the next one.

### Typed listeners

`events.On` is the normal way to register, see [Sync and async listeners](#sync-and-async-listeners). `events.ListenTo` registers the same way, but leaves it to the caller whether the listener runs sync or async. Both take the routing key from the zero value of the event type and hands the event to your function as that type, so there is no type assertion to write and no way to get the wrong type:

```go
func SendWelcomeMessage(ctx context.Context, event domain.UserRegistered) error {
    log.Printf("welcome %s", event.Email)
    return nil
}
```

`On` and `ListenTo` are functions and not methods on the `Dispatcher`, because Go does not allow a method to declare its own type parameters.

### Listeners by name

`Listen` registers a handler for an explicit name, which is what you reach for when the name comes from somewhere else, such as configuration:

```go
dispatcher.Listen("user.registered", func(ctx context.Context, event events.Event) error {
    log.Println("registered:", event.EventName())
    return nil
})
```

### Wildcard listeners

`ListenAll` registers a handler for every event, no matter its name. This is the place for cross-cutting concerns such as an audit log or a metrics collector. Wildcard handlers run after the handlers of the event itself, and like `Listen` they run on the worker pool for `Publish`:

```go
dispatcher.ListenAll(func(ctx context.Context, event events.Event) error {
    metrics.Count(event.EventName())
    return nil
})
```

To drop a registration again, use `Forget`:

```go
dispatcher.Forget("user.deleted")
```


## Sync and async listeners

Whether a listener runs right away or on the worker pool is decided by the listener when it is registered, not by the code that publishes. The controller never has to know who is listening, or how slow they are:

```go
events.On(dispatcher, ConfirmOrder(dispatcher))                 // sync
events.On(dispatcher, NotifyCustomer, events.Async())           // async
```

| | `events.On(d, h)` | `events.On(d, h, events.Async())` | `ListenTo` / `Listen` / `ListenAll` |
|---|---|---|---|
| On `Publish` | runs before `Publish` returns | worker pool | worker pool |
| On `Dispatch` | runs inline | runs inline | runs inline |
| Error | returned by `Publish` | error handler and `Failures()` | error handler and `Failures()` |
| Use for | turning one event into the next, quick checks | mail, websockets, anything slow or external | existing code, wildcard listeners |

A sync listener is the right place for a link in a chain of events, such as `ConfirmOrder` turning `order.payment-received` into `order.confirmed`. It runs in the goroutine that publishes, so it never takes up a worker while it publishes the next event itself.

Keep sync listeners quick. They run in the request goroutine, so a slow one makes the request slow.


## Publishing from a controller

A controller gets the dispatcher injected through its `Loader`, and publishes as the very last step of the action:

```go
type EventController struct {
    ControllerBase
    dispatcher *events.Dispatcher
}

// Loader receives the dispatcher from the DI container
func (event *EventController) Loader(dispatcher *events.Dispatcher) {
    event.dispatcher = dispatcher
}

func (event *EventController) Payment(r RequestBody[PaymentWebhookRequest]) Result {
    if result := r.Validate(); result.HasError {
        return Response.WithErrors(result.Errors).Back()
    }

    // ... the work of the action, return on any error

    // Last step: only reached when everything above succeeded
    if err := event.dispatcher.PublishDetached(r.R.Context(), domain.PaymentReceived{
        OrderID: r.Body.OrderID,
        Amount:  r.Body.Amount,
        Email:   r.Body.Email,
    }); err != nil {
        return Response.PrintJson(map[string]any{"error": err.Error()}, 500)
    }

    return Response.PrintJson(map[string]any{"published": "order.payment-received"}, 202)
}
```

The dispatcher has to be registered in the container, so `Loader` can receive it:

```go
container.Register(func() *events.Dispatcher { return dispatcher })
```

### Depend on the Publisher interface

A controller that only publishes doesn't need the whole dispatcher. Let it ask for the `events.Publisher` interface instead, and bind the interface in the container:

```go
container.Register(func() events.Publisher { return dispatcher })
```

```go
type TodoDoneController struct {
    ControllerBase
    todos  todo.Repository
    events events.Publisher
}

func (c *TodoDoneController) Loader(todos todo.Repository, events events.Publisher) *TodoDoneController {
    c.todos, c.events = todos, events
    return c
}
```

The controller can't register or forget listeners, and a test can hand it a fake publisher that only collects the events. Only the code that wires listeners at startup (`listeners.Register`) gets `*events.Dispatcher`. See [The Publisher interface](#the-publisher-interface) and [Application layers](layers.md).

Why it is built this way:

- **Publish last.** Every `return` before the publish is a failed action, so a failed action can never publish. An action that causes several events collects them in a local slice and publishes them together at the end.
- **The client hears about a failing sync listener.** The sync listeners run inside `PublishDetached` and their errors are returned, so the action can answer with a 500.
- **Use `PublishDetached`, not `Publish`.** The async listeners run after the response has been sent, and the request context is cancelled at that exact moment. With `Publish` they would all be skipped with `context canceled`.
- **No hidden wiring.** The dispatcher is a field on the controller, so a controller that is not wired up fails at startup, when Going cannot resolve its `Loader`, and not on the first request.

A failing sync listener also stops the async listeners of the event: the client is told the action failed, so no side effect may run for it. See [Errors from Publish](#errors-from-publish).

Note that Going calls `Loader` once per route, not once per controller. Register listeners in `RegisterEvents` at startup, never in `Loader`, or they are registered once for every route of the controller and run that many times.


## Publishing from a service

When an action does more than translating a request, the work belongs in a service. The controller turns the request into a call, and the service checks, stores and publishes. The `DomainListnerExample` app registers users this way:

```
RegisterController.Register ──▶ UserService.Register ──▶ Users.EmailExists   (repository)
                                                     ──▶ Users.Create         (repository, ORM)
                                                     ──▶ PublishDetached(UserRegistered)
```

### The service

The service owns an interface for the persistence it needs, so the business logic never talks to the ORM directly, and a test can hand it a fake instead of a database:

```go
package UserService

var ErrEmailTaken = errors.New("user: the email is already registered")

// Users is the persistence the service needs. The repository implements it.
type Users interface {
    EmailExists(ctx context.Context, email string) (bool, error)
    Create(ctx context.Context, user *db.User) (*db.User, error)
}

type UserService struct {
    users      Users
    dispatcher *events.Dispatcher
}

func New(users Users, dispatcher *events.Dispatcher) *UserService {
    return &UserService{users: users, dispatcher: dispatcher}
}

func (service *UserService) Register(ctx context.Context, registration Registration) (*db.User, error) {
    // 1. check: nothing happens when the email is taken
    exists, err := service.users.EmailExists(ctx, registration.Email)
    if err != nil {
        return nil, fmt.Errorf("user: checking the email: %w", err)
    }
    if exists {
        return nil, ErrEmailTaken
    }

    // 2. store: a failure returns before anything is published
    saved, err := service.users.Create(ctx, &db.User{
        Name:     registration.Name,
        Email:    registration.Email,
        Password: security.HashPassword(registration.Password),
    })
    if err != nil {
        return nil, fmt.Errorf("user: storing the user: %w", err)
    }

    // 3. publish: only reached when the user is stored, built from what the
    // database returned, so the event carries the real id
    if err := service.dispatcher.PublishDetached(ctx, domain.UserRegistered{
        ID: saved.Id, Name: saved.Name, Email: saved.Email,
    }); err != nil {
        return saved, fmt.Errorf("user: the user was stored, but publishing failed: %w", err)
    }

    return saved, nil
}
```

The service publishes with `PublishDetached`, because the context it gets is normally the request context, which is cancelled the moment the response has been sent.

When a sync listener fails, the user is already stored. The service returns the stored user together with the error, so the caller knows both, and the async listeners do not run. Making the store and the publish one atomic step takes an outbox table, see the note at the end of this section.

### The repository

The repository is the only place that knows how a model is stored:

```go
package repository

type UserRepository struct{}

// The compiler checks that the repository fulfils what the service needs
var _ UserService.Users = UserRepository{}

func (UserRepository) EmailExists(ctx context.Context, email string) (bool, error) {
    return db.User{}.DB(ctx).Where("email", email).First().Any(), nil
}

// DB has a value receiver and works on a copy, so the id is set on the copy
// Save returns, and that copy is what is returned
func (UserRepository) Create(ctx context.Context, user *db.User) (*db.User, error) {
    return user.DB(ctx).Save()
}
```

### Wiring

The container builds the service once, with its repository and the dispatcher:

```go
container.Register(func() *UserService.UserService {
    return UserService.New(repository.UserRepository{}, dispatcher)
})
```

The controller only asks for the service:

```go
type RegisterController struct {
    ControllerBase
    users *UserService.UserService
}

func (register *RegisterController) Loader(users *UserService.UserService) {
    register.users = users
}

func (register *RegisterController) Register(r RequestBody[RegisterUserRequest]) Result {
    if result := r.Validate(); result.HasError {
        return Response.WithErrors(result.Errors).Back()
    }

    _, err := register.users.Register(r.R.Context(), UserService.Registration{
        Name: r.Body.Name, Email: r.Body.Email, Password: r.Body.Password,
    })

    switch {
    case errors.Is(err, UserService.ErrEmailTaken):
        return Response.WithErrors(map[string][]string{"email": {"The email is already registered"}}).Back()
    case err != nil:
        log.Printf("register: %v", err)
        return Response.WithErrors(map[string][]string{"email": {"The user could not be registered, try again"}}).Back()
    }

    return View("auth.register", Params{"Title": "Register Page"})
}
```

The route has to be registered with the controller and the method name. A method value such as `registerController.Register` skips the `Loader`, and the service would be nil:

```go
var registerController = new(RegisterController)

webrouter.Post("/registerPost", registerController, "Register").Name("auth.register.post")
```

### The listener

What happens after the registration is decided by the listeners, not by the service. For now the app only writes a line in the log:

```go
// LogUserRegistered writes a line in the log when a user has registered.
// The email is left out on purpose, a log is not the place for personal data.
func LogUserRegistered(_ context.Context, event domain.UserRegistered) error {
    log.Printf("user: user %d (%s) has registered", event.ID, event.Name)
    return nil
}
```

```go
events.On(dispatcher, LogUserRegistered, events.Async())
```

It is async, so the request that registered the user never waits for it. Sending a welcome mail later is one more listener, the service does not change.

### Testing the service

The service is tested with an in-memory fake of `Users`, without a database:

```go
type fakeUsers struct{ stored []*db.User }

func (fake *fakeUsers) EmailExists(_ context.Context, email string) (bool, error) {
    for _, user := range fake.stored {
        if user.Email == email {
            return true, nil
        }
    }
    return false, nil
}

func (fake *fakeUsers) Create(_ context.Context, user *db.User) (*db.User, error) {
    saved := *user
    saved.Id = int64(len(fake.stored) + 1)
    fake.stored = append(fake.stored, &saved)
    return &saved, nil
}

func TestRegisterPublishesTheGeneratedID(t *testing.T) {
    dispatcher := events.New()
    defer dispatcher.Shutdown()

    var published = make(chan domain.UserRegistered, 1)
    events.On(dispatcher, func(_ context.Context, event domain.UserRegistered) error {
        published <- event
        return nil
    })

    saved, err := UserService.New(&fakeUsers{}, dispatcher).Register(context.Background(),
        UserService.Registration{Name: "Nick", Email: "nick@example.com", Password: "Secret123!"})
    if err != nil {
        t.Fatal(err)
    }

    if event := <-published; event.ID != saved.Id {
        t.Fatalf("the event must carry the generated id %d, got %d", saved.Id, event.ID)
    }
}
```

The listener is sync, so the event has been received when `Register` returns.

### When the event must never be lost

The store and the publish are two steps. The listeners run from an in-memory queue, so if the process dies between the two, or before the queued async listeners have run, the event is lost while the user is stored. For most events that is acceptable. For the ones that are not, a payment for example, store the event in an outbox table in the same database transaction as the change, and let a job publish the rows from that table.


## Recording events on the model

[Publishing from a service](#publishing-from-a-service) builds the event in the service, after the save. When the business rules live on a rich model instead (a `db` row plus behaviour, see [Application layers](layers.md)), the model knows best what happened, but it must still never publish. The pattern Kontoret uses splits the two:

```
command ──▶ Repository.Find ──▶ model.Complete()    records TodoCompleted
        ──▶ Repository.Save                          a failure returns, nothing is published
        ──▶ Publisher.PublishAll(t.ReleaseEvents()...)
```

### 1. The model records

Every ActiveRecord model has `Record(event)` and `ReleaseEvents()` built in, so a rich model that embeds a row can record right away (see [Concerns](concerns.md#case-recording-domain-events)). A method records an event only when it changed something:

```go
type Todo struct {
    *db.Todo // db.Todo embeds ActiveRecord[*db.Todo], so Todo has Record and ReleaseEvents
    mixins.Fileable[*db.Todo]
}

// Complete marks the todo as done. It returns false and records nothing
// when the todo is already done.
func (t *Todo) Complete(by int64, at time.Time) bool {
    if t.Done() {
        return false
    }
    t.Completed_at, t.Completed_by = stamp(at), by
    t.Record(TodoCompleted{TodoID: t.Id, Title: t.Title, By: by})
    return true
}
```

The method only changes the struct in memory. It never saves and never sees a dispatcher.

### 2. The command saves, then publishes

A command is a small struct with one `Execute`. It loads the model through the repository, lets the model do the work, saves it, and publishes what the model recorded:

```go
// Deps is what a command works with.
type Deps struct {
    Todos  Repository
    Events events.Publisher
}

// change loads the todo, lets change modify it, and saves and publishes when
// it was changed.
func (d Deps) change(ctx context.Context, id int64, change func(*model.Todo) bool) error {
    t, err := d.Todos.Find(ctx, id)
    if err != nil {
        return err
    }
    if !change(t) {
        return nil // rule 2: nothing changed, nothing is saved or published
    }
    if err := d.Todos.Save(ctx, t); err != nil {
        return err // rule 3: the save failed, nothing is published
    }
    return d.Events.PublishAll(ctx, t.ReleaseEvents()...)
}

// Complete marks a todo as done for the logged-in user.
type Complete Deps

func (c Complete) Execute(ctx context.Context, id int64) error {
    return Deps(c).change(ctx, id, func(t *model.Todo) bool {
        return t.Complete(auth.UserID(ctx), time.Now())
    })
}
```

The controller builds the command and calls it:

```go
err := todo.Complete{Todos: c.todos, Events: c.events}.Execute(r.R.Context(), int64(id))
```

### Why this is not the ORM hook

The rule in [When to publish an event](#when-to-publish-an-event) still holds: the model never publishes. What it forbids is a hook on `Save()`, and the recorder is something else:

- **The event names a business action.** It is recorded by `Complete()`, not by "a row was written". Renaming a todo goes through `Rename()` and records `TodoRenamed`, never a generic `todo.updated` (rule 1).
- **Nothing is recorded when nothing changed.** The method returns false and the command saves nothing (rule 2).
- **Publishing still happens in the application layer, after the save.** The events wait in the model until the command has saved it. A failed save returns before `PublishAll`, so no listener reacts to something that wasn't stored (rule 3).
- **`ReleaseEvents()` empties the list**, so saving the same model twice never publishes an event twice.
- **The ORM never publishes and never writes the events.** `Save()` and `Update()` leave them in the model for the caller, and they live in an unexported field, so they are no column and are not in `ToJson()`.

How the events travel with the model:

- `DB(ctx)` keeps them when the model's `DB` copies its receiver (`user := &_user`, as in [Concerns](concerns.md)). With `user := &User{}` they are lost.
- Rows loaded with `First()` or `Get()` always start without events, even when the query was built from a model that had recorded some.
- A repository that builds a new model after an insert (to pick up the generated id) has to carry the recorded events over to it, or they are lost.

### Testing a command

With a fake repository and a fake publisher, a command is tested without a database or a dispatcher:

```go
// fakePublisher remembers what was published.
type fakePublisher struct{ published []events.Event }

func (f *fakePublisher) PublishAll(_ context.Context, events ...events.Event) error {
    f.published = append(f.published, events...)
    return nil
}
```

Assert that the event was published after exactly one save, and that a second `Complete` on a done todo publishes nothing.

### The Publisher interface

`events.Publisher` and `events.Recorder` are in `super/events`, next to the dispatcher:

```go
// Publisher is what controllers and commands publish domain events with.
// Dispatcher fulfils it; in a test a fake Publisher can collect the events
// instead of running listeners.
type Publisher interface {
    PublishAll(ctx context.Context, events ...Event) error
}

// Recorder is a model that has recorded events about what happened to it.
// Every ActiveRecord model in super/db is one (see ActiveRecord.Record).
// ReleaseEvents returns the events and forgets them, so they are only
// published once.
type Recorder interface {
    ReleaseEvents() []Event
}

// PublishAll publishes the events in order, after the action has succeeded.
// Each goes through PublishDetached: the context keeps its values (such as
// the logged-in user) but is never cancelled, so the async listeners can
// finish after the response has been sent. The sync listeners' errors are
// joined and returned; a failing event never stops the ones after it.
func (dispatcher *Dispatcher) PublishAll(ctx context.Context, events ...Event) error {
    var errs []error
    for _, event := range events {
        if err := dispatcher.PublishDetached(ctx, event); err != nil {
            errs = append(errs, err)
        }
    }
    return errors.Join(errs...)
}

var _ Publisher = (*Dispatcher)(nil)
```

Note that every event is published even when an earlier one failed, so one failing sync listener doesn't stop the other events of the action. Each event's own async listeners are still skipped when its sync listeners fail.


## Publishing events

The dispatcher offers three ways to send an event, and the difference is whether you need to know that the listeners are done, and whether they should survive the request that published them.

### Publish

`Publish` first runs the sync listeners of the event in the calling goroutine and returns their errors. When they all succeed, it hands the event to the worker pool for every other listener and returns without waiting for them:

```go
// A job owns its context, so a plain Publish is right: when the scheduler
// stops, the queued listeners of the job are cancelled with it.
func Runner(job jobs.Job) {
    ctx := job.Ctx()

    for _, invoice := range overdueInvoices(ctx) {
        if err := dispatcher.Publish(ctx, domain.InvoiceOverdue{ID: invoice.ID}); err != nil {
            log.Printf("invoice %d: %v", invoice.ID, err)
        }
    }
}
```

For a service that stores something and then publishes, see [Publishing from a service](#publishing-from-a-service).

Handing the event to the pool only blocks while the queue is full, and it gives up as soon as the context is done or the dispatcher is shutting down.

### PublishDetached (outliving the caller)

`PublishDetached` is `Publish` with the cancellation removed. It is the way to publish from a controller, and from any other code whose context ends before the listeners are done:

```go
dispatcher.PublishDetached(request.R.Context(), domain.PaymentReceived{OrderID: "1001"})
```

The request context is cancelled the moment the response has been sent, but the listeners only start working after that. A plain `Publish` therefore hands the workers a context that is already done, and every listener is skipped with `context canceled`:

```
events: listener for "order.payment-received" was not called: context canceled
events: listener for "order.confirmed" was not called: context canceled
```

`PublishDetached` keeps the values of the request context (trace ids, auth info, tenant) but drops its cancellation, so an event published by a request still gets handled after the client is gone.

It never times out on its own, so a detached listener should still bound its own work. Give the listener a context with a deadline if it talks to something that can hang:

```go
func SendWelcomeMessage(ctx context.Context, event domain.UserRegistered) error {
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()

    return sendMail(ctx, event.Email)
}
```

### Dispatch (synchronous)

`Dispatch` calls every listener in the calling goroutine, async ones included, and returns their errors joined together. Use it when the event is part of a transaction and the caller has to know that it was fully handled:

```go
if err := dispatcher.Dispatch(ctx, domain.UserRegistered{ID: user.Id, Email: user.Email}); err != nil {
    return err
}
```

### Which one should I use?

| | `Publish` | `PublishDetached` | `Dispatch` |
|---|---|---|---|
| Sync listeners | before it returns | before it returns | inline |
| Async listeners | worker pool | worker pool | inline |
| Returns | sync errors, or the queue error | sync errors, or the queue error | every listener error |
| A sync listener fails | async listeners are skipped | async listeners are skipped | every listener still runs |
| Cancelled with the caller | yes | no | yes |
| Use for | jobs, schedulers, background work | controllers, the default from a request | when every listener must finish before the caller goes on |

Publishing inside a listener is fine, and it is how a chain of events is built. Prefer a sync listener for the link that publishes: an async listener that publishes takes up a worker while it waits for room in the queue, and if every worker does that at once while the queue is full, nothing is left to empty it.


## Dispatcher options

`events.New` starts the workers right away and takes options for the ones you want to change:

```go
dispatcher := events.New(
    events.WithWorkers(8),                              // default: 4
    events.WithBuffer(512),                             // default: 256
    events.WithErrorHandler(func(err error) {           // default: none
        log.Printf("events: %v", err)
    }),
)
```

`WithWorkers` sets how many listeners may handle events at the same time. `WithBuffer` sets how many events may wait in the queue before `Publish` blocks. Both ignore values below one.


## Error handling

A listener returns an error by returning one, and the dispatcher decides what happens to it. Two rules hold for sync and async listeners alike:

- A listener that fails never stops the other listeners of its own kind. The one exception is on purpose: a failing sync listener keeps `Publish` from queueing the async ones, see below.
- A listener that panics is recovered and turned into an error, so one broken listener can never take down the request or the worker that called it.

### Errors from Dispatch

`Dispatch` returns the errors of all listeners, joined with `errors.Join`, so `errors.Is` and `errors.As` still work:

```go
err := dispatcher.Dispatch(ctx, domain.UserRegistered{ID: 1})

if errors.Is(err, ErrMailServerDown) {
    // roll something back
}
```

They are returned rather than reported to the error handler, because the caller sees them right here.

### Errors from Publish

The errors of the sync listeners are returned by `Publish`, joined together. Every sync listener is still called, but when at least one of them fails the event is **not** handed to the worker pool, so none of the async listeners run, wildcard listeners included. The caller is told the event failed, so a side effect such as a mail or a websocket push must not happen for it:

```go
events.On(dispatcher, ReserveStock)                          // sync, fails
events.On(dispatcher, SendConfirmationMail, events.Async())  // never runs

err := dispatcher.Publish(ctx, domain.OrderPlaced{ID: 1})   // the ReserveStock error
```

`Dispatch` is different: it runs every listener inline and returns all their errors, whatever fails.

An asynchronous listener runs after `Publish` has already returned, so its error cannot be returned to the caller. It goes to two places instead:

```go
// 1. The error handler, if you registered one with WithErrorHandler.
dispatcher.Failures()      // every collected error, safe to read from a monitor
dispatcher.ClearFailures() // drop them and get the number that were dropped
```

Always register an error handler in production. Without one, an asynchronous failure is only visible through `Failures()`, which nothing reads by default.


## Context and cancellation

The context you publish with is the context the listeners receive, and it is checked before every listener is called:

```go
func SendWelcomeMessage(ctx context.Context, event domain.UserRegistered) error {
    select {
    case <-ctx.Done():
        return ctx.Err() // the request is gone, there is nobody left to greet
    default:
    }

    return sendMail(event.Email)
}
```

A listener that blocks forever is the one thing that can keep `Shutdown` waiting, so a listener should always respect its context, the same way a job runner does.

Because the publishing context is the listener context, an event published with a request context has almost no time left. That is why a controller publishes with `PublishDetached`.


## Graceful shutdown

`Shutdown` stops accepting new events and waits until the queued ones have been handled. The queue is drained, not thrown away, so an event that was accepted still gets handled before the process exits:

```go
func main() {
    var app = app.NewApp()

    dispatcher := events.New(events.WithErrorHandler(logFailure))
    listeners.RegisterEvents(dispatcher)

    // Handle the events that are still queued before the app exits
    defer dispatcher.Shutdown()

    app.Start()
}
```

Publishing after a shutdown returns `events.ErrShutdown`, so a request that races the shutdown fails loudly instead of silently losing an event. `Shutdown` is safe to call more than once.


## WebSocket integration

Listeners can push to WebSocket clients with the `channels` package, the same way jobs and controllers do:

```go
func SendWelcomeMessage(_ context.Context, event domain.UserRegistered) error {
    var websocketChannel = channels.WebSocketMessageProvider{}

    websocketChannel.SendMessageToSocket(channels.Socket{
        URL:     "/ws/message",
        Message: "Welcome " + event.Name,
        Data: map[string]any{
            "event": event.EventName(),
            "id":    event.ID,
            "email": event.Email,
        },
    })

    return nil
}
```

This triggers the `AppReceiver` method in the hub registered for `/ws/message`.


## Runnable example

The `DomainListnerExample` app wires all of this together, so the whole chain can be watched instead of only read about:

```bash
go run main.go
```

`POST https://localhost:8443/events/payment` publishes `order.payment-received` as the last step of the action. The sync listener `ConfirmOrder` confirms the order and publishes `order.confirmed` before the controller answers `202`, and the async listener `NotifyCustomer` pushes that to the WebSocket clients of `/ws/events`:

```bash
curl -sk https://localhost:8443/events/payment \
  -H 'Content-Type: application/json' \
  -d '{"order_id":"1001","amount":499,"email":"nick@example.com"}'
```

```
events: ConfirmOrder got order.payment-received, order 1001 is paid, so it is confirmed
events: ▶ order.payment-received
events: NotifyCustomer got order.confirmed, order 1001 is confirmed
events: ▶ order.confirmed
events: pushing Your order 1001 is confirmed to 1 client(s)
```

The async lines may come in another order, they run on different workers. Send an invalid body and nothing is published at all, the action returns on the failed validation before it reaches the publish.

`GET https://localhost:8443/events` lists the chain and every registered listener, which is the quickest way to see what an app has registered.

Note that the hub keeps its own clients, keyed by connection id. The socket manager only tracks connections for broadcasting, so a hub that pushes to its own clients has to remember them when a client invokes a `subscribe` action.


## Testing

A dispatcher is cheap to create, so a test can have its own without touching the application:

```go
func TestSendWelcomeMessage(t *testing.T) {
    dispatcher := events.New()
    defer dispatcher.Shutdown()

    var called bool
    events.ListenTo(dispatcher, func(_ context.Context, event domain.UserRegistered) error {
        called = event.Email == "nick@example.com"
        return nil
    })

    err := dispatcher.Dispatch(context.Background(), domain.UserRegistered{Email: "nick@example.com"})
    if err != nil {
        t.Fatalf("Dispatch returned %v, want nil", err)
    }

    if !called {
        t.Error("the listener did not receive the event")
    }
}
```

Because `Dispatch` is synchronous, a test never has to sleep. When you test an asynchronous listener, wait for the outcome rather than for a fixed duration, and keep the race detector on, since the listener runs on another goroutine:

```go
// Tests are clean under -race, run them that way:
// go test -race ./tests/...
```

A listener registered with `events.On` without `Async` has run when `Publish` returns, so it can be tested without waiting too.

### Testing a controller

Send a request through the real router and assert on the response and on what the listeners saw:

```go
func TestPayment(t *testing.T) {
    t.Chdir("../../..") // see below

    dispatcher := events.New()
    defer dispatcher.Shutdown()

    var confirmed = make(chan domain.OrderConfirmed, 1)
    events.On(dispatcher, listeners.ConfirmOrder(dispatcher))
    events.On(dispatcher, func(_ context.Context, event domain.OrderConfirmed) error {
        confirmed <- event
        return nil
    })

    mux := http.NewServeMux()
    router := webstdlib.NewMyRouter().UseContainer(container.GetContainer(dispatcher))
    routes.Eventsrouter(router).RegisterRoutes(mux)

    request := httptest.NewRequest(http.MethodPost, "/events/payment",
        strings.NewReader(`{"order_id":"A-1","amount":100,"email":"a@b.dk"}`))
    request.Header.Set("Content-Type", "application/json")
    response := httptest.NewRecorder()
    mux.ServeHTTP(response, request)

    if response.Code != http.StatusAccepted {
        t.Fatalf("expected 202, got %d", response.Code)
    }

    // The chain is sync, so it has run when the request returns
    select {
    case <-confirmed:
    default:
        t.Fatal("the order was not confirmed")
    }
}
```

Two things to know when a test goes through the real router:

- Going keeps named routes in a package global, so a router with named routes can only be registered once per test binary. Build it once with `sync.Once`, and `Forget` the listeners a test registered when it is done.
- Validation looks for `internal/resources/rules/rules.json` relative to the working directory. A test in a sub folder has to run from the project root, for example with `t.Chdir("../../..")`.


## API reference

### Events

- `events.Event` - Interface every event implements, one method: `EventName() string`
- `events.Handler` - `func(ctx context.Context, event Event) error`
- `events.Base` - Optional embedded struct, provides a fallback event name
- `events.Wildcard` - The reserved name `"*"` that matches every event
- `events.ErrShutdown` - Returned when publishing to a dispatcher that stopped
- `events.Publisher` - Interface with `PublishAll`, what controllers and commands depend on
- `events.Recorder` - Interface with `ReleaseEvents() []Event`, a model that records events

### Dispatcher

- `events.New(options ...Option) *Dispatcher` - Create a dispatcher and start its workers
- `events.On(dispatcher, handler, options ...ListenOption)` - Register a typed handler, sync unless `Async()` is passed
- `dispatcher.Listen(name string, handler Handler)` - Register a handler for a name, it follows the caller
- `events.ListenTo(dispatcher, handler)` - Register a typed handler that follows the caller
- `dispatcher.ListenAll(handler Handler)` - Register a handler for every event
- `dispatcher.Forget(name string)` - Drop the handlers of a name
- `dispatcher.HasListeners(name string) bool` - Whether anything listens for a name
- `dispatcher.Names() []string` - The sorted names that currently have a handler
- `dispatcher.Publish(ctx, event) error` - Run the sync listeners, queue the event for the rest when they all succeeded
- `dispatcher.PublishDetached(ctx, event) error` - `Publish` without the cancellation of the calling context
- `dispatcher.PublishAll(ctx, events ...Event) error` - `PublishDetached` for each event in order, the errors joined
- `dispatcher.Dispatch(ctx, event) error` - Call every listener now, return their errors
- `dispatcher.Failures() []error` - The errors collected from asynchronous listeners
- `dispatcher.ClearFailures() int` - Drop them and return how many were dropped
- `dispatcher.Shutdown()` - Stop accepting events and drain the queue

### Options

- `events.WithWorkers(count int)` - How many listeners run at the same time, default 4
- `events.WithBuffer(size int)` - How many events may wait in the queue, default 256
- `events.WithErrorHandler(handler func(error))` - Called for every asynchronous failure

### Recording on models (`super/db`)

- `model.Record(event events.Event)` - Record that something happened to the model, published later by whoever saves it
- `model.ReleaseEvents() []events.Event` - The recorded events in order, and forget them; every model is an `events.Recorder`

### Listener options

- `events.Async()` - Run the listener on the worker pool instead of before `Publish` returns
