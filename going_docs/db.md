# ORM - Database access

- [Introduction](#introduction)
- [Setup](#setup)
- [Get request](#get-request)
- [Add routes](#add-routes)



<a name="introduction"></a>
## Introduction

This section introduce how to use the the build in ORM for Going. 
This is very much inspired on how other framework ORM work, and is created to 
be a payoff between performance and developerment speed.  


<a name="setup"></a>
## Setup
For creating a new ORM, you have to define a new struct. Its strongly recommended you add this struct to the path ```/app/models/db/```.The struct include the ActiveRecord struct, which is a take a generic type, that shall be the new ORM you create: 

In the example: you start by defining the struct, every field kan be a column in the database. ActiveRecord has to take the type of a pointer to current struct. 

In the DB function which is required, you defined a your ORM, with table name and Columns. 

```go 
type User struct {
	ActiveRecord[*User] `json:"-" swaggerignore:"true"`
	Name                string `json:"name" validate:"required"`
	Age                 int64  `json:"-" validate:"min=0,max=99"`
	Email               string `json:"email" validate:"required"`
	Password            string `json:"password" validate:"required" hidden:"true"`
	SessionToken        string `json:"-" hidden:"true"`
	Company_id          int64  `json:"Company_id" validate:"required"`
}

// Creating DB instance. 
func (_user User) DB() *User {
	user := &User{}
	user.Table = "users"
	user.Columns = user.Columns()
	
	// Creating DB
	user.ParentDB = CreateORM(user)
	return user
}

func (user *User) columns() Columns {
	return Columns{
		// Column		  "values"
		"age":          &user.Age,
		"name":         &user.Name,
		"email":        &user.Email,
		"password":     &user.Password,
		"sessiontoken": &user.SessionToken,
		"company_id":   &user.Company_id,
	}
 }
```



<a name="Get-request"></a>
## Get request
how to use The ORM:
Its important to call the DB before starting adding info.  
The DB take in the GO context, which has been a standard for GO DB. So bare so to addd a context the the DB. 
```go 
var users = User{}.DB(context.Background()).Get() // return all users. 
```

<a name="Get-request"></a>
## Create/Update request
how to use The ORM: 
```go 
var users = User{}.DB(context.Background()) 
user.Name ="Nicklas" 
user.Age = 30
user.Email = "njep@Going.com"
savedUser := user.Save() // stored id, created_at, updated_at in the new saveUser with more info

savedUser.Age = 34 // im not 30 
SavedUser.Update() // update the current instance. 
```





<a name="addRoutes"></a>
## Add Routes

Either for API or for genereal request it can be usefull if the model can include what action can be exccuted for a given object. also known as Level 3 in Richardson Maturity model level 3: https://martinfowler.com/articles/richardsonMaturityModel.html#level3

This can be done because Going ORM and routing is working together to include actions. Example: 

in your router filer ex: `routes/web.go` you can have the following: 

```go
func Samplerouter() *web.MyRouter {

	SampleRouter := web.NewMyRouter()
	var sampleController = SampleController{}

	SampleRouter.GET("/sample/index", sampleController.ShowUser).Name("sample.index")
	SampleRouter.POST("/sample/store", sampleController.Store).Name("sample.store")
	SampleRouter.GET("/sample/show/{id}", sampleController.ShowUser).Name("sample.ShowInput")
}
```
This is your routes, in your controller, you can define your model like this: 

```go
type SampleController struct {
	ControllerBase
}

func (c *SampleController) Get() Result {
	user := models.User{}.DB().AddRoutes("sample").Get()
	return Response.PrintJson(user)
}
```
The way this is works is, Addroutes will do two things. firs step: in the function addRoutes you define a string called sample, it will look for all routes with the name starting the this prefix "sample". Thats how it get a list of routes. 
Second, for each object it will replace the input {id} with a concret value. The way this is done is, it will check if if the model has this property id. (DB column) and then replace it. 


### Foreign unknown key: 
What if in your URL you have an foreign value, that is not part of the model?  
in the addRoutes function has a optional input parameter, which is a ```map[string]string``` That is the type of key:id, value: foreignID, which will the same for all the objects. Which is usefull for URL that have a parent. example: 

```go
func Samplerouter() *web.MyRouter {

	SampleRouter := web.NewMyRouter()
	var sampleController = SampleController{}

	SampleRouter.GET("/sample/index", sampleController.ShowUser).Name("sample.index")
	SampleRouter.POST("/sample/store", sampleController.Store).Name("sample.store")
	SampleRouter.GET("/sample/show/{id}", sampleController.ShowUser).Name("sample.ShowInput")

    SampleRouter.GET("/sample/group/{groupId}/show/{id}", sampleController.ShowUser).Name("sample.ShowInput")
}
```
This is your routes, in your controller, you can define your model like this: 

```go
type SampleController struct {
	ControllerBase
}

func (c *SampleController) Get() Result {
	user := models.User{}.DB().AddRoutes(
        "sample",
         map[string]string{
            "groupId" : "1"}).
            Get()
	return Response.PrintJson(user)
}
```
Which will return the routes and replace the groupId with 1.



## Update 
The update is straight forward, just get the DB record, change the value, and call the update, the ORM will use the primary key for updating the record. See the example below: 

```go 
var user = db.User{}.DB(context.Background()).Where("id", 1).First()
	user.Age = 1000
	if err := user.Update(); err != nil {
		// Handle error
	}
```

## Delete a record
Delete a record, is as easy as update One, get it, and call delete.
```go 
user := new(db.User).DB(context.Background()).Where("id", 3).First()
	user.Delete()
```


## Select
If you want to make a custom SQL query, you can choose to Create your struct, and set the DB, and call the Select function, and this will be a clean SQL query, with no assumtion about table, colums and so on. The only thing is that, it will know what DB to connect to. 

## Pagination: 
It really simple to use pagination in Going ORM, use call the Pagination function, with the number of element per page, and then Going will handle the rest. 

```go 

//url: /sample
func (c *SampleController) Get(r Request) Result {

	users := db.User{}.DB(r.R.Context()).Pagination(r.R, 3)
	return Response.PrintJson(users)
}
```

Example 
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
        },
        {
            "created_at": "2026-06-14T16:41:07Z",
            "email": "nanna@live.dk",
            "id": 2,
            "name": "Nanna",
            "pivots": null,
            "routes": null,
            "updated_at": "2026-06-17T20:49:50Z"
        },
        {
            "created_at": "2026-06-18T19:23:26Z",
            "email": "mom@live.dkk",
            "id": 4,
            "name": "mom",
            "pivots": null,
            "routes": null,
            "updated_at": "2026-06-18T19:23:26Z"
        }
    ],
    "first_page_url": "/sample?page=1",
    "next_page_url": "/sample?page=2",
    "path": "/sample",
    "per_page": 3,
    "prev_page_url": ""
}
```