![Gear](https://raw.githubusercontent.com/teambition/gear/master/gear.png)
[![Build Status](http://img.shields.io/travis/teambition/gear.svg?style=flat-square)](https://travis-ci.org/teambition/gear)
[![Coverage Status](http://img.shields.io/coveralls/teambition/gear.svg?style=flat-square)](https://coveralls.io/r/teambition/gear)
[![License](http://img.shields.io/badge/license-mit-blue.svg?style=flat-square)](https://raw.githubusercontent.com/teambition/gear/master/LICENSE)
[![GoDoc](http://img.shields.io/badge/go-documentation-blue.svg?style=flat-square)](http://godoc.org/github.com/teambition/gear)

=====
A lightweight, composable and high performance web service framework for Go.

## Demo

### Simple service
```go
package main

import (
	"fmt"
	"github.com/teambition/gear"
	"github.com/teambition/gear/logging"
)

func main() {
	app := gear.New()

	// Add logging middleware
	app.UseHandler(logging.Default())

	// Add router middleware
	router := gear.NewRouter()
	router.Use(func(ctx *gear.Context) error {
		// do some thing.
		fmt.Println("Router middleware...", ctx.Path)
		return nil
	})
	router.Get("/", func(ctx *gear.Context) error {
		return ctx.HTML(200, "<h1>Hello, Gear!</h1>")
	})
	app.UseHandler(router)
	app.Error(app.Listen(":3000"))
}
```

### HTTP2 with Push

https://github.com/teambition/gear/tree/master/example/http2

```go
package main

import (
	"net/http"

	"github.com/teambition/gear"
	"github.com/teambition/gear/logging"
	"github.com/teambition/gear/middleware/favicon"
)

// go run app.go
func main() {

	const htmlBody = `
<!DOCTYPE html>
<html>
  <head>
    <link href="/hello.css" rel="stylesheet" type="text/css">
  </head>
  <body>
    <h1>Hello, Gear!</h1>
  </body>
</html>`

	const pushBody = `
h1 {
  color: red;
}
`

	app := gear.New()

	app.UseHandler(logging.Default())
	app.Use(favicon.New("../../testdata/favicon.ico"))

	router := gear.NewRouter()
	router.Get("/", func(ctx *gear.Context) error {
		ctx.Res.Push("/hello.css", &http.PushOptions{Method: "GET"})
		return ctx.HTML(200, htmlBody)
	})
	router.Get("/hello.css", func(ctx *gear.Context) error {
		ctx.Type("text/css")
		return ctx.End(200, []byte(pushBody))
	})
	app.UseHandler(router)
	app.Error(app.ListenTLS(":3000", "../../testdata/server.crt", "../../testdata/server.key"))
}
```

### A CMD tool: static server

https://github.com/teambition/gear/tree/master/example/staticgo

It is a useful CMD tool that serve your local files as web server.
You can build `osx`, `linux`, `windows` version with `make build`.

```go
package main

import (
	"flag"

	"github.com/teambition/gear"
	"github.com/teambition/gear/logging"
	"github.com/teambition/gear/middleware/static"
)

var (
	address = flag.String("addr", "127.0.0.1:3000", `address to listen on.`)
	path    = flag.String("path", "./", `static files path to serve.`)
)

func main() {
	flag.Parse()
	app := gear.New()

	app.UseHandler(logging.Default())
	app.Use(static.New(static.Options{Root: *path}))

	logging.Println("staticgo v1.0.0, created by https://github.com/teambition/gear")
	logging.Printf("listen: %s, serve: %s\n", *address, *path)
	app.Error(app.Listen(*address))
}
```



## Import

```go
// package gear
import "github.com/teambition/gear"
```

## About Router
[gear.Router](https://godoc.org/github.com/teambition/gear#Router) is a tire base HTTP request handler.
Features:

1. Support regexp
2. Support multi-router
3. Support router layer middlewares
4. Support fixed path automatic redirection
5. Support trailing slash automatic redirection
6. Automatic handle `405 Method Not Allowed`
7. Automatic handle `501 Not Implemented`
8. Automatic handle `OPTIONS` method
9. Best Performance

The registered path, against which the router matches incoming requests, can contain three types of parameters:

| Syntax | Description |
|--------|------|
| `:name` | named parameter |
| `:name*` | named with catch-all parameter |
| `:name(regexp)` | named with regexp parameter |
| `::name` | not named parameter, it is literal `:name` |


Named parameters are dynamic path segments. They match anything until the next '/' or the path end:

Defined: `/api/:type/:ID`
```
/api/user/123             matched: type="user", ID="123"
/api/user                 no match
/api/user/123/comments    no match
```

Named with catch-all parameters match anything until the path end, including the directory index (the '/' before the catch-all). Since they match anything until the end, catch-all parameters must always be the final path element.

Defined: `/files/:filepath*`
```
/files                           no match
/files/LICENSE                   matched: filepath="LICENSE"
/files/templates/article.html    matched: filepath="templates/article.html"
```

Named with regexp parameters match anything using regexp until the next '/' or the path end:

Defined: `/api/:type/:ID(^\d+$)`
```
/api/user/123             matched: type="user", ID="123"
/api/user                 no match
/api/user/abc             no match
/api/user/123/comments    no match
```

The value of parameters is saved on the gear.Context. Retrieve the value of a parameter by name:
```
type := ctx.Param("type")
id   := ctx.Param("ID")
```

## About Middleware
```go
// Middleware defines a function to process as middleware.
type Middleware func(*gear.Context) error
```

`Middleware` can be used in app layer or router layer or middleware inside. It be good at composition.
We should write any module as a middleware. We should use middleware to compose all our business.

There are three build-in middlewares currently: https://godoc.org/github.com/teambition/gear/middleware

```go
// package middleware
import (
	"github.com/teambition/gear/middleware/cors"
	"github.com/teambition/gear/middleware/favicon"
	"github.com/teambition/gear/middleware/static"
)
```
1. [CORS middleware](https://godoc.org/github.com/teambition/gear/middleware/cors#New) Use to serve CORS request.
2. [Favicon middleware](https://godoc.org/github.com/teambition/gear/middleware/favicon#New) Use to serve favicon.ico.
3. [Static server middleware](https://godoc.org/github.com/teambition/gear/middleware/static#New) Use to serve static files.

All this middlewares can be use in app layer, router layer or middleware layer.

## About Hook
`Hook` can be used to some teardowm job dynamically. For example, Logger middleware use `ctx.OnEnd` to write logs to underlayer. Hooks are executed in LIFO order, just like go `defer`. `Hook` can only be add in middleware. You can't add another hook in a hook.

```go
ctx.After(hook func())
```
Add one or more "after hook" to current request process. They will run after middleware process(means context process `ended`), and before `Response.WriteHeader`. If some middleware return `error`, the middleware process will stop, all "after hooks" will be clear and not run.

```go
ctx.OnEnd(hook func())
```
Add one or more "end hook" to current request process. They will run after `Response.WriteHeader` called. The middleware error will not stop "end hook" process.

Here is example using "end hook" in Logger middleware.
```go
func (l *Logger) Serve(ctx *gear.Context) error {
	// Add a "end hook" to flush logs.
	ctx.OnEnd(func() {
		log := l.FromCtx(ctx)
		log["Status"] = ctx.Status()
		log["Type"] = ctx.Res.Get(gear.HeaderContentType)
		log["Length"] = ctx.Res.Get(gear.HeaderContentLength)

		// Don't block current process.
		go l.consume(log, ctx)
	})
	return nil
}
```

## Documentation

https://godoc.org/github.com/teambition/gear

## Benchmark

### Gear with "net/http": 50030
```sh
> wrk 'http://localhost:3333/?foo[bar]=baz' -d 10 -c 100 -t 4

Running 10s test @ http://localhost:3333/?foo[bar]=baz
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.22ms    3.91ms 155.60ms   97.49%
    Req/Sec    12.58k     1.26k   18.76k    84.25%
  501031 requests in 10.01s, 65.46MB read
Requests/sec:  50030.72
Transfer/sec:      6.54MB
```

### Iris with "fasthttp": 70310
```sh
> wrk 'http://localhost:3333/?foo[bar]=baz' -d 10 -c 100 -t 4

Running 10s test @ http://localhost:3333/?foo[bar]=baz
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.37ms  648.31us  15.60ms   89.48%
    Req/Sec    17.75k     2.32k   39.65k    84.83%
  710317 requests in 10.10s, 102.29MB read
Requests/sec:  70310.19
Transfer/sec:     10.13MB
```

### Gin with "net/http": 50195
```sh
> wrk 'http://localhost:3333/?foo[bar]=baz' -d 10 -c 100 -t 4

Running 10s test @ http://localhost:3333/?foo[bar]=baz
  4 threads and 100 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.07ms    1.50ms  30.44ms   90.04%
    Req/Sec    12.62k     1.12k   15.42k    77.50%
  502815 requests in 10.02s, 65.69MB read
Requests/sec:  50195.68
Transfer/sec:      6.56MB
```

## License
Gear is licensed under the [MIT](https://github.com/teambition/gear/blob/master/LICENSE) license.
Copyright &copy; 2016 [Teambition](https://www.teambition.com).
