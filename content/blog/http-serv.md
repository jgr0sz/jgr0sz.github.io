---
title: "http-serv"
date: 2026-04-14
draft: false
tags: ["projects", "go", "backend"]
---

Recently, my focus has been on trying to learn backend development (and Go), since the breadth of my experience is using Cloudflare/Github to host static sites like this one. As a way to try and understand things hands-on, I was inspired by someone to build an HTTP server from scratch as a starting point. 

Despite this project really making me feel at times like I was putting the cart before the horse, I learned a great deal about HTTP, sockets, route handling, and how protocols like TCP function. 

# How It Works (as of writing)
In order for a request not to 404, server-side routes are defined through the function `addRoute()`. It takes in a specified HTTP method, path, and a handler function like `http.handleFunc()`, where an array `routes` receives the data after it has been packed into a `Route` struct composed of all 3 arguments.

```go
type HandlerFunc func(*Request) *Response

type Route struct {
	method  string
	path    string
	handler HandlerFunc
}
```

`listenTCP()` is called after routes are registered, where I used `net.Conn` for all the networking jazz since it seemed like the "from scratch" option; `listenTCP()` takes in a singular address argument and blocks until a connection is found on it, from which `net.Conn` is concurrently handled by the first of many server handler functions.

Following setting a 5-second deadline for the connection, the contents of it are passed to `parseRequest()`, which in summary splits up, categorizes, and packs data into a `Request` struct consisting of the following:

```go
type Request struct {
	method  string
	path    string
	proto   string
	headers map[string]string
	body    string
}
```

This makes it convenient for when we compare `Request`'s method/path fields with those of the `Route` objects in `routes`.

This comparison/creation of a response all occurs in `invokeRoute()`, which starts by searching for a matching path *and* method. If only the former is found, a 405 response is thrown. If both are found, the handler function is called, allowing for desired server-side behavior to occur.

As you probably noticed from earlier, `HandlerFunc` requires a `Response` type to be returned, which essentially complements `Request`:

```go
type Response struct {
	status     int
	statusText string
	headers    map[string]string
	body       string
}
```

It was at this point I realized that having to make a user registering routes type out something like `foobar := &Response{status, statusText, tons of headers, large body}` would be tedious, so I decided to abstract it using a package-level `NewResponse()` that optionally works without any provided headers, defaulting to `Content-Type: text/plain`. This functionality goes hand-in-hand with the internal portions where 404 and 405 are thrown, since both can use this function and not have to provide headers. Once this function runs, a `Response` is finally returned from `invokeRoute()`, from which it is sent to a `writeResponse()` that uses the `net.Conn` object and the response to construct and send back a formatted response.

![Source code sample of server-side routes](/media/httpserv1.png)
(Sample of some defined server-side routes in VSC, along with what a `Request` struct would look like)

![Sample of a functioning route](/media/httpserv2.png)
(Plaintext response from a `GET /login` route, following a `POST` response from the previous screenshot's code)

# Going forward
It works somewhat decently? I was able to get multiple webpages and various HTTP methods to be handled concurrently via `curl` and `net.Dial()` tests, albeit it lacks functionality in the form of middleware and query parameters, two aspects I'm just beginning to wrap my head around. A salient improvement I will likely add ASAP is the `routes` array and the way it's traversed for a `Route` object with a matching method and path. My DSA foundation is rather weak, but my intuition tells me that linearly searching a dynamic array like so is acceptable in smaller applications, but is an inferior solution. 

Now that I more or less understand HTTP requests/responses on a lower level, my next project will likely be to make something more fleshed-out that takes advantage of Go's robust libraries. Feel free to send me any suggestions, and thank you for reading :-)