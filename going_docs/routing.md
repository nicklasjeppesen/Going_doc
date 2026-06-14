# Router

- [Introduction](#introduction)
- [use of basis router](#use-of-basis-router)
- [Create new router group](#create-new-router-group)


## Introduction
The Going web router, is a basis router. 


for simple hello world, go to the folder routes and choose web.go 
```go 
webrouter.GET("/hello", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "Hello world")
})
```

### Types: 
The router support all the classic route types, as Go servemux support, and will also be registered as you was writing pure servemux

```go 
webrouter.GET()
webrouter.POST()
webrouter.OPTIONS()
Webrouter().DELETE()
Webrouter().PATCH()
```



### Route to Controller: 
You can direct a route to a controller method, like this

```go
func Webrouter() *web.MyRouter {

	webrouter := web.NewMyRouter()

	var homeController = HomeController{}
	webrouter.GET("/", homeController.Home).Name("home.front")
	return webrouter
}
```
The controller will look like this: 

```go 
package controller

type HomeController struct {
	ControllerBase
}

func (c *HomeController) Home() Result {
	return View("index", Params{"Title": "Going App", "Message": "Welcome to Going"})
}
```

### Naming route: 
You can add a name to a route, that can be used later, like this: 

```go 
webrouter.GET("/", homeController.Home).Name("home.front")
```
See db for further use. 


### Route input 

You can also add a parameter to the route like this: 
 
```go 
webrouter.GET("/call/room/{roomId}", videoCallController.Show)
```
Which in the controller is handle like this: 

```go
func (c *VideoCallController) Show(request Request, roomId string) Result {
	var user = new(db.User).DB(request.R.Context()).
		Where("id", request.Auth().GetUserId()).
		First()

	return View("call.show", Params{"room": roomId, "user": user})
}
```
NOTE: it is important that the parameter name is equal to the name on the route URL, thats the only way, Going can parse this. 

##  Middleware 

For a single URL, you can add a middleware with dot after the router: 
```go 
	webrouter.GET("/protected", loginController.Protected).AddMiddleware(middleware.JWTMiddleware)
```

### Route groups
If you need to add a prefix, or middleware for a large groups of URLs, like user need to be login, you can register an entire webgroup 

Create new file in routes / or other folder you like. create a function that return a *web.MyRouter, add your URLs. For going knowing these URL's, go to ``` internal/app/provider/route/routeservice.go```

Start by create a new function that return *webstdlib.MyRouter, the function should return your newly created routergroup. 

```go 
func mapSampleRoute() *webstdlib.MyRouter {
	return webrouter.Samplerouter(). // Your new router
		AddmiddlewareGroup(middleware.WebMiddlewareGroup()). // Optional, to return an 
		Addmiddleware(internalMiddelware.JWTMiddleware). // optional, Can be call multiple times 
		Addprefix("/sample") // Optional, create a prefix, for an entire routing group
}
```
When you have create the function, it time to register your routes, That is done in the function map, where your routing group, have a method that take GO's router, r *http.Sermux, and register all your routes. 

```go
func (route *RouteServiceProvider) Map(r *http.ServeMux) {
	mapwebRoute().RegisterRoutes(r)    // register the general Web provider
	mapSampleRoute().RegisterRoutes(r) // register new workspace
}
```
And then you are done! 