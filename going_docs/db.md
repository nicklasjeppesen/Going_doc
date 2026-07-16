# ORM - Database access

- [Introduction](#introduction)
- [Setup](#setup)
- [Querying](#querying)
- [Create](#create)
- [Update](#update)
- [Delete](#delete)
- [Where conditions](#where-conditions)
- [Eager loading](#eager-loading)
- [Transactions](#transactions)
- [Select (raw queries)](#select-raw-queries)
- [Pagination](#pagination)
- [Add routes](#add-routes)


## Introduction

This section introduces how to use the built-in ORM for Going.
The ORM is inspired by Laravel's Eloquent and provides an ActiveRecord pattern with a chainable query builder, supporting PostgreSQL and SQLite.


## Setup

To create a new ORM model, define a struct that embeds `ActiveRecord[*YourType]`. It is strongly recommended to place model files in `/app/models/db/`.

The struct includes the `ActiveRecord` generic type, which should be a pointer to the current struct. Every field represents a column in the database.

The `DB(ctx)` function is required - it initializes the ORM with table name, columns, and a database connection:

```go
import (
    "context"
    . "github.com/nicklasjeppesen/going_internal/super/db"
)

type User struct {
    ActiveRecord[*User] `json:"-" swaggerignore:"true"`
    Name                string `json:"name" validate:"required"`
    Age                 int64  `json:"-" validate:"min=0,max=99"`
    Email               string `json:"email" validate:"required"`
    Password            string `json:"password" validate:"required" hidden:"true"`
    SessionToken        string `json:"-" hidden:"true"`
    Company_id          int64  `json:"Company_id" validate:"required"`
}

// Creating DB instance
func (_user User) DB(ctx context.Context) *User {
    user := &User{}
    user.Table = "users"
    user.Columns = user.columns()

    // Creating ORM with context
    user.ParentDB = CreateORM(ctx, user)
    return user
}

func (user *User) columns() Columns {
    return Columns{
        // Column        "values"
        "age":          &user.Age,
        "name":         &user.Name,
        "email":        &user.Email,
        "password":     &user.Password,
        "sessiontoken": &user.SessionToken,
        "company_id":   &user.Company_id,
    }
}
```

### JSON and struct tags

- `json:"-"` hides the field from JSON output.
- `hidden:"true"` hides the field when the model is serialized to JSON via `ToJson()`.
- `validate:"..."` tags are used by the validation system.


## Querying

Always call `DB(ctx)` before performing any database operations. The context follows Go's standard `context.Context` pattern.

### Get all records
```go
users := User{}.DB(context.Background()).Get() // returns Collection[*User]
```

### Get first record
```go
user := User{}.DB(context.Background()).Where("id", 1).First() // returns *User
```

### With conditions
```go
users := User{}.DB(context.Background()).Where("age", 30).Get()
```

### WhereIn
```go
users := User{}.DB(context.Background()).WhereIn("id", []int64{1, 2, 3}).Get()
```

### Ordering
```go
users := User{}.DB(context.Background()).OrderBy("name").Get()
users := User{}.DB(context.Background()).OrderByDesc("created_at").Get()
```

### Limit and Offset
```go
users := User{}.DB(context.Background()).Limit(10).OffSet(20).Get()
```

### Eager loading relationships
```go
user := User{}.DB(context.Background()).With("Company").First()
fmt.Println(user.Company().Item()) // returns the related Company
```

### Pessimistic locking (PostgreSQL only)
```go
user := User{}.DB(context.Background()).Where("id", 1).LockForUpdate().First()
```


## Create

```go
user := new(User).DB(context.Background())
user.Name = "Nicklas"
user.Age = 30
user.Email = "njep@Going.com"
savedUser, err := user.Save() // returns (*User, error) with auto-generated id, created_at, updated_at
```


## Update

Fetch the record, change the values, and call `Update()`. The ORM uses the primary key for updating:

```go
user := User{}.DB(context.Background()).Where("id", 1).First()
user.Age = 1000
if err := user.Update(); err != nil {
    // Handle error
}
```


## Delete

Delete a record by fetching it first and calling `Delete()`:

```go
user := User{}.DB(context.Background()).Where("id", 3).First()
user.Delete()
```


## Where conditions

The ORM supports several query conditions:

```go
// Basic where
.Where("column", value)

// WhereIn (for IN clauses)
.WhereIn("column", []int64{1, 2, 3})

// Or condition
.Or("column", value)

// Polymorphic where
.WhereMorph("entryable", "User")
```


## Eager loading

Eager loading prevents N+1 query problems by loading related models in advance:

```go
// Load a single relation
company := Company{}.DB(ctx).With("Users").First()
for _, user := range company.Users().Items() {
    fmt.Println(user.Name)
}

// Load multiple relations
user := User{}.DB(ctx).With("Company", "Posts").First()
```

See the [Relationships](relationships.md) section for details on defining relationships.


## Transactions

Wrap multiple operations in a transaction. If the function returns an error or panics, the transaction is rolled back:

```go
err := User{}.DB(ctx).Transaction(func(txUser *User) error {
    txUser.Name = "Updated Name"
    if _, err := txUser.Save(); err != nil {
        return err
    }
    // more operations...
    return nil
})
```

You can also attach an existing transaction:

```go
txUser := user.WithTx(tx)
```


## Select (raw queries)

For custom SQL queries, use `Select()` which returns `[]map[string]any`:

```go
results := User{}.DB(ctx).Select("SELECT * FROM users WHERE age > ?", 25)
```


## Pagination

Pagination is built into the ORM. Call `Pagination()` with the HTTP request and the number of items per page:

```go
func (c *SampleController) Get(r request.Requestbase) {
    users := db.User{}.DB(r.R.Context()).Pagination(r.R, 3)
    Response.PrintJson(users)
}
```

Example response:

```json
{
    "current_page": 1,
    "data": [
        {
            "created_at": "0001-01-01T00:00:00Z",
            "email": "nicklas@live.dk",
            "id": 1,
            "name": "Nicklas",
            "pivots": null,
            "routes": null,
            "updated_at": "2026-06-18T19:28:32Z"
        }
    ],
    "first_page_url": "/sample?page=1",
    "next_page_url": "/sample?page=2",
    "path": "/sample",
    "per_page": 3,
    "prev_page_url": ""
}
```


## Add routes

The ORM integrates with the routing system to include HATEOAS-style action links in your API responses (Level 3 in the [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html#level3)).

Define your named routes:

```go
func Samplerouter() *customrouter.MyRouter {
    sampleRouter := customrouter.NewMyRouter()
    var sampleController = SampleController{}

    sampleRouter.GET("/sample/index", sampleController.ShowUser).Name("sample.index")
    sampleRouter.POST("/sample/store", sampleController.Store).Name("sample.store")
    sampleRouter.GET("/sample/show/{id}", sampleController.ShowUser).Name("sample.ShowInput")
    return sampleRouter
}
```

Then in your controller, use `AddRoutes()` with the route prefix:

```go
type SampleController struct {
    customrouter.BaseController
}

func (c *SampleController) Get(r request.Requestbase) {
    users := models.User{}.DB(r.R.Context()).AddRoutes("sample").Get()
    response.NewResponse().PrintJson(users).ServeHTTP(r.W, r.R)
}
```

`AddRoutes("sample")` finds all routes with names starting with "sample" and replaces `{id}` placeholders with each model's primary key.

### Foreign key placeholders

If your URL contains a foreign key that is not part of the model, pass a `map[string]string` to `AddRoutes`:

```go
func Samplerouter() *customrouter.MyRouter {
    sampleRouter := customrouter.NewMyRouter()
    var sampleController = SampleController{}

    sampleRouter.GET("/sample/index", sampleController.ShowUser).Name("sample.index")
    sampleRouter.GET("/sample/group/{groupId}/show/{id}", sampleController.ShowUser).Name("sample.ShowInput")
    return sampleRouter
}
```

```go
func (c *SampleController) Get(r Request) {
    users := models.User{}.DB(r.R.Context()).AddRoutes(
        "sample",
        map[string]string{"groupId": "1"},
    ).Get()
    Response.PrintJson(users)
}
```

This will return routes with `groupId` replaced by "1" for all objects.
