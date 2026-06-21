# Mixins 

- [Introduction](#introduction)
- [Case](#case)


## Introduction
Mixin or Concerns (in rails) is just a way to add more functionality to an object. 
Go use this, naturally, by composition. Going add little extra to this pattern, and try to demostrate how we think this would be usefull for your application by a demostration.

## Case: 
Let say, in your application, a user can have an image, a post, a comment an so on. how would you implement this, without adding extra column to the database table? 
We use what rails calls for Polymorphic Associations, and here is how: 

The following steps is: 
1. Create the DB struct 
2. Create mixins struct 
3. Create the Repository where you add the DB and the Mixin


Create the DB struct, for this, we using a user struct and a image. This lives in the DB package.
```go 
type Picture struct {
	ActiveRecord[*Picture] `json:"-" swaggerignore:"true" validate:"-"`
	Name                   string `json:"name" validate:"required"`
	Imageable_type         string
	Imageable_id           int64
}

// Create a DB object for DB actions
func (_picture Picture) DB(ctx context.Context) *Picture {
	picture := &_picture
	picture.Table = "pictures"
	picture.Columns = picture.columns()
	picture.ParentDB = CreateORM(ctx, picture)
	return picture
}

func (picture *Picture) columns() Columns {
	return Columns{
		// Column		  "values"
		"name":           &picture.Name,
		"imageable_type": &picture.Imageable_type,
		"imageable_id":   &picture.Imageable_id,
	}
}

// user struct
type User struct {
	ActiveRecord[*User] `json:"-" swaggerignore:"true" validate:"-"`
	Name                string `json:"name" validate:"required"`
	Age                 int64  `json:"age" validate:"min=0,max=99"`
	Email               string `json:"email" validate:"required"`
	Password            string `json:"password" validate:"required" hidden:"true"`
	SessionToken        string `json:"-" hidden:"true"`
}

func (_user User) DB(ctx context.Context) *User {
	user := &_user
	user.Table = "users"
	user.Columns = user.columns()
	user.ParentDB = CreateORM(ctx, user)
	return user
}

func (user *User) columns() Columns {
	return Columns{
		// Column		  "values"
		"name":         &user.Name,
		"email":        &user.Email,
		"password":     &user.Password,
		"sessiontoken": &user.SessionToken,
	}
}
```


This is the mixin, living in the package mixins: 
```go 

package mixins

import (
	"concernDemo/internal/app/models/db"
	"context"

	. "github.com/nicklasjeppesen/going_internal/super/collections"
	. "github.com/nicklasjeppesen/going_internal/super/db/types"
)

// user implementations
type ImageConern[A IConcern[A]] struct { 
	*Concern[A]
}

func (q *ImageConern[A]) NewImage(concernType A) {
	q.Concern = &Concern[A]{IConcern: concernType}
}

// optional, just for demostrating you can create custom Queries.
func (s *ConcernsImplementation[A]) Scope() A {
	return s.Where("id", 2)
}

func (s *ImageConern[A]) Pictures() Collection[*db.Picture] {
	return db.Picture{}.DB(context.Background()).
		Where("imageable_type", s.GetName()).
		Where("imageable_id", s.PrimaryKey()).
		Get()
}
```

What you see here, is we use a structure, where we get image, that is equal the foreign relation type (name) and id, which we can get from the concern type. Now it time for using creating our repository, living in the package repository: 

```go 
type UserRepo struct {
	*db.User // repository DB type 
	mixins.ImageConern[*db.User] // the mixin type, have to be a ActiveRecord like the user
}

func (_userRepo UserRepo) DB(ctx context.Context) *UserRepo {
	userRepo := new(UserRepo)
	userRepo.User = new(db.User).DB(context.Background()).First()

	userRepo.ImageConern.NewImage(userRepo.User)
	return userRepo
}
```

Here is how it might get used: 

```go 

userService := repository.UserRepo{}.DB(context.Background())
userImage := userService.Pictures() // return a list of pictures
fmt.Println(userImage[0].Name)      // Get the name of the first picture //

// Create a new user object, or update
userService.Name = "Nicklas"
userService.Update() // or .Save(), for storing new entry
```