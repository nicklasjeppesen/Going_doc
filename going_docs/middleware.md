# Middleware

- [Introduction](#introduction)
- [Middleware type](#middleware-type)
- [Creating a middleware](#creating-a-middleware)
- [Built-in middleware](#built-in-middleware)
- [Middleware on route](#middleware-on-route)
- [Middleware groups](#middleware-groups)
- [Register middleware](#register-middleware)


## Introduction

This section describes how to use the middleware functionality in Going. Middleware follows the convention of wrapping request handlers to add cross-cutting concerns like authentication, CSRF protection, and CORS.


## Middleware type

Going defines middleware as a function that takes a `request.Handler` and returns a `request.Handler`:

```go
type Middleware func(request.Handler) request.Handler
```

A `MiddlewareGroup` is a slice of middleware:

```go
type MiddlewareGroup []Middleware
```

You can create a middleware group with `Chain()`:

```go
group := middleware.Chain(
    middlewareOne,
    middlewareTwo,
    middleware.CorsMiddleware,
)
```


## Creating a middleware

Create a middleware by wrapping the request handler:

```go
package middleware

import "github.com/nicklasjeppesen/going_internal/super/request"

func LoggingMiddleware(next request.Handler) request.Handler {
    return func(req *request.Requestbase) {
        // Action before reaching the handler
        log.Printf("Request: %s %s", req.R.Method, req.R.URL.Path)

        next(req)

        // Action after the request is handled
    }
}
```

Middleware can be placed anywhere, but is recommended to be placed at `app/http/middleware/`.


## Built-in middleware

Going provides several built-in middleware in the `super/middleware` package:

### JWTMiddleware
Reads the session cookie, verifies the JWT, and injects the user ID into the request context:

```go
webrouter.GET("/protected", controller.Protected).AddMiddleware(middleware.JWTMiddleware)
```

### CsrfMiddleware
For GET requests, generates a CSRF token and sets it as a cookie + context. For POST/PUT/DELETE, validates the token from the cookie against the form body or JSON payload:

```go
webrouter.POST("/submit", controller.Submit).AddMiddleware(middleware.CsrfMiddleware)
```

### CorsMiddleware
Sets CORS headers if the `Origin` header matches an allowed origin. Handles OPTIONS preflight requests:

```go
webrouter.GET("/api/data", controller.Data).AddMiddleware(middleware.Cors("http://localhost:3000"))
```

### PanicRecovery
Catches panics and renders a rich error page in debug mode, or a plain "Internal Server Error" in production:

```go
webrouter.GET("/", controller.Index).AddMiddleware(middleware.PanicRecovery)
```


## Middleware on route

Attach middleware to a single path using `.AddMiddleware()`. You can chain multiple middlewares:

```go
func SampleRouter() *customrouter.MyRouter {
    SampleRouter := customrouter.NewMyRouter()

    SampleRouter.GET("/nicklas", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprint(w, "passed")
    }).AddMiddleware(middleware.CorsMiddleware)

    SampleRouter.GET("/data/{id}/{name}", sampleController.GetData).
        AddMiddleware(middleware.LoggingMiddleware).
        AddMiddleware(middleware.JWTMiddleware)

    return SampleRouter
}
```


## Middleware groups

Define reusable middleware groups, recommended at `app/http/middleware/groups.go`:

```go
package middleware

import (
    stdlibmiddleware "github.com/nicklasjeppesen/going_internal/super/middleware"
)

func WebMiddlewareGroup() stdlibmiddleware.MiddlewareGroup {
    return stdlibmiddleware.Chain(
        LoggingMiddleware,
        stdlibmiddleware.PanicRecovery,
    )
}
```

Groups can be applied to route maps:

```go
SampleRouter.GroupsWithMiddleware("/api", middleware.JWTMiddleware,
    SampleRouter.GET("/one", sampleController.One).AddMiddleware(middleware.LoggingMiddleware),
    SampleRouter.GET("/two", sampleController.Two),
)
```


## Register middleware

Register middleware on the router in `app/providers/routeservice.go`:

```go
type RouteServiceProvider struct{}

func (route *RouteServiceProvider) Map(r *http.ServeMux) {
    // register the general Web provider
    mapwebRoute().RegisterRoutes(r)

    // register new route group
    mapSampleRoute().RegisterRoutes(r)
}

// Define the "web" route
func mapwebRoute() *customrouter.MyRouter {
    return webrouter.Webrouter().
        AddmiddlewareGroup(middleware.WebMiddlewareGroup()).
        Addmiddleware(middleware.CsrfMiddleware)
}

// Define the sample route
func mapSampleRoute() *customrouter.MyRouter {
    return webrouter.Samplerouter().
        AddmiddlewareGroup(middleware.WebMiddlewareGroup()).
        Addmiddleware(middleware.JWTMiddleware).
        Addprefix("/sample")
}
```
