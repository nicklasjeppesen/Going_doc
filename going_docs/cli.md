# CLI Commands

- [Introduction](#introduction)
- [Make commands](#make-commands)
  - [make:controller](#makecontroller)
  - [make:apicontroller](#makeapicontroller)
  - [make:model](#makemodel)
  - [make:hub](#makehub)
  - [make:middleware](#makemiddleware)
  - [make:job](#makejob)
  - [make:cert](#makecert)
  - [make:migration](#makemigration)
- [Migrate](#migrate)


## Introduction

Going includes a Cobra-based CLI tool called `art` that provides code generation commands (similar to Laravel's `artisan`). These commands scaffold boilerplate code for your application.


## Make commands

### make:controller

Generate a new controller file:

```
art make:controller UserController
```

Generate a resource controller with CRUD methods (Index, Show, Store, Update, Destroy):

```
art make:controller UserController -r
```


### make:apicontroller

Generate a new API controller:

```
art make:apicontroller PostController
```


### make:model

Generate a new model file:

```
art make:model User
```

Generate a model with a migration:

```
art make:model User -m
```

Generate a model with a controller:

```
art make:model User -c
```

Generate a model with both migration and controller:

```
art make:model User -m -c
```


### make:hub

Generate a new WebSocket hub file at `app/http/hubs/`:

```
art make:hub post
```

This creates a hub with the basic structure:

```go
type PostHub struct {
    socket.BaseHub
}

func (post *PostHub) RegisterRoutes() {
    post.On("sendMessage", post.handleNewMessage)
}

func (post *PostHub) handleNewMessage(parameters []string, client *socket.Client) error {
    return nil
}

func (post *PostHub) CancelConnection(client *socket.Client) {}

func (post *PostHub) AppReceiver(message channels.Socket) {}
```


### make:middleware

Generate a new middleware file:

```
art make:middleware LoggingMiddleware
```


### make:job

Generate a new background job file:

```
art make:job CleanupJob
```


### make:cert

Generate a self-signed TLS certificate:

```
art make:cert localhost
```

This uses Go's `generate_cert.go` to create certificate and key files for local development.


### make:migration

Generate a new timestamped SQL migration file:

```
art make:migration create_users_table
```

This creates a file at `internal/database/migrations/scripts/` with a timestamp prefix. The file is driver-specific (SQLite or PostgreSQL based on your `DB_CONNECTION` env var).


## Migrate

Run all pending database migrations:

```
art migrate
```

This reads the `DB_CONNECTION` from your `.env` file and applies all pending migration scripts.

### Supported databases

- **SQLite**
- **PostgreSQL**

### Environment configuration

```
# SQLite
DB_CONNECTION=sqlite
DB_PATH=./database.db

# PostgreSQL
DB_CONNECTION=postgres
DB_HOST=localhost
DB_USER=user
DB_PASS=password
DB_DATABASE=myapp
```
