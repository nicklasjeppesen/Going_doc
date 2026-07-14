# Dependency Injection Container

- [Introduction](#introduction)
- [Creating a container](#creating-a-container)
- [Registering services](#registering-services)
- [Using with the router](#using-with-the-router)
- [Resolving in controllers](#resolving-in-controllers)
- [Singleton pattern](#singleton-pattern)


## Introduction

Going includes a dependency injection (DI) container that automatically resolves controller dependencies. This follows a singleton pattern where services are created once and cached for subsequent resolutions.


## Creating a container

```go
import "github.com/nicklasjeppesen/going_internal/super/customrouter"

container := customrouter.NewContainer()
```


## Registering services

Register services by providing a zero-arg factory function. The return type becomes the key for resolution:

```go
// Register a user service
container.Register(func() *services.UserService {
    return services.NewUserService()
})

// Register a mail service
container.Register(func() *services.MailService {
    return services.NewMailService()
})

// Register a cache service
container.Register(func() *cache.Cache {
    return cache.NewRedisCache()
})
```

Dependencies between services are resolved lazily. If `UserService` depends on `MailService`, the container will create `MailService` first and inject it when `UserService` is resolved.


## Using with the router

Wire the container into your router:

```go
webrouter := customrouter.NewMyRouter()
webrouter.UseContainer(container)
```

For WebSocket routers:

```go
socketRouter := socket.NewSocketRouter()
socketRouter.UseContainer(container)
```


## Resolving in controllers

When a controller has a `Loader()` method, the framework resolves its parameters from the container:

```go
type UserController struct {
    customrouter.BaseController
    userService *services.UserService
    mailService *services.MailService
}

// The Loader method is called before any action method
// Parameters are resolved from the DI container
func (c *UserController) Loader(userService *services.UserService, mailService *services.MailService) *UserController {
    c.userService = userService
    c.mailService = mailService
    return c
}

func (c *UserController) Store(r request.Requestbase) {
    // Use injected services
    user := c.userService.Create(r.R.FormValue("email"))
    c.mailService.SendWelcomeEmail(user.Email)
}
```

The `Loader` method must return a pointer to the controller (or a compatible type). The framework calls this method before any action, ensuring services are available.


## Singleton pattern

The container uses a singleton pattern:

1. First resolution: The factory function is called, the result is cached
2. Subsequent resolutions: The cached instance is returned

This means all controllers that depend on the same service will share the same instance. If you need per-request instances, consider creating factory methods within the service itself.

```go
// This creates ONE instance that is reused
container.Register(func() *services.UserService {
    return services.NewUserService()
})
```
