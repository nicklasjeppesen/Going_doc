# Authentication

- [Introduction](#introduction)
- [Login](#login)
- [Get Authenticated user](#get-authenticated-user)
 

## Introduction

Going Support basis autentication, that you would expect from a simple application. 


## Register user
For register a new user, you should just make a post request to the server. 
For storing password, you can use the build in functionality of hashPassword, for hashing a user password. 
```go 
// Post method for register a new user
func (register *RegisterController) Register(r Request) Result {
	user := new(models.User).DB(r.R.Context())
	user.Name = r.R.FormValue("name")
	user.Email = r.R.FormValue("email")
	user.Password = security.HashPassword(r.R.FormValue("password"))
	user, _ = user.Save()
}
```



## Login
For Login, you should create a simple post form, with email, and password, and send a request to the server,

```go 
// Post action for login
func (login *LoginController) Login(r Request) Result {
	auth := r.Auth()
	auth.Email = r.R.FormValue("email")
	auth.Password = r.R.FormValue("password")

	if auth.Attempt() {
		return Response.Redirect("/protected")
	} else {
		return Response.WithErrors(
			Params{
				"message": "Invalid credentials",
				"email":   "Email findes ikke"}).
			Back()
	}
}
```

## Get Authenticated user

For getting the Authenticated user, the first thing, is you have to add the authentication middleware to your route, that setting the information. 
add the middleware JWTMiddleare to your route Like this, See the section middleware for more options.

```go 
webrouter.GET("/protected", loginController.Protected).AddMiddleware(middleware.JWTMiddleware)
```

Now you can get an authenticated in the controller like this
```go 
func (home *HomeController) Index(request Request) {
	authUserId := request.Auth().GetUserId()
	fmt.Println("userId: ", authUserId)
}
```
