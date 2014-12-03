# Stack

Stack makes it simple to create context-aware middleware chains for Go web applications.

It provides two main features:

1. A convenient interface for chaining middleware handlers and application handlers together, inspired by [Alice](https://github.com/justinas/alice).
2. The ability to pass request-scoped data (or *context*) between middleware and application handlers.

[Skip to the example &rsaquo;](#example)

## Installation

```bash
go get github.com/alexedwards/stack
```

## Usage

### Creating a Chain

New middleware chains are created using `stack.New`:

```go
stack.New(middlewareOne, middlewareTwo, middlewareThree)
```

You should pass middleware as parameters in the same order that you want them executed (reading left to right).

The `stack.New` function accepts middleware using the following pattern:

```go
func middlewareOne(ctx stack.Context, h http.Handler) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
     // do something middleware-ish, accessing ctx
     h.ServeHTTP(w, r)
  })
}
```

Middleware with the signature `func(http.Handler) http.Handler` can also be added to a chain, using the `stack.Middleware` adapter. This makes it easy to register third-party middleware with Stack.

For example, if you want to use Goji's [httpauth](https://github.com/goji/httpauth) middleware you would do the following:

```go
authenticate := stack.Middleware(httpauth.SimpleBasicAuth("user", "pass"))

stack.New(authenticate, middlewareOne, middlewareTwo)
```

### Adding Handlers

Application handlers are added to a chain using the `Then` method. When this method has been called the chain becomes a `http.Handler` and can be registered with Go's `http.DefaultServeMux`.

```go
http.Handle("/", stack.New(middlewareOne, middlewareTwo).Then(appHandler))
```

The `Then` method accepts handlers using the following pattern:

```go
func appHandler(ctx stack.Context) http.Handler {
  return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
     // do something applicaton-ish, accessing ctx
  })
}
```

Any object which satisfies the `http.Handler` interface can also be used in `Then` thanks to the `stack.Handler` adapter. For example:

```go
fs :=  http.FileServer(http.Dir("./static/"))

http.Handle("/", stack.New(middlewareOne, middlewareTwo).Then(stack.Handler(fs)))
```

Similarly the `stack.HandlerFunc` adapter is provided so a function with the signature `func(http.ResponseWriter, *http.Request)` can also be used. For example a function like:

```go
func foo(w http.ResponseWriter, r *http.Request) {
  w.Write([]byte("foo"))
}
```

Can be used in `Then`:

```go
http.Handle("/", stack.New(middlewareOne, middlewareTwo).Then(stack.HandlerFunc(foo)))
```

### Example

This example chains together the third-party `httpAuth` middleware package and a custom `tokenMiddleware` function. This middleware sets a `token` value in the context which can be later accessed by the `tokenHandler` function.

```go
package main

import (
    "fmt"
    "github.com/alexedwards/stack"
    "github.com/goji/httpauth"
    "net/http"
)

func main() {
    authenticate := stack.Middleware(httpauth.SimpleBasicAuth("user", "pass"))

    http.Handle("/", stack.New(authenticate, tokenMiddleware).Then(tokenHandler))
    http.ListenAndServe(":3000", nil)
}

func tokenMiddleware(ctx stack.Context, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Add a value to Context with the key 'token'
        ctx["token"] = "c9e452805dee5044ba520198628abcaa"
        next.ServeHTTP(w, r)
    })
}

func tokenHandler(ctx stack.Context) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Retrieve the token from Context and print it
        fmt.Fprintf(w, "Token is: %s", ctx["token"])
    })
}
```

Requesting the resource should give you a response like:

```shell
$ curl -i user:pass@localhost:3000/
HTTP/1.1 200 OK
Content-Length: 41
Content-Type: text/plain; charset=utf-8

Token is: c9e452805dee5044ba520198628abcaa
```

### Getting and Setting Context

You should be aware that `stack.Context` is implemented as a `map[string]interface{}`, scoped to the goroutine executing the current HTTP request.

To keep your code type-safe at compile time it's a good idea to use getters and setters when accessing Context.

The above example is better written as:

```go
...

func tokenMiddleware(ctx stack.Context, next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        SetToken(ctx, "c9e452805dee5044ba520198628abcab")
        next.ServeHTTP(w, r)
    })
}

func tokenHandler(ctx stack.Context) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Token is: %s", Token(ctx))
    })
}

func SetToken(ctx stack.Context, token string) {
    ctx["token"] = token
}

func Token(ctx stack.Context) string {
    return ctx["token"].(string)
}
```

As a side note: If you're planning to pass Context to a secondary goroutine for processing you'll need to make sure that it is safe for concurrent use, probably by implementing a [mutex lock](http://www.alexedwards.net/blog/understanding-mutexes) around potentially racy code.

### Reusing Stacks

Like Alice, you can easily reuse middleware chains in Stack:

```go
stdStack := stack.New(middlewareOne, middlewareTwo)

extStack := stdStack.Append(middlewareThree, middlewareFour)

http.Handle("/foo", stdStack.Then(fooHandler))

http.Handle("/bar", extStack.Append(middlewareFive).Then(barHandler))
```
