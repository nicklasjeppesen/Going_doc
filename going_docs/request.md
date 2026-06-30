# Request

- [Introduction](#introduction)



## Introduction 
This section describe a request can be handle using Going build in request method. 

## Basis GO request 

```go
auth.Email = r.R.FormValue("email")
auth.Password = r.R.FormValue("password")
```

R is going Requet, so yo can retrive the request.R.Context(), and other information if needed. 

### Auth user
You can get the authenticated user from the request very easy (NOTE, only if the middleware JWTMiddleware is added to the route)

```go 
func (c *VideoCallController) Show(request Request) {
	var user = new(db.User).DB(request.R.Context()).
		Where("id", request.Auth().GetUserId()).
		First()
}
```

## Request body 
Going make it very easy to retrive data from the user, ex. data from user, no matter it json or form request, going will check the header, and decide where to parse the data from. Ex. 

```Go 
type Message struct {
	ActiveRecord[*Message] `json:"-" swaggerignore:"true" validate:"-"`
	Type                   int64  `form:"type" json:"type" validate:"required"`
	Chat_id                int64  `form:"chat_id" json:"chat_id" validate:"required"`
	Sender_id              int64  `form:"sender_id" json:"sender_id" validate:"required"`
	Messages               string `form:"messages" json:"messages" validate:"required"`
}
```
You can have this struct, dont need to be an ActiveRecord struct, with property tag. It can be used like this: 


```go 
func (c *MessageController) Create(request RequestBody[db.Message]) Result {
	if result, err := request.Body.DB(request.R.Context()).Save(); err == nil {
		return Response.PrintJson(result)
	} else {
		return Fail.StatusBadRequest(err.Error())
	}
}
```
Request.Body return the struct, after Going has parsed the data to it, then we set up the DB, where Active record first validate the data, and continue to save to the DB if no errors. 
OBS: remember to add the form tag, if you get info from a html form, or add json, if you get info from a json body