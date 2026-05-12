# Middelware 

- [Introduction](#introduction)

- [Creating a middleware](#creating-a-middleware)

- [Middleware on route](#middleware-on-route)

- [middleware groups](#middleware-groups)

- [Register middleware](#register-middleware)


<a name="introduction"></a>
## Introduction

this section describe how to use the middelware functionality in going. IT assumed you have read the section on router before this section. Middleware here follow the same convention as in Go's standard http.ServeMux middelware convention. and example of a middleware can be seem below:

<a name="creating-a-middleware"></a>
## Creating a middleware


```go
func MiddlewareTwo(next http.HandlerFunc) http.HandlerFunc {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {

		// Action before reaching the http handler 
		next.ServeHTTP(w, r)
        // Action after the request reach the http handler
	})
}
```
middleware can be placed anywhere, but is recommended to be placed here: `app/http/middleware/handler.go` or create you own go file with a new middleware


<a name="middleware-on-route"></a>
## Middleware on route
This example demostrate how to attach middlewares to a single path, or adding multiple. or creating  a group with a given middle for all, and then add a custom middleware for a single on. 
If you have many routes, you should create a new route file, with a new route, and add seperate with a prefix and the nedded middleware, see next section

```go
func Samplerouter() *web.MyRouter {

	SampleRouter := web.NewMyRouter()
	var sampleController = SampleController{}

	SampleRouter.GET("/nicklas", func(w http.ResponseWriter, r *http.Request) {

		fmt.Fprint(w, "passed")
	}).AddMiddleware(stdMiddleware.Cors)

	SampleRouter.GET("/hejverden/{id}/{name}", sampleController.APIONE).
		AddMiddleware(middleware.MiddlewareFour).
		AddMiddleware(middleware.MiddlewareFive)

	// -------------------- API -------------------//
	SampleRouter.GroupsWithMiddleware("/api", middleware.AuthMiddleware,
		SampleRouter.GET("/one", sampleController.APIONE).AddMiddleware(middleware.LoggingMiddleware),
		SampleRouter.GET("/two", sampleController.APIONE),
	)
	return SampleRouter
}
```

<a name="groups-and-more"></a>
## Middleware groups

You can define a group of middleware, it recommended to define on on this path: `app/http/middleware/groups.go`

```go
func WebMiddlewareGroup() stdlibmiddleware.MiddlewareGroup {
	return stdlibmiddleware.Chain(
		MiddleOne,
		MiddlewareTwo,
		stdlibmiddleware.Cors,
		LoggingMiddleware,
		inertija.InertiaMiddleware,
	)
}
```
<a name="register-middleware"></a>
## Register middleware
Register middleware on the router: in the path: `app/providers/routeservice.go` is here where you can register middleware for an entire group of router maps, Se the example below. The example show how to add an entire middleware group, and a single middleware the router. also, it demo how to add a prefix to a route. 
```go
type RouteServiceProvider struct {
}

func (route *RouteServiceProvider) Map(r *http.ServeMux) {
	
    // register the general Web provider
    mapwebRoute().RegisterRoutes(r)   
    
    // sample workspace for testing
	mapSampleRoute().RegisterRoutes(r) 
}


// Define the "web" route
func mapwebRoute() *webstdlib.MyRouter {
	return webrouter.Webrouter().
		AddmiddlewareGroup(middleware.WebMiddlewareGroup()). 
		Addmiddleware(middleware.MiddlewareThree).           
		Addmiddleware(internalMiddelware.CsrfMiddleware)
}

// define the map sample route 
func mapSampleRoute() *webstdlib.MyRouter {
	return webrouter.Samplerouter().
		AddmiddlewareGroup(middleware.WebMiddlewareGroup()).
		Addmiddleware(internalMiddelware.JWTMiddleware).
		Addprefix("/sample")
} 
```