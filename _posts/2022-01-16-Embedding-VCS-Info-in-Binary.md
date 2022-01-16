---
title: Embedding VCS Info in Go binary
description: Embed Version Control System information info in the Go binary.
image:
  src: /assets/posts/Embedding-VCS-Info-in-Binary/embed.jpg
  show_in_post: false
date: 2022-01-16 12:15:00 +0200
categories: []
tags: [golang,coding]
---

Binaries we produce from our applications are something like black boxes! When we create and distribute Go binaries we 
cannon distinguish somehow the version of the binary or any other metadata. Unless we deliberately add appropriate code
and update it on every single build we make we cannot offer this capability. It makes sense that when we want to be able 
to identify different versions of our application from each binary directly.

Embedding various metadata within our binary can significantly improve debugging , monitoring, logging and more.
Version Control System (VCS) information, build time and build environment are some metadata that when included within the 
binary will make our lives easier!

Go offers two ways for adding such metadata within a binary. The first one presented is using special flags that are 
used at compile time to pass within the binary the appropriate metadata. The second way is available from Go version 
1.18 onwards and makes use of the Debug package.

## Using Build Link Flags

Go's rich tooling provides a mechanism that reads the Go archive or object for a package main, along with its dependencies, 
and combines them into an executable binary. This is available through the [cmd/link](https://pkg.go.dev/cmd/link) package

An option named `-ldflags` is available through the build command which allows the insertion of dynamic information to 
the binary at build time without any modification or our source code.

In order to understand how `-ldflags` option works we can examine the following code snippet:
```go
package main

import "fmt"

var Version = "0.9"
var BuildTime = "2022-01-01 00:00"

func main(){
	fmt.Println("Hello build info!")
	fmt.Println("Version:\t" +Version)
	fmt.Println("Built @:\t" +BuildTime)
}
```

We can clearly see that our app will compile and when run it will produce the following output:
```terminal
$ go build -o main .
$ ./main
  Hello build info!
  Version:	v0.9
  Built @:	2022-01-01 00:00
```

As mentioned earlier, we are able to use `-ldflags` at build time in order to pass in flags to the `go tool link` which 
runs as a part of the `go build` process. It's syntax is 
```console
$ go build -ldflags="-flag" -o main .
```

Using this syntax we can pass multiple different link flags to our binary at build time. In our example we are going to 
use the `-X` flag which allows us to write information to a variable. The syntax of the ldflags using th `-X` option is:
```console
$ go build -ldflags="-X 'package_name.variable_name=value'" -o main .
```
Where `variable_name` is the name of the variable we want to replace and `package_name` the name of the package that the 
variable resides. `value` is the new value of the variable we want to assign. So, back to our application, we would like 
to add the current version and the real build time. This can be achieved by using the following build command:
```console
$ go build -ldflags="-X 'main.Version=v1.0.0' -X 'main.BuildTime=$(date)'" -o main .
```
Running our app after the successful build we we get this output:
```console
$ ./main
  Hello build info!
  Version:	v1.0.0
  Built @:	Fri Jan 14 18:41:49 EET 2022
```

We can see that our new desired values of the specific variables have been linked to our binary successfully.
This way we can replace any specific value within any package of our application at build time which can provide vital
assistance when investigating issues.

> It is more convenient to automate the build process while using -ldfags so that the binary includes all the required 
> information. A make file might be a possible solution!

A worth mentioning tool here is the (go tool nm)[https://pkg.go.dev/cmd/nm]. 
This tool lists the symbols defined or used by an object file, archive, or executable. For our executable we can invoke 
it like this:
```console
$ go tool nm ./main | grep main
  1124020 D main..inittask
  1133f70 D main.BuildTime
  10bf660 R main.BuildTime.str
  1133f80 D main.Version
  10beeb4 R main.Version.str
  108a0c0 T main.main
  1031b00 T runtime.main
...
```
Here we can see that the variables we used are within this list. This way we can easily find any available variable and build again the application replacing them.

So in a last example where we want to provide our binary with the last commit's hash we can use the following source:
```go
package main

import "fmt"

var Version = "0.9"
var BuildTime = "2022-01-01 00:00"
var CommitHash = ""

func main(){
	fmt.Println("Hello build info!")
	fmt.Println("Version:\t" +Version)
	fmt.Println("Built @:\t" +BuildTime)
	fmt.Println("Commit:\t" +CommitHash)
}
```

and then build and run our application:
```consolse
$ go build -ldflags="-X 'main.Version=v1.0.0' -X 'main.BuildTime=$(date)' -X 'main.CommitHash=$(git rev-parse HEAD)'" -o main .
$ ./main
  Hello build info!
  Version:	v1.0.0
  Built @:	Sat Jan 14 18:31:14 EET 2022
  Commit:	52519871fd3199ec66becab0dc8b14fbdbdbfdce
```

## Using the Debug package

From Go 1.18 onwards Go provides a structure that has been updated to include a new field Settings `[]debug.BuildSetting`
The (runtime/debug.BuildInfo)[https://pkg.go.dev/runtime/debug@master#BuildInfo] returned by [runtime/debug.ReadBuildInfo()]
(https://pkg.go.dev/runtime/debug@master#ReadBuildInfo) returns a key-value pairs describing a binary.
 In order to see this new feature in action we can use the following source:
 ```go
package main

import (
	"fmt"
	"runtime/debug"
)

func main() {
	fmt.Println("Hello build info!")
	info, ok := debug.ReadBuildInfo()
	if !ok {
		return
	}
	fmt.Println("Key:\tValue")
	for _, kv := range info.Settings {
		fmt.Println(kv.Key + ":\t" + kv.Value)
	}
}
 ```

We can now build our application as usual and then run it:
```console
$ go build -o main . 
$ ./main
  Hello build info!
  Key:	Value
  -compiler:	gc
  CGO_ENABLED:	1
  CGO_CFLAGS:	
  CGO_CPPFLAGS:	
  CGO_CXXFLAGS:	
  CGO_LDFLAGS:	
  GOARCH:	amd64
  GOOS:	darwin
  GOAMD64:	v1
  vcs:	git
  vcs.revision:	4071203188d039a852220d88dad45df0dbfaae7a
  vcs.time:	2022-01-15T16:47:19Z
  vcs.modified:	true
```

From those available build info we can review some Go specific attributes that used at build time such as the `GOARCH` and
the `GOOS`, and also within the `vcs` object we can see some specific attributes about the Version Control System (VCS)
that this app might use. In our case we have used the `git` VCS and there are some specific attributes set. The 
`vcs.revision` is the last's commit's hash, the `vcs.time` is the time that the commit was made and the `vcs.modified`
attribute signifies if the source has been modified or not since the last commit.

## Sum Up
We have seen how we can use `ldflags` to inject valuable information into Go binaries at build time. 
Using this feature we can pass through feature flags, environment information, versioning information, and more without altering our source code. Apart from this, from Go 1.18, we can exploit the `BuildSetting` available through the
`runtime/debug.BuildInfo` in order to retrieve valuable information about our VCS state at build time. Of course, we
can combine both solutions to achieve the desired outcome from our binaries!
