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
such as `{id}`, matches any one segment, and the `Call` has it in `params`. `Routes.respond` gives
the first matching route's response. When nothing matches, it gives:

- 404 if no route has that path.
- 405, with `Allow` listing the methods, if routes have that path but not that method.
- 500 with the reason, if the route fails.

A route can do any effect: a `List<Route<@Db>>` serves routes that use `@Db`.

`Server.serve(port, routes)` serves them until the program ends, one connection at a time, and
answers anything that isn't HTTP with 400. `Server.handleNext(listener, routes)` answers a single
connection.

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
uk.m4xy.http.server    Route, Call, Routes and Server
uk.m4xy.http.json      JsonBodies: requests and responses of values, as JSON, through fk-serialization
```

Each package uses only those above it. `Wire` is pure, apart from reading from a `Reader`, so a test
can parse a message from a string. Each message is the only one on its connection: it says
`Connection: close`, and gives its length with `Content-Length`. What's received may also be
chunked, or run until the connection ends.

`Network` opens connections, receives and sends on them, and closes them, the same whether they're
plain or TLS. `Sockets` handles it with the machine's sockets, and OpenSSL's TLS over them.
`Connections.reading` reads what a connection receives a line or some bytes at a time, keeping what
it hasn't read yet for the next read. `Http` declares `@Network`, as its default handler uses it. So
anything that does `@Http` does `@Network` too, even under `Server.answering`, which uses no
network.

Not yet supported:

- **A server over https.** `Server` serves plain HTTP only, as it has no certificate to serve with.
- **Concurrency.** A connection carries one request, and a server answers one connection at a time.

## Building

`kiln build` builds and tests it; `kiln publishLocal` publishes it to `~/.kiln/repository`. Its
tests run a client against a server over real sockets: the server runs in a forked copy of the
test program. Those in `tls.fk` need the internet: they fetch `https://example.com`, and check that
badssl.com's expired, wrong-host and self-signed certificates are refused.
