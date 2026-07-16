# Request

- [Introduction](#introduction)
- [Basic request handling](#basic-request-handling)
- [Requestbase](#requestbase)
- [Authenticated user](#authenticated-user)
- [Request body](#request-body)
- [Validation](#validation)
- [Reflection-based parameter injection](#reflection-based-parameter-injection)


## Introduction

This section describes how to handle requests using Going's built-in request handling. Going provides a custom request wrapper and a reflection-based function dispatcher that automatically injects parameters into controller methods.


## Basic request handling

The `Requestbase` struct wraps Go's standard `http.ResponseWriter` and `*http.Request`:

```go
func controller(r Request) {
    // Access the standard Go request
    email := r.R.FormValue("email")
    password := r.R.FormValue("password")
    ctx := r.R.Context()

    // Access the response writer
    fmt.Fprint(r.W, "Hello")
}
```


## Request

Available methods:

- `r.R.FormValue("key")` - Get form/query values
- `r.R.Context()` - Get the request context
- `r.PrintJson(v)` - Write JSON response
- `r.Auth()` - Get the auth struct (requires JWT middleware)
- `r.Validate()` - Validate the request
- `r.GetInputs()` - Get all form inputs as a map


## Authenticated user
Get the authenticated user from the request (only if the `JWTMiddleware` is added to the route):

```go
func (c *VideoCallController) Show(request Request) {
    user := new(db.User).DB(request.R.Context()).
        Where("id", request.Auth().GetUserId()).
        First()
}
```

See the [Authentication](authentication.md) section for details on login and middleware setup.


## Request body

Going makes it easy to receive data from the user, whether it's JSON or form data. Going checks the `Content-Type` header and parses accordingly.

Define a struct for the request body:

```go
type Message struct {
    ActiveRecord[*Message] `json:"-" swaggerignore:"true" validate:"-"`
    Type                   int64  `json:"type" form:"type" validate:"required"`
    Chat_id                int64  `json:"chat_id" json:"chat_id" validate:"required"`
    Sender_id              int64  `json:"sender_id" json:"sender_id" validate:"required"`
    Messages               string `json:"messages" json:"messages" validate:"required"`
}
```

The struct does not need to be an `ActiveRecord` - any struct with property tags works.

Use `RequestBody[T]` in your controller to automatically parse the body:

```go
func (c *MessageController) Create(request RequestBody[db.Message]) {
    result, err := request.Body.DB(request.R.Context()).Save()
    if err != nil {
       return Fail.StatusBadRequest(err.Error())
    }
    Response.PrintJson(result)
}
```

`request.Body` returns the parsed struct after Going has deserialized the request body into it.


## Validation

Validate the request body using the built-in validation:

```go
func (c *MessageController) Create(request RequestBody[db.Message]) {
    if result := request.Validate(); result.HasError {
        return Response.PrintJson(result.GetErrors())
    }
    // proceed with valid data...
}
```

See the [Validation](validation.md) section for details on validation rules and custom validators.


## Reflection-based parameter injection

Going uses reflection to automatically resolve function parameters in controller methods. When a route is registered with a method name, the framework inspects the function signature and injects the appropriate arguments:

### Supported parameter types

| Type | Resolution |
|------|-----------|
| `http.ResponseWriter` | Injected directly from the request |
| `*http.Request` | Injected directly from the request |
| `request.Requestbase` | Wraps `w` and `r` |
| `request.RequestBodybase[T]` | Parses request body into `T` |
| `int` | Parsed from URL path parameters |
| `string` | Taken from URL path parameters |

### Example

```go
// Route definition
webrouter.GET("/users/{id}", userController.Show)

// Controller method - parameters are injected automatically
func (c *UserController) Show(r Request, id int) {
    user := new(db.User).DB(r.R.Context()).Where("id", id).First()
    return Response.PrintJson(user)
}
```

The `id` parameter is automatically extracted from the URL path `/users/123` and passed as an `int` to the controller method.
