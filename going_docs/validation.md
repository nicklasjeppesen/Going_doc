# Validation

- [Introduction](#introduction)
- [Basic validation](#basic-validation)
- [Validation in controller](#validation-in-controller)
- [Error messages](#error-messages)
- [Custom validation rules](#custom-validation-rules)
- [Custom per-struct validation](#custom-per-struct-validation)


## Introduction

This section describes how to use validation to validate input from the user. Going uses the [go-playground/validator](https://github.com/go-playground/validator) package with customizable error messages loaded from a JSON configuration file.


## Basic validation

Add `validate` struct tags to your fields. The validation is powered by `go-playground/validator`:

```go
type InputStore struct {
    Id    int64  `json:"id" validate:"number,required,min=30"`
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"required,email"`
}
```

Common validation tags:
- `required` - Field must be present
- `email` - Must be a valid email
- `min=X` / `max=X` - Minimum/maximum value or length
- `number` - Must be a number
- `oneof=a b c` - Must be one of the specified values

See the [validator documentation](https://github.com/go-playground/validator) for all available tags.


## Validation in controller

When using `RequestBodybase[T]`, call `Validate()` to validate the input:

```go
func (c *TestController) Store(r request.RequestBodybase[*InputStore]) {
    if result := r.Validate(); result.HasError {
        // Validation failed
        Response.WithErrors(result.GetErrors()).Back()
        return
    }
    // Validation passed, use r.Body
    Response.PrintJson(r.Body)
}
```

`Validate()` returns a `*Result[T]` with:
- `HasError` - Whether validation failed
- `Errors` - Map of field names to error messages


## Error messages

Going uses a `rules.json` file (at `internal/resources/rules/rules.json`) to define custom error messages. This allows you to provide human-readable error messages instead of the default validator messages.

### Default messages

Set default messages for each validation tag:

```json
{
    "default_messages": {
        "required": "The {field} field is required",
        "email": "Please enter a valid email address",
        "min": "The {field} must be at least {param} characters"
    }
}
```

### Field-specific messages

Override messages for specific struct fields:

```json
{
    "field_messages": {
        "InputStore": {
            "email": {
                "required": "Email address is required for registration",
                "email": "Please enter a valid email address"
            }
        }
    }
}
```

### Registering custom validation rules

Register custom validation rules at startup using `RegisterValidation()`:

```go
import "github.com/nicklasjeppesen/going_internal/super/validation"

func init() {
    validation.RegisterValidation("even", func(fl validator.FieldLevel) bool {
        value := fl.Field().Int()
        return value%2 == 0
    })
}
```

Then use the custom tag in your struct:

```go
type Input struct {
    Number int64 `json:"number" validate:"even"`
}
```


## Custom per-struct validation

For complex business logic rules, implement the `Validator` interface on your struct:

```go
type InputStore struct {
    Id    int64  `json:"id" validate:"number,required"`
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"required"`
}

func (i *InputStore) Validate() error {
    if i.Id <= 30 {
        return errors.New("Id value must be minimum 30")
    }
    return nil
}
```

This method is called after the struct tag validation, allowing you to add business rules like database lookups or cross-field validation.
