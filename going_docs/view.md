# Views

- [Introduction](#introduction)
- [Creating a view](#creating-a-view)
- [Views structure](#views-structure)
  - [Partial views](#partial-views)
- [View names convention](#view-names-convention)
- [Components and sub views](#components-and-sub-views)
  - [Components](#components)
- [Other base views](#other-base-views)
- [Built-in view functions](#built-in-view-functions)
  - [render](#render)
  - [component](#component)
  - [haserrors](#haserrors)
  - [hasflash](#hasflash)
  - [old](#old)
  - [csrf_token](#csrf_token)
  - [Add your own](#add-your-own)

## Introduction

This section describes how to use the view functionality in Going. Going uses Go's built-in templates with custom functions and design choices for productive development, giving you options similar to other popular web frameworks.

Going supports two view systems:
- **Server-side templates** - Go's built-in `html/template` with custom functions
- **Inertia.js** - SPA-like behavior with React/Vue frontends


## Creating a view

All views should be placed in `resources/views/`. Views are returned from controllers using the `View()` function:

```go
package controller

type HomeController struct {
	ControllerBase
}

func (c *HomeController) Home() Result {
	return View("index")
}
```

This function return a the template file ```resources/views/index.template```

you can add paramter to the view by parsing the Params as a second parameter: 


```go 
package controller

type HomeController struct {
	ControllerBase
}

func (c *HomeController) Home() Result {
	return View("index", Params{"Title": "Going App", "Message": "Welcome to Going"})
}
```

This renders the template file `resources/views/index.tmpl`.


## Views structure

It is assumed all views start from the base view at `resources/views/base.template`.

When you return a view name in your controller, Going replaces `ContentView` with the view name in the custom `render` function, and the view receives all view data:

```go
// resources/views/base.template

{{ define "base" }}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="csrf_token" content="{{ .csrf_token }}">
    <title>{{ .Title }}</title>
</head>
<body>
    <main>
        {{ render .ContentView . }}
    </main>
</body>
</html>
{{ end }}
```

### Partial views

Define sections in your view and place them at specific locations in the base view:

```go
// resources/views/base.template
{{ define "base" }}
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="csrf_token" content="{{ .csrf_token }}">
    <title>{{ .Title }}</title>
    <link rel="stylesheet" href="/assets/css/style.css">
    <script src="/assets/js/socket.js"></script> 
</head>
<body>
    <main>
        {{ render .ContentView . }}        
    </main>

    <footer>
        {{ render (print .ContentView "_footer") . }}
    </footer>   
</body>
</html>
{{ end }}
```

Then define the partial in your view:

```go 
{{ define "index" }}
// resources/views/index.template
<body>
    <div class="card">
        <h1>{{ .Title}} 👋</h1>
        <p>{{ .Message}}</p>
        <div class="nav-links">
            <a href="/login">Log ind</a> | 
            <a href="/register">Opret konto</a>
        </div>
    </div>
</body>

{{ end }}

{{ define "index_footer" }}
    <h5>Index footer {{ .Title }}</h5>
{{ end }}
```

**Note**: Name your partials using the convention `ContentView_section` to avoid collisions.


## View names convention

In the controller you define a path to your view by dot notation. This login will return the view in folder resources/views/auth/login.template
 
```go 
type LoginController struct {
	ControllerBase
}

// get
func (login *LoginController) LoginGet() Result {
	return View(
		"auth.login",
		Params{"Title": "Login", "Message": "Please Login!"})

}
```
In your login view, you should define your view, acording to the path to the file, also in dot notation, like this. 

```go
// path: resources/views/auth/login.template
{{ define "auth.login" }}
<div class="card">
    <h1>{{ .Title }}</h1>
... rest of the view
{{end}}
// Exaple of the footer section. 
{{ define "auth.login_footer" }}
    <h5>footer {{ .Title }}</h5>
{{ end }}
```

## Components and sub views

Going provides functions for writing reusable view components, similar to React components.

### Components

Create a template file as a component:

```go
// resources/views/shared/input.template
{{ define "shared.input" }}
    <label for="id_{{.Name}}">{{.Title}}</label>
    <input
        type="{{or (index . "Type") "text"}}"
        id="id_{{.Name}}"
        name="{{.Name}}"
        value="{{ if index . "oldValue" }} {{.oldValue}} {{ end }}"
        required
    >
{{ end }}
```

Use the component in your views:

```go
<form enctype="multipart/form-data" action="/login" method="POST">
    {{component "shared.input" "Title" "Email" "Name" "email" "oldValue" ( old "email" .) }}
    {{component "shared.input" "Title" "Password" "Name" "password" "Type" "password" }}
    {{csrf_token .}}
    <button type="submit">Log in</button>
</form>
```

The `component` function takes the component name followed by key-value pairs as parameters.


## Other base views

You don't need to use the default base view. You can switch between or omit base views:

```go
// Default view with base template
var View = template.TemplateView{BaseView: "base", CustomViewFunctions: viewProvider.GetCustomViewFunction}.View

// SubView - no base template, just the content
var SubView = template.TemplateView{BaseView: "", CustomViewFunctions: viewProvider.GetCustomViewFunction}.View

// Admin view with admin base template
var AdminView = template.TemplateView{BaseView: "admin", CustomViewFunctions: viewProvider.GetCustomViewFunction}.View
```

## Build in view functions

### component 
Is another build in function, that was described earlier

### render

Renders a named template with data. Primarily used in the base view to render the main content and partials:

```go
{{ render .ContentView . }}
{{ render (print .ContentView "_footer") . }}
```

### component

Renders a reusable template component with key-value parameters:

```go
{{component "shared.input" "Title" "Email" "Name" "email"}}
```

### haserrors

A boolean indicating whether there are any validation errors:

```go
{{ if .haserrors }}
    <div class="mb-4 text-red-600">
        <p>An error occurred</p>
    </div>
{{ end }}
```

Get specific error messages:

```go
{{ if index .errors "email" }}
    <div class="text-red-500 mt-1 text-sm">
        {{ .errors.email }}
    </div>
{{ end }}
```

### hasflash

True if flash data was passed via `response.With()`:

```go
{{ if .hasflash }}
    {{ range $key, $value := .flash }}
        <h1>{{ $value }}</h1>
    {{ end }}
{{ end }}
```

### old

Returns the value that was posted to your controller, useful for repopulating forms:

```go
<input
    type="email"
    name="email"
    value="{{ old "email" . }}"
/>
```

### csrf_token

Renders a hidden CSRF token input field:

```go
<input type="hidden" name="csrf_token" value="{{ .csrf_token }}" />
```

### Add your own

Add custom view functions in `myapp/internal/app/providers/view`:

```go
var CustomViewFunctions = map[string]any{
    "uppercase": func(s string) string {
        return strings.ToUpper(s),
    },
}
```
