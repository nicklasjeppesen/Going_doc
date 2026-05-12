# ORM - Database access

-[Introduction](#introduction)

-[Setup](#setup)

-[Get request](#get-request)

-[Add routes](#add-routes)



<a name="introduction"></a>
## Introduction

This section introduce how tod use the the build in ORM for Going. 
This is very much inspired on how other framework ORM work, and is created to 
be a payoff between performance and developerment speed. This ORM does not use GO's build in reflection, this is to avoid performance problem but gives som extra time on setup of a database. 


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
```go 
var users = User{}.DB().Get() // return all users. 
```

<a name="Get-request"></a>
## Create/Update request
how to use The ORM: 
```go 
var users = User{}.DB() 
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
