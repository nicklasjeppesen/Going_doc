# Validation

- [Introduction](#introduction)
- [Create validation](#create-validation)
- [Validation in controller](#Validation-in-controller)


## Introduction 
This section describe how to use use validation to validate input from the user. 
Going is using the Google playground basis validator, for simply validate input. see github for that: https://github.com/go-playground/validator

This mean you can create a struct, and add the validate struct to it, like this: 

```go
type InputStore struct {
	Id    int64  `json:"id" validate:"number,required,min=30"`
	Name  string `json:"name" validate:"required"`
	Email string `json:"e-mail" validate:"required"`
}
```
For validate input in your controller, you can check the result like the following: 

```go 
func (c *TestcontrollerController) Store(r RequestBody[*InputStore]) Result {

	if result := r.Validate(); result.Error {
		return Response.PrintJson(result.ErrorMessage.Error())
	}
	return Response.PrintJson(r.Body)
}
```
Going will parse the input to the struct, and validate the input for you.


## Custom validation 
Some time, you need more custom validation for a given model. You add a the func Validate() error to your model, which can hold custom validation rules. 

Se the example below: 

```go 
type InputStore struct {
	Id    int64  `json:"id" validate:"number,required"`
	Name  string `json:"name" validate:"required"`
	Email string `json:"e-mail" validate:"required"`
}

// Simple example
func (i *InputStore) Validate() error {
	if i.Id <= 30 {
		return errors.New("Id value shall be minimum 30")
	}
	return nil
}
```
This is a simple example, for demostrating specific rules, could be usefor for look up values in DB etc. 