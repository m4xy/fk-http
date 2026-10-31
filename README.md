# fk-http

HTTP/1.1 for [fckt](https://github.com/m4xy/fckt): a client and a server over one model of requests
and responses.

```kotlin
import uk.m4xy.http.*
import uk.m4xy.http.Messages.*
import uk.m4xy.http.Statuses.*
import uk.m4xy.http.client.Client
import uk.m4xy.http.server.*

shop = [
  Route(Get, "/items/{id}", call -> text(ok, "item ${call.params.get("id").getOrElse("?")}")),
  Route(Post, "/items", call -> text(created, "made ${call.request.text}")),
]

main() = Server.serve(8080, shop)                                  // a server

main() = println(Client.get("http://localhost:8080/items/3").text)   // a client
```

A [kiln](https://github.com/m4xy/kiln) project uses it as `library("uk.m4xy:fk-http:0.1.0")`. Its
TLS is OpenSSL's, through the OCaml package `ssl`, so a machine that builds with it needs
[opam](https://opam.ocaml.org) and that package:

```
brew install opam pkgconf && opam init --bare --no-setup && opam switch create default ocaml-system
PKG_CONFIG_PATH=$(brew --prefix openssl@3)/lib/pkgconfig opam install ssl ocamlfind
```

kiln links it into any program that uses fk-http, without the program saying so.

## The model

`uk.m4xy.http` is what both sides share:

- `Request(method, url, headers, body)`
- `Response(status, headers, body)`
- `Method`: an enumeration, `Get`, `Post` and the rest; `MethodText` writes and reads it as `GET`.
- `Status(code)`: `Statuses` names them (`ok`, `notFound`), with each one's `reason` and whether
  it `isSuccess()`.
- `Url(scheme, host, port, path, query)`: `UrlText` reads `http://shop.example:8080/items?page=2`
  and writes it back.
- `Header(name, value)`: `Headers` finds one by name in any case.
- `Messages`: `text(status, text)` makes a response, and `.text` and `.header(name)` read a
  request or response.

Bodies are `Bytes`, so anything can be sent; `.text` reads one as UTF-8.

## The client

`Client.get(url)`, `Client.post(url, body, contentType)` and `Client.send(request)` send through the
`Http` effect. They fail only where there's no response: a response saying it didn't work, such as a
404, is still a response, so look at its status.

`main` handles `Http` with `HttpClient`, which opens a connection for each request and closes it
once the response is read. An https URL's connection goes through TLS. The host's certificate must
be signed by one the machine's OpenSSL trusts, and be for that host. Otherwise there's no response,
only the reason: `expired.badssl.com's certificate isn't trusted: certificate has expired`. A test can answer `Http` itself, so nothing is sent:

```kotlin
Http(send = request -> Ok(text(ok, "stubbed"))) { Client.get("http://shop.example/items") }
```

## The server

A `Route(method, path, respond)` answers requests with that method and path. A segment in braces,
such as `{id}`, matches any one segment, and the `Call` has it in `params`. Routes are found in a
tree of their paths' segments, `Routers.of(routes)`, not tried in turn, so the order they're
written in doesn't matter: a fixed segment comes before a parameter, so `/links` goes to the route
for `/links` and not that for `/{slug}`, and where a fixed segment leads nowhere further, a
parameter's routes are tried. Two routes for one method whose paths match the same requests, as
`GET /links/{slug}` and `GET /links/{id}`, are refused before anything is served. When nothing
matches, `Routes.respond` gives:

- 404 if no route has that path.
- 405, with `Allow` listing the methods, if routes have that path but not that method.
- 500 with the reason, if the route fails.

A route can do any effect: a `List<Route<@Db>>` serves routes that use `@Db`.
`Routes.mounted("/api/users", routes)` puts routes under a prefix, so each part of a service can
say its own and the service mount them together.
`Routes.around(routes) { respond -> ... }` answers each route inside a handler, as a database
transaction for each request; `Routes.aroundCalls(routes) { call, respond -> ... }` is given the
call too, as what logs each request's path and how it was answered is.

### Endpoints

An endpoint is a route that says what it reads from a call, as an `Input`: a path's parameter,
`param("id")`; a query's, `query("page")`; a header, `header("Accept")`; the body as text,
`bodyText`, or as JSON, `body(serializer)`. A block of inputs reads several, and finds everything
wrong with a call, not just the first, as fk-sqlite's `Columns` do:

```kotlin
lookup: Input<Lookup> = Inputs {
  id by param("id").as("a number") { it.toIntOrNone() }
  page by query("page")
  Lookup(id, page)
}

routes: List<Route> = [
  get("/items/{id}", lookup) { l -> text(ok, "item ${l.id}") },
  post("/items", body(ItemSerializer)) { item -> json(created, ItemSerializer, saved(item)) },
]
```

A call whose input can't be read is answered 400, saying everything that's wrong, before the
endpoint's function runs. An input that reads a parameter its path hasn't got, as
`get("/items/{id}", param("slug"))`, is wrong where it's written, so it fails as the endpoint is
made, when the program starts, not in some call later.

### OpenAPI

Endpoints say what they read, so `OpenApi.document(title, version, routes)` writes an OpenAPI 3
document of them: each path's operations, their path, query and header parameters, and the body each
reads, its schema the shape its serializer describes. What a route answers it says itself:
`.answeringJson(created, ItemSerializer)` for JSON, `.answering(noContent)` for none, and
`.summarised("...")` says what it's for. Each `@Serializable` type is a schema of its own, named in
full, referred to wherever it's used. `OpenApi.documenting("/openapi.json", title, version, routes)`
is a route that serves it.

`Server.serve(port, routes)` serves them until the program ends, and answers anything that isn't
HTTP with 400. Each connection is a task of its own, as `fckt.concurrent` has them: while one waits
for a slow client, the others are answered. A connection that receives or sends nothing for
`timeout`, 30 seconds unless it says otherwise, `serve(port, routes, timeout = Duration(5000))`, is
answered 400, saying so, and closed. Tasks take turns, one running at a time, so routes never see
each other half way through anything; `executor`, `InTurn` unless it says otherwise, says in what
order. `Server.handleNext(listener, routes)` answers a single
connection, waiting for it as `Network` does.

`Server.answering(routes) { ... }` answers the requests its block sends with the routes, without a
network. It tests a client and its server together:

```kotlin
@Test @Fail @Network itemsAreFound() = Server.answering(shop) {
  assertEquals("item 3", Client.get("http://shop.example/items/3").text)
}
```

## JSON

A body can be a value, as JSON, written and read by its
[fk-serialization](https://github.com/m4xy/fk-serialization) serializer, which its processor writes
for each type marked `@Serializable`. `JsonBodies` has both sides:

```kotlin
import uk.m4xy.http.json.JsonBodies.*

shop: List<Route> = [
  Route(Get, "/items/{id}", call -> json(ok, ItemSerializer, found(call.params))),
  Route(Post, "/items", call -> receiving(call, ItemSerializer) { item -> json(created, ItemSerializer, saved(item)) }),
]

item = Client.get("https://shop.example/items/3").json(ItemSerializer)
made = postJson("https://shop.example/items", ItemSerializer, Item("desk", 99))
```

`json` answers with a value, as `application/json`, and `Response.json` reads one, failing, saying
why, where it isn't one. `receiving` gives a route the value a request sent, and where it sent none,
answers 400, saying why, as it's the client that's wrong, not the route. `postJson` and `putJson`
send one.

## How it works

```
uk.m4xy.http           The model: Request, Response, Method, Status, Url, Header
uk.m4xy.http.wire      Wire: requests and responses as bytes, and read back through @Reader
uk.m4xy.http.net       Network: connections, plain or TLS, which main handles with Sockets; Connections reads them
uk.m4xy.http.client    Http: sending requests, which main handles with HttpClient; Client
uk.m4xy.http.server    Route, Call, Routes, Routers, Endpoints, Inputs and Server
uk.m4xy.http.openapi   OpenApi, documents of routes
uk.m4xy.http.json      JsonBodies: requests and responses of values, as JSON, through fk-serialization
```

Each package uses only those above it. `Wire` is pure, apart from reading from a `Reader`, so a test
can parse a message from a string. Each message is the only one on its connection: it says
`Connection: close`, and gives its length with `Content-Length`. What's received may also be
chunked, or run until the connection ends.

`Network` opens connections, receives and sends on them, and closes them, the same whether they're
plain or TLS. `Sockets` handles it with the machine's sockets, and OpenSSL's TLS over them. `PolledSockets` handles it the same way, but where a task would wait to accept, receive or send, it parks until `fckt.io.Polls`' `select` says it can go on. A handler's code runs where it's installed, so it goes around each task's own code, not around the server.
`Connections.reading` reads what a connection receives a line or some bytes at a time, keeping what
it hasn't read yet for the next read. `Http` declares `@Network`, as its default handler uses it. So
anything that does `@Http` does `@Network` too, even under `Server.answering`, which uses no
network.

Not yet supported:

- **A server over https.** `Server` serves plain HTTP only, as it has no certificate to serve with.
- **Keep-alive.** A connection carries one request.
- **Parallelism.** A server's tasks take turns on one core, which suits what mostly waits for the
  network; work that keeps a core busy keeps the others waiting while it does.
- **Waiting in a route's own requests.** A route's `Client` calls, and connecting and TLS, wait as
  the system does, holding up the other tasks meanwhile. OpenSSL can hold what it's received where
  `select` can't see it, and the `ssl` package can't say whether it does, so TLS is never waited
  for as a task.

## Building

`kiln build` builds and tests it; `kiln publishLocal` publishes it to `~/.kiln/repository`. Its
tests run a client against a server over real sockets: the server runs in a forked copy of the
test program. Those in `tls.fk` need the internet: they fetch `https://example.com`, and check that
badssl.com's expired, wrong-host and self-signed certificates are refused.
