# ORM - Database Relationships

- [Introduction](#introduction)
- [BelongsTo](#belongsto)
- [HasOne](#hasone)
- [HasMany](#hasmany)
- [BelongsToMany](#belongs-to-many)
- [BelongsToMorph](#belongstomorph)
- [HasManyMorph](#hasmanymorph)
- [Query modifiers](#query-modifiers)


## Introduction

Going ActiveRecord provides a set of relationship types for creating associations between your models. The relationship methods are defined on your model structs and use the `CallerMethodName()` convention to automatically determine the relationship key.


## BelongsTo

A belongs-to association sets up a relationship where each instance of the declaring model "belongs to" one instance of the other model. For example, if each user belongs to one company:

```go
type User struct {
    ActiveRecord[*User] `json:"-" swaggerignore:"true" validate:"-"`
    Name                string `json:"name" validate:"required"`
    Age                 int64  `json:"age" validate:"min=0,max=99"`
    Email               string `json:"email" validate:"required"`
    Password            string `json:"password" validate:"required" hidden:"true"`
    SessionToken        string `json:"-" hidden:"true"`
    Company_id          int64  // foreign key
}

func (user *User) columns() Columns {
    return Columns{
        "age":          &user.Age,
        "name":         &user.Name,
        "email":        &user.Email,
        "password":     &user.Password,
        "sessiontoken": &user.SessionToken,
        "company_id":   &user.Company_id,
    }
}

// Set up the relationship
func (user *User) Company() BelongsTo[*Company] {
    return NewBelongsTo(new(Company).DB(user.GetCtx()), user)
}
```

### Usage

```go
user := new(db.User).DB(context.Background()).With("Company").First()
fmt.Println(user.Company().Item()) // returns Company struct
```

### Convention

Going calls `GetName()` on the related model and appends `_id` to find the foreign key column. For `Company()`, it looks for `Company_id` in the User struct.


## HasOne

A has-one association indicates that one other model has a reference to this model. For example, if each company has one user:

```go
type Company struct {
    ActiveRecord[*Company] `json:"-" swaggerignore:"true" validate:"-"`
    Name                   string `json:"name" validate:"required"`
}

func (c *Company) User() HasOne[*User] {
    return NewHasOne(new(User).DB(c.GetCtx()), c)
}
```

### Usage

```go
company := new(db.Company).DB(context.Background()).With("User").First()
fmt.Println(company.User().Item()) // returns User struct
```

### Helper function

Define a helper to avoid string conflicts with the `With()` method:

```go
func (c *Company) WithUser() *Company {
    return c.With("User")
}

// Usage
company := new(db.Company).DB(context.Background()).WithUser().First()
fmt.Println(company.User().Item())
```

### Convention

Going calls `GetName()` on the parent model and appends `_id` to look up the foreign key in the related table. For `User()` on Company, it looks for `Company_id` in the users table.


## HasMany

Has-many is similar to has-one, but assumes zero or more related records. Reuse the User struct from the belongs-to example and change the Company struct:

```go
type Company struct {
    ActiveRecord[*Company] `json:"-" swaggerignore:"true" validate:"-"`
    Name                   string `json:"name" validate:"required"`
}

func (c *Company) Users() HasMany[*User] {
    return NewHasMany(new(User).DB(c.GetCtx()), c)
}
```

### Usage

```go
company := new(db.Company).DB(context.Background()).WithUsers().First()
for _, user := range company.Users().Items() {
    fmt.Println(user.ToJson())
}
```

You can also define a helper:

```go
func (c *Company) WithUsers() *Company {
    return c.With("Users")
}
```


## Belongs to many
association creates a direct many-to-many relationship with another model, with no intervening model. This association indicates that each instance of the declaring model refers to zero or more instances of another model.

```go
func (c *Company) Users() BelongsToMany[*User] {
    return NewBelongsToMany(new(User).DB(c.GetCtx()), c)
}

func (c *Company) WithUsers() *Company {
    return c.With("Users")
}
```

### Usage

```go
company := new(db.Company).DB(context.Background()).WithUsers().First()
for _, user := range company.Users().Items() {
    fmt.Println(user.Name)
}
```

### Pivot table convention

The pivot table is named alphabetically by combining both model names: `company_user`, with columns `company_id` and `user_id`. You can customize the name with `PivotTableName()`.

### Pivot columns

Include extra columns from the pivot table:

```go
func (c *Company) Users() BelongsToMany[*User] {
    return NewBelongsToMany(new(User).DB(c.GetCtx()), c).PivotCols("active")
}
```

Access pivot data:

```go
company := new(db.Company).DB(context.Background()).WithUsers().First()
for _, user := range company.Users().Items() {
    fmt.Println(user.Name)
    fmt.Println(user.Pivots["active"]) // pivot column value
}
```

### Pivot operations

```go
// Attach a new record to the pivot
relation.Attach(userId)

// Detach a record from the pivot
relation.Detach(userId)

// Update an existing pivot record
relation.UpdateExistingPivot(userId, map[string]any{"active": false})

// Filter by pivot columns
relation.WherePivot("active", true)

// Order by pivot columns
relation.OrderByPivot("created_at")
```


## BelongsToMorph

Polymorphic associations (inspired by Rails delegates) allow a model to belong to different types. Instead of single table inheritance (STI), you use a type column and an ID column to reference different models.

### The idea

An `Entry` table has `entryable_type` (the model name) and `entryable_id` (the foreign key). This lets you retrieve entries from different models (User, Company, etc.) ordered by creation time.

### Implementation

Define an interface for all delegatable models:

```go
type IEntry interface {
    types.IRepository // Required

    // Abstract methods that each model must implement
    Title() string

    // Optional - can be overridden
    PrintHi() string
}

// Base struct that models embed to become delegatable
type EntryAble struct{}

func (e *EntryAble) PrintHi() string {
    return "Hi from EntryAble"
}
```

Define the Entry model:

```go
type Entry struct {
    ActiveRecord[*Entry]
    Name               string
    EntryAble_type     string
    EntryAble_id       int64
    EntryAble          IEntry // value holder
}

const delegate = "entryable"

// Register all models that can be entryable
var entryAbles = []IEntry{
    new(Company).DB(context.Background()),
    new(User).DB(context.Background()).With("Company"),
}

func (e Entry) DB(ctx context.Context) *Entry {
    entry := &Entry{}
    entry.Table = "entries"
    entry.Columns = Columns{
        "name":             &entry.Name,
        delegate + "_id":   &entry.EntryAble_id,
        delegate + "_type": &entry.EntryAble_type,
    }
    entry.ParentDB = CreateORM(ctx, entry)
    return entry
}

// Set up the relationship
func (e *Entry) Entryable() BelongsToMorphRelation[IEntry] {
    return NewBelongsToMorph(entryAbles, e)
}

// Scopes for filtering by type
func (e *Entry) Company() types.IDB[*Entry] {
    return e.WhereMorph(delegate, new(Company).DB(e.GetCtx()).GetName())
}

func (e *Entry) User() types.IDB[*Entry] {
    return e.WhereMorph(delegate, new(User).DB(e.GetCtx()).GetName())
}
```

Models implement the interface:

```go
type Company struct {
    ActiveRecord[*Company]
    Name    string
    EntryAble // embed to become delegatable
}

func (c Company) DB(ctx context.Context) *Company {
    company := &Company{}
    company.Table = "companies"
    company.Columns = Columns{
        "name": &company.Name,
    }
    company.ParentDB = CreateORM(ctx, company)
    return company
}

// Implement the abstract method
func (company *Company) Title() string {
    return company.Name + ", Company Title is called"
}

// Override the default method
func (company *Company) PrintHi() string {
    return company.EntryAble.PrintHi() + " with added logic from Company"
}
```

### Usage

```go
entries := models.Entry{}.DB(context.Background()).With("Entryable").OrderByDesc("id").Company().Get()
for _, entry := range entries {
    fmt.Println(entry.EntryAble.Title())
    fmt.Println(entry.EntryAble.PrintHi())
}
```

### DB Schema (SQLite)

```sql
CREATE TABLE IF NOT EXISTS entries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    entryable_type TEXT NOT NULL,
    entryable_id INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TRIGGER IF NOT EXISTS update_entries_updated_at
AFTER UPDATE ON entries
FOR EACH ROW
BEGIN
    UPDATE entries
    SET updated_at = CURRENT_TIMESTAMP
    WHERE id = OLD.id;
END;
```


## HasManyMorph

The inverse of BelongsToMorph - a model has many polymorphic entries:

```go
func (u *User) Entries() HasManyMorphRelation[*Entry] {
    return NewHasManyMorph(entryAbles, u)
}
```

### Usage

```go
user := new(db.User).DB(context.Background()).With("Entries").First()
for _, entry := range user.Entries().Items() {
    fmt.Println(entry.Name)
}
```


## Query modifiers

HasMany and BelongsToMany relationships support query modifiers for filtering, sorting, and limiting the related records:

```go
// Add where conditions to a relationship query
func (c *Company) ActiveUsers() HasMany[*User] {
    return NewHasMany(new(User).DB(c.GetCtx()), c).
        Where("active", true).
        OrderBy("name")
}
```

Modifiers are applied when the relationship is loaded via eager loading (`With()`).
