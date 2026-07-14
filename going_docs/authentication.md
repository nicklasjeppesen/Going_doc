# Authentication

- [Introduction](#introduction)
- [Register user](#register-user)
- [Login](#login)
- [Logout](#logout)
- [Get Authenticated user](#get-authenticated-user)


## Introduction

Going supports basic authentication with JWT-based session tokens and bcrypt password hashing. The auth system is built into the `super/auth` package and works with the middleware layer.


## Register user
For registering a new user, make a POST request to the server. Use `security.HashPassword()` to hash the password before storing it.

```go
// Post method for registering a new user
func (register *RegisterController) Register(r Request) Result{
    user := new(models.User).DB(r.R.Context())
    user.Name = r.R.FormValue("name")
    user.Email = r.R.FormValue("email")
    user.Password = security.HashPassword(r.R.FormValue("password"))
    user.Save()

 // return view
}
```



## Login
For login, create a POST form with email and password, and send a request to the server.

The `auth.Auth` struct handles the authentication flow: it looks up the user by email, verifies the password hash, generates a JWT token, stores it in the database, and sets it as a cookie.

```go
// Post action for login
func (login *LoginController) Login(r Request) Result{
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

To get the authenticated user, add the `JWTMiddleware` to your route. This middleware reads the session cookie, verifies the JWT, extracts the user ID, and injects it into the request context.

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

You can then use this ID to fetch the full user from the database.:

```go
func (c *VideoCallController) Show(request Request) {
    user := new(db.User).DB(request.R.Context()).
        Where("id", request.Auth().GetUserId()).
        First()
    // use user...
}
```
