# ORM - Database Relationships

- [Introduction - TODO](#introduction)
- [Setup - TODO](#Setup)
- [BelongsToMorph](#BelongsToMorph)


<a name="introduction"></a>
## Introduction

<a name="Setup"></a>
## Setup



<a name="BelongsToMorph"></a>
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
			new(Company).DB(),
			new(User).DB().With("company"),
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
	var entries = models.Entry{}.DB().With("entryable").OrderByDesc("id").Company().Get()
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