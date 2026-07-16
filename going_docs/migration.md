# Migration

- [Introduction](#introduction)
- [Create a new migration](#create-a-new-migration)
- [Run migration](#run-migration)


## Introduction

Going provides simple migration support for SQLite and PostgreSQL. Migrations are plain SQL files managed via the CLI tool.


## Create migration

Create a new migration using the CLI:

```
art make:migration post
```

This creates a timestamped SQL migration file in `internal/database/migrations/scripts/`. The file is a plain SQL file where you define your database schema:

```sql
-- 20260618192832_post.sql
CREATE TABLE IF NOT EXISTS posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    title TEXT NOT NULL,
    body TEXT,
    user_id INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```


## Run migration

Run all pending migrations:

```
art migrate
```

The tool reads the `DB_CONNECTION` from your `.env` file and applies all pending migration scripts to the database.

### Environment variables

Ensure your `.env` file has the correct database configuration:

```
DB_CONNECTION=sqlite
DB_PATH=./database.db

# or for PostgreSQL:
DB_CONNECTION=postgres
DB_HOST=localhost
DB_USER=user
DB_PASS=password
DB_DATABASE=myapp
```
