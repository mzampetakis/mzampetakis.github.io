---
title: Log Requests in Go
description: Log requests in Golang using an HTTP middleware.
image: /assets/posts/Log-Requests-in-Go/railway.jpg
show_image_post: false
date: 2020-08-17 23:40:00 +0300
categories: []
tags: [golang,coding]
---

It's a common sense that logging your HTTP request in your application is essential. The reasons for doing so are numerous. Error reporting, tracing, monitoring, performance tuning and so on.

One of the easiest way to do this in Go is by using an HTTP middleware. Bellow we will show a simple but elegant way of doing it!

## What is a middleware

When we build a web server it is common to use some logic among multiple endpoints to perform some tasks. Some of these tasks might be to authenticate the user, to add some headers in the response or just log something!

> An HTTP middleware in Go allows to abstract and share functionality among multiple requests.

So, an HTTP middleware allows us to organize this functionality in a higher level. More specifically, between the router and the handlers.

## A Simple middleware

A minimum middleware set up in Go requires setting up the handler and a handle function with the desired middleware.

```go
package main

import (
	"log"
	"net/http"
)

func myMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("Entering middleware")
		next.ServeHTTP(w, r)
		log.Println("Leaving middlewareOne")
	})
}

func handleFunc(w http.ResponseWriter, r *http.Request) {
	log.Println("Executing handleFunc")
	w.Write([]byte("OK"))
}

func main() {
	mux := http.NewServeMux()
	handler := http.HandlerFunc(handleFunc)
	mux.Handle("/", myMiddleware(handler))
	log.Println("Listening on :8000...")
	log.Fatal(http.ListenAndServe(":8000", mux))
}
```

The magic happens in line `24` where we enforce the `myMiddleware` to be used by our handler for our `handler`. Running the above will give us the following result:

```console
2020/08/12 20:08:35 Listening on 8000...
2020/08/12 20:08:40 Entering middleware
2020/08/12 20:08:40 Executing handleFunc
2020/08/12 20:08:40 Leaving middleware
```

This makes sense how can we exploit middlewares to achieve the desired result every single time. Middlewares can be cascaded but this requires to take extra care for the order that will be used. 

### Log Requests

Back to the request logging we require some request details to be logged in our application. Thankfully all this information is within `*http.Request`. So for our example we could log the request using this middleware:  

```go
func myMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("Entering middleware")
		log.Printf("%s : Method: %s | URL: %s%s | Proto: %s",
			time.Now().Format(time.RFC3339), r.Method, r.Host, r.URL, r.Proto)
		next.ServeHTTP(w, r)
		log.Println("Leaving middleware")
	})
}
```

At line `4` the middleware logs a Datetime alongside some basic information of the request. The output we get is as follows:

```console
2020/08/12 20:44:15 Listening on 8000...
2020/08/12 20:44:20 Entering middleware
2020/08/12 20:44:20 2020-08-17T22:14:40+03:00 : Method: GET | URL: localhost:8000/ | Proto: HTTP/1.1
2020/08/12 20:44:20 Executing handleFunc
2020/08/12 20:44:20 Leaving middleware
```

> There is no need to log explicitly date and time as `log` already prefixes our logging string.

### Tracing support

Moving a step forward for our request logger we would like to add some tracing information for each request. Apart from creating and logging a unique request id we should also save this generated uuid within each request. There is no better place to save this other that the HTTP context.

> Context package makes it easy to pass request-scoped values, cancelation signals, and deadlines across API boundaries to all the goroutines involved in handling a request.

More about the Go's context package can be found [here](https://golang.org/pkg/context/).

The following middleware generates a uuid, logs at the request logging nad passes it inside the request's context.

```go
func myMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("Entering myMiddleware")
		requestID := uuid.New()
		log.Printf("%s : %s: Method: %s | URL: %s%s | Proto: %s",
			time.Now().Format(time.RFC3339), requestID, r.Method, r.Host, r.URL, r.Proto)
		ctx := context.WithValue(r.Context(), "request-id", requestID)
		next.ServeHTTP(w, r.WithContext(ctx))
		log.Println("Leaving myMiddleware")
	})
}
```

Now, the request log prints the generated request uuid and has the following format:

```console
2020/08/12 22:17:36 Listening on 8000...
2020/08/12 22:17:39 Entering middleware
2020/08/12 22:17:39 2020-08-17T22:17:39+03:00 : a81ea2c1-55c3-4a31-b331-b5acf2cfac6d: Method: GET | URL: localhost:8000/ | Proto: HTTP/1.1
2020/08/12 22:17:39 Executing handleFunc
2020/08/12 22:17:39 Leaving middleware
```

From this point we have access to this request-id within our handler function allowing us to log with this unique id.

> Keep in mind that now our middleware has a double role. The first is to log each request and the second on is to populate the context with a `request-id`. This might not be a preferred solution, however, it was easier to demonstrate it here like this.

At this point all seem good. That's not completely correct. As seen [heat this issue](https://go-review.googlesource.com/c/go/+/30084):

>Using a context.Context Value key of type string is a terrible idea and walks into the minefield of a global namespace. We should've outlawed it from day 1.
>All context value keys should be made from user-defined types.

Which means we have to use an alternative/custom type for our `request-id` context key.

The following could solve the issue we just describef by introducing the `RequestIDKey` type and use it when setting the value into context.

```go
// RequestIDKey type is the type used as key to store request-id into context
type RequestIDKey string

func myMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("Entering myMiddleware")
		requestID := uuid.New()
		log.Printf("%s : %s: Method: %s | URL: %s%s | Proto: %s",
			time.Now().Format(time.RFC3339), requestID, r.Method, r.Host, r.URL, r.Proto)
		requestIDKey := RequestIDKey("request-id")
		ctx := context.WithValue(r.Context(), requestIDKey, requestID)
		next.ServeHTTP(w, r.WithContext(ctx))
		log.Println("Leaving myMiddleware")
	})
}
```
From now on, the same type `RequestIDKey` should be used in order to retrieve the `request-id` whenever needed!

## Log response

We can now easily log the response of each handler. Just by extending our middleware after the `log.Println("Leaving middleware")` line and using the `http.ResponseWriter` object. We could use the same middleware or another one depending on our case.

## Conclusion

As mentioned earlier, middlwares have many use cases. Logging is just one of them. Middlwares are used in most API services in Go for authentication and authorization, for compression, for recovery for content type and many more cases. 

Extra caution should be taken when using multiple middlewares as well as when group of routes are used!
