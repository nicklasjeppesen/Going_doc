# ORM - Database Relationships

- [Introduction](#introduction)
- [Setup](#Setup)
- [BelongsTo](#belongsto)
- [HasOne](#hasone)
- [HasMany](#hasmany)
- [Belongs to many](#belongs-to-many)
- [BelongsToMorph](#belongstomorph)


## Introduction
Going Activerecord a set of options for creating relationsship between your objects. Classic relationships you would know. 


## Belongsto 
A belongsto association sets up a relationship with another model, such that each instance of the declaring model "belongs to" one instance of the other model. For example, if your application includes company and users, and each user can be assigned to exactly one company, you'd declare the user model this way:

```go 
type User struct {
	ActiveRecord[*User] `json:"-" swaggerignore:"true" validate:"-"`
	Name                string `json:"name" validate:"required"`
	Age                 int64  `json:"age" validate:"min=0,max=99"`
	Email               string `json:"email" validate:"required"`
	Password            string `json:"password" validate:"required" hidden:"true"`
	SessionToken        string `json:"-" hidden:"true"`
	Company_id          int64  // foreign Id
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
// set up the relationship
func (user *User) Company() BelongsTo[*Company] {
	return NewBelongsTo(new(Company).DB(user.GetCtx()), user)
}
```

how to use this? 

```go 
user := new(db.User).DB(context.Background()).With("Company").First()
	fmt.Println(user.Company().Item()) // return Company struct
```

### Details
The magic is, here is, you create a new function Called Company, and in then add your current user and company. Going will assume you follow a given convention for naming your foreign_id DB column. Going will call company.GetName() function and add _id at the end, for looking for the id in user struct, and then get the company struct. 


## HasOne
A hasOne association indicates that one other model has a reference to this model. That model can be fetched through this association.

For example, if each company in your application has only one user, you'd declare the company model like this:

```go 
type Company struct {
	ActiveRecord[*Company] `json:"-" swaggerignore:"true" validate:"-"`
	Name                   string `json:"name" validate:"required"`
}
// DB & columns func

func (c *Company) User() HasOne[*User] {
	return NewHasOne(new(User).DB(c.GetCtx()), c)
}
```

### how to use it: 

``` go
company := new(db.Company).DB(context.Background()).With("User").First()
fmt.Println(company.User().Item()) // return user struct
```

You can also define a helper function, to avoid string conflict, like this: 
```GO 
func (c *Company) WithUser() *Company {
	return c.With("User")
}
```
Use:
```go
company := new(db.Company).DB(context.Background()).WithUser().First()
fmt.Println(company.User().Item()) // return user struct 
```


### Details 
Going asumme a given naming convention as before, for making this so simple. 
When you define "User" in which, it asumme you include the exact name of the User function. Going will call the Company.GetName() function, and add _id, and look up in the user table for a column where this columns is equal company.PrimaryId. 

## HasMany
Has Many, is similar to hasOne, just that, Going asumme there exists either 0 more many users that belongs to a company. you can reuse the user struct, from the example belongsTo, and chage the company struct, to use a HasMany object. 


```go 
type Company struct {
	ActiveRecord[*Company] `json:"-" swaggerignore:"true" validate:"-"`
	Name                   string `json:"name" validate:"required"`
}
// DB & columns func

func (c *Company) User() HasMany[*User] {
	return NewHasMany(new(User).DB(c.GetCtx()), c)
}
```
How to use it
```go 
company := new(db.Company).DB(context.Background()).WithUser().First()
	for _, user := range company.User().Items() {
		fmt.Println(user.ToJson()) // print the user struct in a json format
	}
```


## Belongs to many
association creates a direct many-to-many relationship with another model, with no intervening model. This association indicates that each instance of the declaring model refers to zero or more instances of another model.

You can implement many-to-many like this: 
```go 
func (c *Company) Users() BelongsToMany[*User] {
	return NewBelongsToMany(new(User).DB(c.GetCtx()), c)
}

func (c *Company) WithUsers() *Company {
	return c.With("Users")
}
```

And use it like this: 

```go 
company := new(db.Company).DB(context.Background()).WithUsers().First()
	for _, user := range company.Users().Items() {
		fmt.Println(user.Name)
	}
```

The DB is expected to have the the following table: "company_user", with the columns: 
* company_id
* user_id


### Pivots columns 
It is possible to include extra column to the pivot table, ex. active column. It can be included like the following: s

```go 
func (c *Company) Users() BelongsToMany[*User] {
	return NewBelongsToMany(new(User).DB(c.GetCtx()), c).PivotCols("active")
}
```

```go 
	company := new(db.Company).DB(context.Background()).WithUsers().First()
	for _, user := range company.Users().Items() {
		fmt.Println(user.Name)
		fmt.Println(user.Pivots["active"]) // print the active columns 
	}
```




## BelongsToMorph

Why is this usefull? 
How do you in a nice way parse functionality to other types? Or how do avoid of using Single Table Inheritance (STI)? 
STI is a table, which hold multiple information about different models in a single database. An. If you have a Case System, and want to see what action that have been one that system? you could include a STI, but that Table will have many rows and have many information, maybe even empty columns. Therefore this solution which is strongly inspired from Rails Delegates Relation implemented. 

### The idea: 
The idea is simple. You have an entry table, that has the the information: id, Entryable_id, entryable_type.., Created_at, Updated_at... Other optional columns. entryable type define the name of the foreign model, like user or company. and the entryable_id is the foreign key in that table. Now, you can retrive information from from both User and company and order in when they where created, which was impossible previously if they live in their own tables.   

### Example of implementation


```go 
/*
//
// BelongsToMorph relation: a model can belong to more models
//This works by having the model to have two column, a type an id column for defining the model of of owner and foreign id.
*/

//- Example of used:
// Its recommendated to define an interface that include Ire
// The interface to be implemented by models that can be entryable

	type IEntry interface {
		IRepository // Required interface to creating the interface.
		
        //
		// Abstract function
		// need to implemnt delegatable models
		Title() string

        // 
        // Not abtract 
        // Implemented in the entryable 
        // Can be override in some children structs
		PrintHi() string
	}

	//The struct that can be Implemented into the struct in models to make them delegatable entries
	type EntryAble struct {
	}

	func (e *EntryAble) PrintHi() string {
		return "Hi from EntryAble"
	}

//* ----- The DB MODEL ----------------------------------------------- //

	type Entry struct {
		ActiveRecord[*Entry] //int, created_at, updated_at
		Name                 string
		EntryAble_type       string
		EntryAble_id         int64
		EntryAble            IEntry // Value holder
	}

    // name of the type in the table
	const delegate = "entryable"
	
    // Models that implement this type  
    var (
		entryAbles = []IEntry{
			new(Company).DB(context.Background()),
			new(User).DB(context.Background()).With("company"),
		}
	)

    // Basis setup for a Entry model
	func (e Entry) DB() *Entry {
		entry := &Entry{}
		entry.Table = "entries"
		entry.Columns = Columns{
			"name":             &entry.Name,
			delegate + "_id":   &entry.EntryAble_id,
			delegate + "_type": &entry.EntryAble_type,
		}
		entry.ParentDB = CreateORM(entry)
		return entry
	}

	// * ------------------Relations --------------------------------//
	func (e *Entry) Entryable() BelongsToMorph[IEntry] {
		return NewBelongsToMorph(entryAbles, e)
	}

	//* ----------- Scopes ----------------------------------------//
	func (e *Entry) Company() IDB[*Entry] {
		return e.WhereMorph(delegate, new(Company).DB().GetName())
	}
	func (e *Entry) User() IDB[*Entry] {
		return e.WhereMorph(delegate, new(User).DB().GetName())
	}
	

    // 
    // The Company struct
    type Company struct {
		ActiveRecord[*Company]
		Name  string

		// DelegateAbles
		EntryAble // include the EntryAble struct to make company delegatable
	}

	func (c Company) DB() *Company {

		company := &Company{}
		company.Table = "companies"
		company.Columns = Columns{
			// Column		  "values"
			"name": &company.Name,
		}
		company.ParentDB = CreateORM(company)
		return company
	}

	// -------------- EntryAble functions --------------------//

	//* Implement the abstract method
	func (company *Company) Title() string {
		return company.Name + ", Company Title is called"
	}
	//* Optional: override the PritHi function
	func (company *Company) PrintHi() string {
		return company.EntryAble.PrintHi() + " with added logic to it from Company"
	}

    // The user struct Will be needed to Implement the Title function too, but Printhi is optional and can be overridden.


	//* -------------- Usage ----------------------------//
	var entries = models.Entry{}.DB().With("Entryable").OrderByDesc("id").Company().Get()
	for _, entry := range entries {
		fmt.Println(entry.EntryAble.Title())
		fmt.Println(entry.EntryAble.PrintHi())
	}
```
Note, when we are calling get, We are also calling the Func Company(), which is kind of a scope, so 
we know, we will only get entries of type: Company, but that optional. 
It can be little tricky to understand, but When you need this pattern, it a life saving, and you gonna love it.

### DB Schema: 
Just to help with the DB schema the entryable schema, even it should be clear to see:
OBS: This is schema for SQLite DB
The model, will automatic add able to the end, and assume that

```sql
CREATE TABLE IF NOT EXISTS entries (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT,
    entryable_type TEXT NOT NULL, 
    entryable_id INTEGER NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);


CREATE TRIGGER IF NOT EXISTS update_entries_updated_at
AFTER UPDATE ON entries
FOR EACH ROW
BEGIN
    UPDATE entries
    SET updated_at = CURRENT_TIMESTAMP
    WHERE id = OLD.id;
END; 
```