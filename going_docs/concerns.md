


Concern 
Definere din nye concern: 
```go 
package Concerns

import (
	. "myapp/internal/super/db/types"
)

type ICustomConcern[A IConcern[A]] interface {
	IConcern[A]
	ScopeA() A
}

// user implementations
type ConcernsImplementation[A ICustomConcern[A]] struct { // replace ICustomConcern with IConcern if you want to use the base concern
	*Concern[A]
	CustomField string `json:"-" swaggerignore:"true" validate:"-"`
}

func (q *ConcernsImplementation[A]) Init(w A) {
	q.Concern = &Concern[A]{IConcern: w}
}

func (s *ConcernsImplementation[A]) Scope() A {
	return s.Where("id", 2)
}


// ----------------------- example on usage of a scope --------------// 
type Company struct {
	ActiveRecord[*Company] `json:"-" swaggerignore:"true"` 
	Name                   string
	concerns.ConcernsImplementation[*User] `json:"-" swaggerignore:"true" validate:"-"`
}

func (c Company) DB() *Company {
	company := &Company{}
	company.Table = "companies"
	company.Columns = company.columns()
	user.ConcernsImplementation.Init(user)
	company.ParentDB = CreateORM(company)
	return company
}
// Force to implement by the custom interface
func (user *User) ScopeA() *User {
	return user
}

//-------------------------- calling -------------------------------------
var company = new(models.Company).DB()
	company.ConcernTest.Scope() 
	fmt.Println(company.First()) // print company instance 

```