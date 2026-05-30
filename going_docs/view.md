# Views 

- [Introduction](#introduction)

- [Creating a View](#creating-a-view)

- [Views structure](#views-structure)


 


<a name="introduction"></a>
## Introduction

this section describe how to use the view functionality in Going. Going is using Go's build in template, for simplicity, but have added custom functions, and other design choices, for productive and giving you some of the same options as in other popular web framework. 

<a name="creating-a-view"></a>
## Creating a view

All views should be place in the folder ressources/views. Views is a function avaiable in your controller, defined in controller_base, which allow you to return a view as shown below. 

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


## Views structure 
It assume all view, is starting the the base view place at ```resources/views/base.template```, 

The when you return a view name in your controller class, Going will replace he ContentView with the name of the view, in the custom render function, and the actually view will get all view data. NOTE: the data parsing from the controller is also avaible here. 

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
Sometimes you might want to define data in your view, and place it, at a given place in the baseView, like a specific footer, or header. This can easy be done with Going custom render func. here we have defined a footer section in the base template file

```go
{{ define "base" }}
<!DOCTYPE html>
<html lang="da">
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

It can den be used like this: 
```

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

OBS: you should name your partial in the naming convention: 'ContentView_section', to avoid colissions. 


## View names convention
View shall be name after their path and filename. 
In our example, index is at the top level in views, but you might want to group views in folders. This can be done as follow: 

In the controller you define a path to your view by dot notation. This login will return the view in folder resoueces/views/auth/login.template
 
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
Going build on functions help to write you view onces, and apply components based deveopment, as you know from ex. from react. 

### Components 
You can simple create a new template file, han use it as a component, ex. 

```go 
// resources/views/shared/input.template
{{ define "shared.input" }}

<label for="id_{{.Name}}">{{.Title}}</label>
        <input type="{{or .Type "text"}}" id="id_{{.Name}}" name="{{.Name}}" value="{{.oldValue}}" required>
{{end}}
```

Simple example of a form section with label and input, This can be used as follow: 

```go 
    <form enctype="multipart/form-data" action="/login" method="POST">
        {{component "shared.input" "Title" "Email" "Name" "email" "oldValue" ( old "email" .) }}
        {{component "shared.input" "Title" "Password" "Name" "password" "Type" "password" }}
        {{csrf_token .}}
        <button type="submit">Log ind</button>
    </form>
```

Going build in component function first input is the name of the define component, and the a list of parameters in the form of key, value,... 