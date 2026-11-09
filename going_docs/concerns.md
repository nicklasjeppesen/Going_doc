# Mixins / Concerns

- [Introduction](#introduction)
- [Case: Polymorphic Associations](#case-polymorphic-associations)
  - [Step 1: Create the DB struct](#step-1-create-the-db-struct)
  - [Step 2: Create the mixin struct](#step-2-create-the-mixin-struct)
  - [Step 3: Create the repository](#step-3-create-the-repository)
  - [Usage](#usage)
- [Case: Recording domain events](#case-recording-domain-events)


## Introduction

Mixins (called "Concerns" in Rails) are a way to add functionality to an object through composition. Go does this naturally, and Going adds a small layer on top to support polymorphic associations.


## Case: Polymorphic Associations

Let's say in your application, a user can have images (a post, a comment, etc.). How would you implement this without adding extra columns to every database table?

We use what Rails calls "Polymorphic Associations". The steps are:

1. Create the DB struct
2. Create a mixin struct
3. Create the repository where you add the DB and the Mixin


### Step 1: Create the DB struct

Create the models in the DB package:

```go
// Picture model
type Picture struct {
    ActiveRecord[*Picture] `json:"-" swaggerignore:"true" validate:"-"`
    Name                   string `json:"name" form:"name" validate:"required"`
    Imageable_type         string `json:"imageable_type" form:"imageable_type" validate:"required"`
    Imageable_id           int64 `json:"imageable_id" form:"imageable_id" validate:"required"`
}

func (_picture Picture) DB(ctx context.Context) *Picture {
    picture := &_picture
    picture.Table = "pictures"
    picture.Columns = picture.columns()
    picture.ParentDB = CreateORM(ctx, picture)
    return picture
}

func (picture *Picture) columns() Columns {
    return Columns{
        "name":           &picture.Name,
        "imageable_type": &picture.Imageable_type,
        "imageable_id":   &picture.Imageable_id,
    }
}

// User model
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
        "name":         &user.Name,
        "email":        &user.Email,
        "password":     &user.Password,
        "sessiontoken": &user.SessionToken,
    }
}
```


### Step 2: Create the mixin struct

Create the mixin in a `mixins` package:

```go
package mixins

import (
	"concernDemo/internal/app/models/db"
	"context"

	. "github.com/nicklasjeppesen/going_internal/super/collections"
	. "github.com/nicklasjeppesen/going_internal/super/db/types"
)

type ImageConcern[A IConcern[A]] struct {
    *Concern[A]
}

func (q *ImageConcern[A]) NewImage(concernType A) {
    q.Concern = &Concern[A]{IConcern: concernType}
}

// Custom query example
func (s *ImageConcern[A]) Scope() A {
    return s.Where("id", 2)
}

func (s *ImageConcern[A]) Pictures() Collection[*db.Picture] {
    return db.Picture{}.DB(s.GetCtx()).
        Where("imageable_type", s.GetName()).
        Where("imageable_id", s.PrimaryKey()).
        Get()
}
```

The mixin queries the `Picture` table using the concern type name and primary key.


### Step 3: Create the repository

Create the repository that composes the DB type with the mixin:

```go
type UserRepo struct {
    *db.User                       // repository DB type
    mixins.ImageConcern[*db.User]  // the mixin type
}

func (_userRepo UserRepo) DB(ctx context.Context) *UserRepo {
    userRepo := new(UserRepo)
    userRepo.User = new(db.User).DB(ctx).First()
    userRepo.ImageConcern.NewImage(userRepo.User)
    return userRepo
}
```


### Usage

```go
userService := repository.UserRepo{}.DB(context.Background())
userImage := userService.Pictures() // returns a list of pictures
fmt.Println(userImage[0].Name)

// Create a new user object, or update
userService.Name = "Nicklas"
userService.Update() // or .Save() for storing a new entry
```


## Case: Recording domain events

Some behaviour every model needs is built into `ActiveRecord` itself, so it needs no mixin. Recording domain events is one: every model has `Record(event)` and `ReleaseEvents()`, and is an `events.Recorder` (`super/events`). A method can then say what happened without publishing anything. The code that saved the model publishes the events afterwards, see [Recording events on the model](events.md#recording-events-on-the-model).

### The model

A rich model that embeds a row gets the methods through it, next to its other concerns:

```go
type Todo struct {
    *db.Todo // embeds ActiveRecord[*db.Todo]: Record and ReleaseEvents
    mixins.Fileable[*db.Todo]
}

func (t *Todo) Trash(at time.Time) bool {
    if t.Trashed() {
        return false
    }
    t.Trashed_at = stamp(at)
    t.Record(TodoTrashed{TodoID: t.Id, Title: t.Title})
    return true
}
```

### Usage

```go
t, _ := todos.Find(ctx, id)
if t.Trash(time.Now()) {
    todos.Save(ctx, t)                              // save first
    publisher.PublishAll(ctx, t.ReleaseEvents()...) // then publish what was recorded
}
```

Things to know:

- `Save()` and `Update()` never publish and never write the events. They are kept in an unexported field, so they are no column and are not in `ToJson()`.
- `ReleaseEvents()` empties the list, so the same event is never published twice.
- `DB(ctx)` keeps the events when it copies its receiver (`picture := &_picture`, as in the example above). Rows loaded with `First()` or `Get()` start without events.
