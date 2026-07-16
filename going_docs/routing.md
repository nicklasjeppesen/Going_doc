# Router

- [Introduction](#introduction)
- [Basic routing](#basic-routing)
- [Route types](#route-types)
- [Route to controller](#route-to-controller)
- [Controller structure](#controller-structure)
- [Naming routes](#naming-routes)
- [Route parameters](#route-parameters)
- [Middleware on routes](#middleware-on-routes)
- [Route groups](#route-groups)
- [DI Container](#di-container)
- [Controller lifecycle](#controller-lifecycle)


## Introduction

The Going web router wraps Go's `http.ServeMux` with Laravel-style features: route groups, path parameter extraction, named routes, middleware chaining, a DI container, and a controller lifecycle with before/after action hooks.


## Basic routing

In your routes folder (e.g. `routes/web.go`):

```go
webrouter.GET("/hello", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello world")
})
```


## Route types

The router supports all standard HTTP methods:

```go
webrouter.GET(path, handler, methodName...)
webrouter.POST(path, handler, methodName...)
webrouter.PUT(path, handler, methodName...)
webrouter.DELETE(path, handler, methodName...)
webrouter.PATCH(path, handler, methodName...)
webrouter.OPTIONS(path, handler, methodName...)
```


## Route to controller

You can direct a route to a controller method:

```go
func Webrouter() *customrouter.MyRouter {
    webrouter := customrouter.NewMyRouter()
    var homeController = HomeController{}

    webrouter.GET("/", homeController.Home).Name("home.front")
    return webrouter
}
```

When a `methodName` is provided as the last argument, the router uses reflection-based controller dispatch with DI and lifecycle hooks:

```go
webrouter.GET("/", homeController, "Home").Name("home.front")
```


## Controller structure

Controllers embed `BaseController` which provides before/after action hooks:

```go
package controller

import "github.com/nicklasjeppesen/going_internal/super/customrouter"

type HomeController struct {
    ControllerBase
}

func (c *HomeController) Home(r request.Requestbase) {
    Response.Print("Hello from Home")
}
```

The framework automatically resolves controller dependencies via the DI container and injects parameters into action methods using reflection. See [Controller lifecycle](#controller-lifecycle) for details.


## Naming routes

You can add a name to a route for URL generation later:

```go
webrouter.GET("/", homeController.Home).Name("home.front")
```

Using named routes in the ORM:

```go
users := models.User{}.DB(ctx).AddRoutes("home").Get()
```

See the [Database](db.md) section for more details on using named routes with the ORM.


## Route parameters

Add path parameters with curly braces. The parameter name must match the controller method argument name:

```go
webrouter.GET("/call/room/{roomId}", videoCallController.Show)
```

The controller receives the parameter via reflection:

```go
func (c *VideoCallController) Show(request Request, roomId string) {
    user := new(db.User).DB(request.R.Context()).
        Where("id", request.Auth().GetUserId()).
        First()

    // use roomId...
}
```

**Important**: The parameter name in the URL must exactly match the variable name in the controller method signature. The router extracts parameters by name from the URL path.


## Middleware on routes

Add middleware to a single route using `.AddMiddleware()`:

```go
webrouter.GET("/protected", loginController.Protected).AddMiddleware(middleware.JWTMiddleware)
```

Chain multiple middlewares:

```go
webrouter.GET("/admin", adminController.Index).
    AddMiddleware(middleware.JWTMiddleware).
    AddMiddleware(middleware.CorsMiddleware)
```


## Route groups

For adding prefixes or middleware to large groups of URLs, create route group functions and register them.

Create a new route file and define a function returning `*customrouter.MyRouter`:

```go
func mapSampleRoute() *customrouter.MyRouter {
    return webrouter.Samplerouter().
        AddmiddlewareGroup(middleware.WebMiddlewareGroup()).
        Addmiddleware(middleware.JWTMiddleware).
        Addprefix("/sample")
}
```

Register all route groups in your route service provider:

```go
type RouteServiceProvider struct{}

func (route *RouteServiceProvider) Map(r *http.ServeMux) {
    mapwebRoute().RegisterRoutes(r)
    mapSampleRoute().RegisterRoutes(r)
}
```

You can also use the `Group()` method to create sub-routers with path prefixes:

```go
func SampleRouter() *customrouter.MyRouter {
    r := customrouter.NewMyRouter()

    api := r.Group("/api")
    api.Middleware(middleware.JWTMiddleware)
    api.GET("/users", userController.Index)
    api.GET("/users/{id}", userController.Show)

    return r
}
```


## DI Container

Going includes a dependency injection container for resolving controller dependencies automatically.

### Register dependencies

Create a container and register your services:

```go
container := customrouter.NewContainer()

// Register a service - the return type becomes the key
container.Register(func() *services.UserService {
    return services.NewUserService()
})

// Register another service
container.Register(func() *services.MailService {
    return services.NewMailService()
})
```

### Use with router

Wire the container into your router:

```go
webrouter.UseContainer(container)
```

### Resolve in controllers

When a controller has a `Loader()` method, the framework resolves its parameters from the container:

```go
type UserController struct {
    ControllerBase
}

func (c *UserController) Loader(userService *services.UserService, mailService *services.MailService) *UserController {
    // Services are injected by the DI container
    c.userService = userService
    return c
}

func (c *UserController) Index(r request.Requestbase) {
    users := c.userService.GetAll()
    response.NewResponse().PrintJson(users).ServeHTTP(r.W, r.R)
}
```

The container uses a singleton pattern - dependencies are created once and cached for subsequent resolutions.


## Controller lifecycle

Controllers support before/after action hooks, inspired by Rails:

### Before actions

```go
type AdminController struct {
    ControllerBase
}

func (c *AdminController) init() {
    c.AddBeforeAction(customrouter.BeforeAction{
        Handler: func(r request.Requestbase) bool {
            // Return true to continue, false to abort
            userId := r.Auth().GetUserId()
            return userId > 0
        },
    })
}
```

### After actions

```go
func (c *AdminController) init() {
    c.AddAfterAction(customrouter.AfterAction{
        Handler: func(r request.Requestbase) {
            // Runs after the action method
            log.Println("Request completed")
        },
    })
}
```

### Filtering with Only/Except

Restrict hooks to specific action methods:

```go
c.AddBeforeAction(customrouter.BeforeAction{
    Handler: func(r request.Requestbase) bool {
        return r.Auth().GetUserId() > 0
    },
    Only: []string{"Index", "Show"}, // Only run for these actions
})

c.AddAfterAction(customrouter.AfterAction{
    Handler: func(r request.Requestbase) {
        log.Println("Action completed")
    },
    Except: []string{"Login"}, // Run for all actions except Login
})
```
