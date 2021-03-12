---
title: "Mastering HTML templates in Go - The fundamentals"
date: 2021-03-11T23:00:00+01:00
tags: ["go", "html"]
summary: "The main reason why we all like the Go programming language so much is because of its simplicity. And how else could the html/template parser be, than just simple."
---

- [Introduction](#introduction)
- [The beauty of its simplicity](#the-beauty-of-its-simplicity)
- [Parsing templates](#parsing-templates)
    - [Parsing from a string](#parsing-from-a-string)
    - [Parsing from a file](#parsing-from-a-file)
    - [Parsing from an io.FS (e.g. embed package)](#parsing-from-an-iofs-eg-embed-package)
- [How values are rendered](#how-values-are-rendered)
    - [Accessing structs, slices and maps](#accessing-structs-slices-and-maps)
        - [Structs](#structs)
        - [Slices](#slices)
        - [Maps](#maps)
- [Control structures: If, else if, else](#control-structures-if-else-if-else)
    - [Truthy and falsy](#truthy-and-falsy)
    - [The hardest part about conditions](#the-hardest-part-about-conditions)
- [Control structures: Loops](#control-structures-loops)
- [Wrapping up](#wrapping-up)

## Introduction

For my last post about [How I build web frontends in Go](/web-frontends-in-go) I've received an overwhelmingly amount of feedback on [reddit](https://www.reddit.com/r/golang/comments/lzymk9/how_i_build_web_frontends_in_go/). But one thing I've noticed is, that there are quite a few people who felt that the `html/template` package was either very hard to deal with or they did not even know about most of its features.

The [official documentation](https://golang.org/pkg/html/template/) of this package is a bit misleading because most of it is actually [here](https://golang.org/pkg/text/template/), so I can partially understand the reasoning. But with this post, I am trying to clean up with this and transfer all of my knowledge over to you.

## The beauty of its simplicity

The main reason why we all like the Go programming language so much is because of its simplicity. And how else could the `html/template` parser be, than just simple. If we take a look into the [source code of the lexer](https://github.com/golang/go/blob/master/src/text/template/parse/lex.go#L76-L87), there are just 10 _keywords_ (and some additional symbols):

```
. (dot), block, define, else, end, if, range, nil, template, with
```

## Parsing templates

The `html/template` package offers three different ways to parse a template, which have all their use cases. But if you pick one, you will probably stick to it throughout your whole project. The three ways are:

1. Parsing from a string
2. Parsing from a file
3. Parsing from an io.FS (e.g. `embed` package since Go 1.16)

### Parsing from a string

Parsing a raw html string is the most basic way of creating a new template and is very easy to reason about. Note here, that we first have to create a new template instance with `template.New("")`. The first argument is the reference name of the template and can be used to render different templates from just one instance. However if we have just one parse template per instance, we can just leave the name empty.

```go
// Parsing a single template
import "html/template"

var html = `Hello {{.}}!`

tmpl, _ := template.New("").Parse(html)
tmpl.Execute(os.Stdout, "World")        // Hello World!
```

```go
// Parsing multiple templates into a single instance
import "html/template"

var hello = `Hello {{.}}!`
var hey = `Hey {{.}}!`

tmpl, _ := template.New("hello").Parse(hello)
tmpl.New("hey").Parse(hey)

tmpl.ExecuteTemplate(os.Stdout, "hello", "World") // Hello World!
tmpl.ExecuteTemplate(os.Stdout, "hey", "World")   // Hey World!
```

### Parsing from a file

Parsing from a file used to be the most common way to initialize template as you could have them separate from your Go code. Using a file instead of a string does not require you to call the `.New("")` function, as it takes the filename of the template as its name. Also note here that the path to the template file must match with the working directory you're executing your Go program from.

```html
<!-- hello.html -->
Hello {{.}}!
```
```go
import "html/template"

tmpl, _ := template.ParseFiles("hello.html")
tmpl.Execute(os.Stdout, "World") // Hello World!
```

### Parsing from an io.FS (e.g. embed package)

With Go 1.16 and the new `io.FS` interface a new way to parse templates was introduced. As the `embed` package allows you to embed a whole directory into you binary and `embed.FS` implements the `io.FS` package, we can use it to ship our templates with out Go binary.
This is the go-to way I am personally working with templates in my code.

```html
<!-- hello.html -->
Hello {{.}}!
```
```go
import "html/template"
import "embed"

//go:embed *.html
var files embed.FS

tmpl, _ := template.ParseFS(files, "hello.html")
tmpl.Execute(os.Stdout, "World") // Hello World!
```

## How values are rendered

In the previous examples, you've already seen some snippets on how to render values inside a template, but let me go deeper on this an explain it in more detail.

When a value like a string is passed into the `tmpl.Execute()` method call, the template engine will map this value to literal `.` (dot), as seen within the curly braces `{{.}}`. It can then be used to just print it out or in an `if`, `range`, `with`, ... instruction.

Printing out the dot (or basically anything) will follow the same formatting conventions as if you were to print out a value with `fmt.Print()`. Here are some examples, so you get a better feeling for it:

```go
// Template: {{.}}

tmpl.Execute(os.Stdout, 1)                          // 1
tmpl.Execute(os.Stdout, 3.14)                       // 3.14
tmpl.Execute(os.Stdout, true)                       // true
tmpl.Execute(os.Stdout, "hello")                    // hello
tmpl.Execute(os.Stdout, []int{1, 2, 3})             // [1 2 3]
tmpl.Execute(os.Stdout, map[string]int{"foo": 1})   // map[foo:1]
tmpl.Execute(os.Stdout, struct{X int; Y int}{1, 2}) // {1 2}
```

### Automatic escaping of user inputs

A crucial part of a HTML templating engine is its capability to deal with unsafe user input. If you for example had a comment section on your web page where users an enter anything they like, the templating engine should be capable of escaping that inputs automatically and therefore making it safe to be rendered. If this were not the case and a user would inject a `<script>alert("hacked")</script>` into his message, this would be called a XSS attack.

The `html/template` package is safe in this case as it prevents these attacks. Let's take a look of what it renders when passing it said message:

```go
// Template: <div>{{.}}</div>
tmpl.Execute(os.Stdout, `<script>alert("hacked")</script>`)
```
```html
<!-- Output -->
<div>&lt;script&gt;alert(&#34;hacked&#34;)&lt;/script&gt;</div>
```

A secret super power of the `html/template` package is, that it also knows in which context a value is rendered. If we were to render the same message inside a `<script>` tag instead, the message will be escaped differently. This is also true for CSS within a `<style>` tag.

```go
// Template: <script>{{.}}</script>
tmpl.Execute(os.Stdout, `<script>alert("hacked")</script>`)
```
```html
<!-- Output -->
<script>"\u003cscript\u003ealert(\"hacked\")\u003c/script\u003e"</script>
```

### Accessing structs, slices and maps

Rendering simple values is easy but not really useful. In most cases you will be passing in more complex data structures such as slices, maps or custom structs, especially when you want to render an entire HTML5 document. In that case you probably have some general page information, user data and slices of content. For that it's really useful to know how you can access all these nested values.

#### Structs

As we know, the dot holds the current value or a reference to it. We can also use it to access nested properties of that value. Here is an example for a struct:

```html
Hello {{.Firstname}} {{.Lastname}}
```
```go
type Person struct {
    Firstname string
    Lastname  string
}

tmpl.Execute(os.Stdout, Person{
    Firstname: "Philipp",
    Lastname:  "Tanlak",
})
```

#### Slices

In most cases when you're working with slices, you will range over them and not access their indexes directly. But for completeness I will show it anyway, before we go over to control structures. Accessing a slice index can be achieved by using the inbuilt `index` template function and it has the following syntax: `{{index items itemIndex}}`

```html
An {{index . 1}} a day keeps the doctor away!
```
```go
tmpl.Execute(os.Stdout, []string{"Orange", "Apple", "Banana"})
```

#### Maps

Similar to slices, maps can be accessed in the very same way using the `index` template function. But in addition to that, it's also possible to access maps like structs. So here are the two ways to access a map:

```html
{{index . "GOOG"}}
<!-- or -->
{{.GOOG}}
```
```go
tmpl.Execute(os.Stdout, map[string]float32{
    "GOOG": 2120.24,
    "AAPL": 121.99,
})
```

## Control structures: If, else if, else

When using html templates for rendering web pages, you will unlikely get around conditional rendering. It can be useful to display certain content based on if a given value is true or false, e.g. if a user is logged in or if a there are search results for a given search term. To deal with this, the `html/template` syntax offers the `if` instruction, which checks if a value is `true` or `false`.

```html
{{if .IsAuthenticated}}
    Hello {{.Username}}!
{{else}}
    Please log in.
{{end}}
```
```go
type Session struct {
    IsAuthenticated bool
    Username        string
}

tmpl.Execute(os.Stdout, Session{
    IsAuthenticated: true,
    Username:        "philipta",
})
```

We can see that ifs are basically just like ifs in Go and there is also an `else if` to check for other conditions.
```
{{else if .OtherVariable}}
```

One thing to note with this `if` instruction is, that it's simply closed with an `end` instruction and not something like `endif`. This is true for all control structures like `range`, `with`, `template`, `block`, etc.

### Truthy and falsy

When thinking of if conditions in the Go, we're not allowed to have truthy or falsy values in our code. Everything must evaluate to a strict boolean value. This is not the case for Go's html templates. Here we're allowed to check for the truthyness of a value. So we can for example check if a string is empty or filled and put that into the if condition like so:

```html
{{if .}}
    String contains the value: {{.}}
{{else}}
    String is empty
{{end}}
```
```go
tmpl.Execute(os.Stdout, "i'm not empty")
```

Here is a list of the most common values you might want to check for truthyness and what it evaluates to:

```go
true                   => true
false                  => false

"foo"                  => true
""                     => false

1                      => true
0                      => false

struct{}               => true
nil                    => false

[]int{5}               => true
[]int{}                => false

map[string]int{"A": 1} => true
map[string]int{}       => false
```

### The hardest part about conditions

Now we've come to the point where most people struggle with and I personally think that it's the hardest part to wrap your head around when working with Go's templating engine. Actually it's pretty simple once you got the hang of it. But if you want to learn anything about templates in Go, this is a must.

As I've mentioned in the very beginning of this post, the template syntax is very minimal but actually very powerful. Because of that, when dealing with if statements, we are not allowed to use operators such as `&&`, `||`, `==`, `!=`, `>`, etc. Therefore we can not have complex expressions like:
```html
<!-- does not work -->
{{if .IsAdmin || .HasEditorAccess }}
```

But we've also seen that the `html/template` package comes with built in functions like `index` from the slices and maps example. In case you've wondered, there are more than just that. Here is a list of the most commonly used inbuilt functions and what they are doing:

```go
and  =>  x && y
or   =>  x || y
not  =>  !x
eq   =>  x == y
ne   =>  x != y
lt   =>  x <  y
gt   =>  x >  y
le   =>  x <= y
ge   =>  x >= y
```

With these functions we can replicate any condition we like, just as if we had direct access to the operators I've mentioned above. So the example from above would be written like:

```html
<!-- does not work -->
{{if .IsAdmin || .HasEditorAccess }}

<!-- does work -->
{{if or .IsAdmin .HasEditorAccess }}
```
If we were to translate this into Go code and these template functions _were_ a thing in Go, it would look like this:
```go
if isAdmin || hasEditorAccess {}

if or(isAdmin, hasEditorAccess) {}
```

> So when dealing with templates, think of these conditions as functions or just as if you were writing in the Lisp programming language.

In every template instructions we are also allowed to use parentheses to encapsulate function calls and compose multiple instructions. For example:

```html
{{if and .IsAuthenticated (or .IsAdmin .HasEditorAccess)}}
```
```go
// equivalent go code
if isAuthenticated && (isAdmin || hasEditorAccess) {}
```

In addition to `and` and `or` there is also `eq`, `ne`, `not` and so on... With that we can build even more complex conditions like this for example:
```html
{{if eq .PageType "post"}} <!-- .PageType == "post" -->
{{if ne "Apple" "Orange"}} <!-- "Apple" != "Orange" -->
{{if not (gt 100 20)}}     <!-- !(100 > 20)         -->
```

If you want to take a look at all the functions in detail, here's the link to the official documentation: [Functions](https://golang.org/pkg/text/template/#hdr-Functions)

So once you wrapped you head around this concept, you will be way more productive with the `html/template` package.

## Control structures: Loops

Now that you know how complex conditions work, let's quickly take a look at another easy part again, which is: Loops. Loops are very useful when you for example want to render a list of blog posts on a page or just show a list of items. For that, the template engine provides a `range` instruction.

The `range` instruction introduces us to a new concept, which I don't really know the name of 😅. But what it does is changing the value inside the dot to the value of the current iterated over item. At first the dot contains the entire slice as its value, but once it enters the range loop, the dot contains the single slice item. Let me show you:

```html
<!-- . (dot) is: []string{"Apple", "Banana", "Orange"} -->
{{range .}}
    <!-- . (dot) is: "Apple" (in 1st iteration)-->
    Value is: {{.}}
{{end}}
```
```go
tmpl.Execute(os.Stdout, []string{"Apple", "Banana", "Orange"})
```

The above example just gives us access to the item values of the slice, but what is also possible is, that we get the index to it as well by using an assignment you know from Go's `for i, item := range items` loop. Here is an example how we can grab the index and the value using template variables:

```html
{{range $index, $value := .}}
    At index {{$index}}, there is: {{$value}}
{{end}}
```
```go
tmpl.Execute(os.Stdout, []string{"Apple", "Banana", "Orange"})
```

In Go we can not only range over slices, but also over maps. This is also possible with the range instruction in a template. This basically looks identical to the previous example:

```html
{{range $ticker, $price := .}}
    Stock price of {{$ticker}} is: {{$price}}
{{end}}
```
```go
tmpl.Execute(os.Stdout, map[string]float32{
    "GOOG": 2120.24,
    "AAPL": 121.99,
})
```

## Wrapping up

In this post we've talked about the fundamental knowledge you need to know about templating in Go. There is way more to talk about, but as a start this should be enough to digest.

I can highly recommend you trying out these things and play around with them, as in the next post, we will cover more template instructions like `with`, `template`, `block` and also custom functions.

Cheers 👋
