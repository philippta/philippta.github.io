---
title: "How I build web frontends in Go"
date: 2021-03-07T17:00:00+01:00
draft: false
tags: ["go", "html"]
summary: "Go is a great language for building all kinds of backend services like APIs. But what about web frontends? Let's have a look..."
---

- [Introduction](#introduction)
- [Why would I choose Go?](#why-would-i-choose-go)
- [Rendering HTML with Go](#rendering-html-with-go)
- [How I structure my templates?](#how-i-structure-my-templates)
  - [The WordPress way](#the-wordpress-way-dont-do-this)
  - [The Django, Rails, Laravel way](#the-django-rails-laravel-way-do-this)
- [Implementing the template renderers](#implementing-the-template-renderers)
    - [The package structure](#the-package-structure)
    - [Bundling up the templates](#bundling-up-the-templates)
    - [Parsing the templates](#parsing-the-templates)
    - [Template helper structs and functions](#template-helper-structs-and-functions)
- [Using the new template functions](#using-the-new-template-functions)
- [Wrapping up](#wrapping-up)

## Introduction

Go is a great language for building all kinds of backend services like APIs or microservices and many people use it for that. But what about web frontends, specifically dynamically rendered web applications? Let's have a look...


## Why would I choose Go?

With every side project I am creating, there is almost certainly a need for a web frontend. Over the past years I've tried many different solutions which fit seemingly perfectly for this job. Especially when it comes to client side rendered pages built with React, Vue, Gatsby, Next.js, etc. But every time I try to build web applications with these frameworks it feels overly complicated, so I end up throwing them away and falling back to basic HTML pages rendered on the server side with Go. But I don't want to talk about the difficulties I have with these, as this is for another post.

## Rendering HTML with Go

In case you haven't heard of it before, Go comes with a built in html templating engine, which you can find in the `html/template` package. The syntax is not that intuitive at first but actually very simple and gets the job done perfectly. Here is a quick example of how to parse and render an html template:

```go
import "html/template"

tmpl, _ := template.New("").Parse("<h1>{{.}}</h1>") // from string
tmpl, _ := template.ParseFiles("demo.html")         // from file

tmpl.Execute(w, "Hello world")
// Output: <h1>Hello world</h1>
```

I don't want to go into much detail how the templating syntax works right now as I'd much rather focus on patterns I've developed for myself which deal with more complex template structures. A reference of every possible template instruction can be found in the [text/template documentation](https://golang.org/pkg/text/template/).

## How I structure my templates?

When it comes to Go, there are two types of possibilities how to deal with template fragments (e.g. header, content, footer).

1. The WordPress way
2. The Django, Rails, Laravel way

### The WordPress way (Don't do this)

The WordPress way of structuring templates is to have a `header.html` and a `footer.html` file, and several others for the content in between. This is not a great idea to do, as it introduces splitting HTML tags in half. One example for this would be:
```html
<!-- header.html -->
{{define "header"}}
<html>
    <body>
        <div class="navbar">...</div>
        <div class="content">
{{end}}
```
```html
<!-- profile.html -->
{{template "header" .}}

<div class="profile">
    Your username: {{.User.Name}}
</div>

{{template "footer" .}}
```
```html
<!-- footer.html -->
{{define "footer"}}
        </div>
    </body>
</html>
{{end}}
```

The corresponding Go code would look something like this:

```go
tmpl, _ := template.ParseFiles("header.html", "footer.html", "profile.html")
tmpl.Execute(w, User{Name: "philippta"})
```

### The Django, Rails, Laravel way (Do this)

A better way to structure templates is to have parent and child templates, as you would define them in Django, Rails or Laravel. This approach results in cleaner and better maintainable templates as it does not cut HTML tags in half and you don't have to worry about forgetting to close a HTML tag in another file. One example for this would be:

```html
<!-- layout.html -->
<html>
    <body>
        <div class="navbar">...</div>
        <div class="content">
            {{block "content" .}}
                <!-- Fallback, if "content" is not defined elsewhere -->
            {{end}}
        </div>
    </body>
</html>
```
```html
<!-- profile.html -->
{{define "content"}}
<div class="profile">
    Your username: {{.User.Name}}
</div>
{{end}}
```

The corresponding Go code would look something like this:

```go
tmpl, _ := template.ParseFiles("layout.html", "profile.html")
tmpl.Execute(w, User{Name: "philippta"})
```

With this approach you're free to add as many child templates as you want and can't forget to include the header and footer templates.
You also have the option to define more than just one "content" block, which is useful if you want to include e.g. additional stylesheets, scripts or just change the title. Here another quick example:
```html
<!-- layout.html -->
<html>
    <head>
        <title>{{block "title"}}Default page title{{end}}</title>

        <script src="app.js"></script>
        {{block "additional_scripts"}}{{end}}
    </head>
    <body>
        <div class="navbar">...</div>
        <div class="content">
            {{block "content" .}}
                <!-- Fallback, if "content" is not defined elsewhere -->
            {{end}}
        </div>
    </body>
</html>
```

## Implementing the template renderers

As we've now covered the basics on how to structure your templates, let me talk about how to render them. In the previous examples I've chosen to compile and execute the templates just inline, but that does not scale very well and can lead to mistakes really quickly. Especially when dealing with data or custom template functions. So to circumvent that I create specific rendering functions for each template. There is not much science behind that, it's solely to make my life easier.

### The package structure

But before I elaborate on that, I want to quickly give you a picture of what a hypothetical Go directory structure could look like.
There is a `main.go` file which has the server code with all the handlers, which in the end render the templates to the browser. Then there is a `html` directory/package which contains all HTML template files and also a `html.go` file containing all template related functions. I know that Go's standard library already defines an `html` package and there could be naming conflicts, but as the only place these functions would be used is our `html` package itself, it's pretty safe. This approach is best described by Ben Johnson's [Standard package layout](https://www.gobeyond.dev/standard-package-layout/)
```
html/
    layout.html
    dashboard.html
    profile/
        show.html
        edit.html
    html.go         <-- here are the rendering functions
main.go             <-- here could be the http handlers
```

So let's take a look into the `html.go` file and see what is defined there.

### Bundling up the templates

I am making use of is the `embed` package introduced in Go 1.16 to bundle up the HTML templates into the resulting Go binary.
The `//go:embed *` instruction will gather all files in the current `html` directory and make the contents available in the `files` variable for later parsing. It has an additional benefit, that you don't have to worry about running the Go binary from the correct location, as it would be necessary when using `.ParseFiles()`.

```go
import "embed"

//go:embed *
var files embed.FS
```

### Parsing the templates

As the templates are now bundled up and generally available in the `files` variable, I can start parsing them. For that I always create a small helper function, which is not exported and only used in this `html` package. Also notice that I'm using the new method `.ParseFS()` instead of `.ParseFiles()` now.

```go
import "html/template"

func parse(file string) *template.Template {
	return template.Must(
        template.New("layout.html").ParseFS(files, "layout.html", file))
}
```

This is optional, but if I need additional custom template functions, I also add another global variable for `template.FuncMap` and add that into the parsing code like this:

```go
var funcs := template.FuncMap{
    "uppercase": func(v string) string {
        return strings.ToUpper(v)
    },
}
```

```diff
  func parse(file string) *template.Template {
  	  return template.Must(
-          template.New("layout.html").ParseFS(files, "layout.html", file))
+          template.New("layout.html").Funcs(funcs).ParseFS(files, "layout.html", file))
  }
```

With that helper function in place, I then have the opportunity to parse all files very easily. Like `files`, I also assign them to package global variables. I know that global variables are not considered a good approach, but as they are very basic and unexported, the risk is very minimal.

```go
var (
    dashboard   = parse("dashboard.html")
    profileShow = parse("profile/show.html")
    profileEdit = parse("profile/edit.html")
)
```

### Template helper structs and functions

The next part is to create developer friendly rendering functions. These are fairly easy as they just take an `io.Writer`, the desired template parameters and execute the template. I am using well defined structs for the data I am passing into the execute function instead of interfaces as it gives me better type safety for the template variables. With that I just have to make sure once, that all variables used in a template are filled correctly. Whenever my data changes, I get feedback on that right at compile time instead of seeing something is missing when looking at it in the browser.

```go
type DashboardParams struct {
    User       User
    Statistics []Statistics
}

func Dashboard(w io.Writer, p DashboardParams) error {
    return dashboard.Execute(w, p)
}

type ProfileShowParams struct {
    User        User
    ProfileInfo Profile
}

func ProfileShow(w io.Writer, p ProfileShowParams) error {
    return dashboard.Execute(w, p)
}

// and so on ...
```

## Using the new template functions

With all that scaffold in place, it's now super easy to use render these HTML templates in a save and developer friendly manner. Just setup your http endpoint handler, fill in the template params struct and execute the template to the `http.ResponseWriter`.

```go
import "github.com/philippta/myproject/html"

http.HandleFunc("/dashboard", func(w http.ResponseWriter, r *http.Request) {
    user := getUser()
    stats := getStatistics()

    params := html.DashboardParams{
        User:       user,
        Statistics: stats,
    }
    html.Dashboard(w, params)
})
```

## Wrapping up

This basically wraps up the way I am writing frontend applications in Go. It's not really hard to implement and makes my life way easier. By using the `html/template` package instead of other frontend frameworks, you focus more on the layout and not introduce unnecessary complexity.

If you're interested in seeing all of this coming together, follow this link to my GitHub repo:
https://github.com/philippta/web-frontend-demo

Cheers 👋
