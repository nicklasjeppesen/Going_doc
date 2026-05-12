


Concern 
Definere din nye concern: 
```go 
package Delegates

import (
	. "myapp/internal/super/db/types"
)

// concret implementations
type ConcernsImplementation struct {
	Concern
}

func (s *ConcernsImplementation) Scope() { s.Where("id", 2) }
s

// ----------------------- example on usage of a scope --------------// 
type Company struct {
	ActiveRecord[*Company] `json:"-" swaggerignore:"true"` 
	Name                   string
	ConcernTest            ConcernsImplementation `json:"-" swaggerignore:"true"`
}

func (c Company) DB() *Company {
	company := &Company{}
	company.Table = "companies"
	company.Columns = company.columns()
	company.ParentDB = CreateORM(company)
	company.ConcernTest = ConcernsImplementation{Concern{company}}
	return company
}

//-------------------------- calling -------------------------------------
var company = new(models.Company).DB()
	company.ConcernTest.Scope() // OBS: Scope Does not return anything, so dot notation i cut of, but can be recreate caling company again.
	fmt.Println(company.First()) // print company instance 

```