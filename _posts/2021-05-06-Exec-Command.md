---
title: Exec.Command
description: Going a step further from executing system commands through Go.
image: /assets/posts/Exec-Command/cmd.jpg
show_image_post: false
date: 2021-05-06 19:40:00 +0300
categories: []
tags: [golang,coding,shell]
---

One of the key features of Go is the considerably rich standard library. 
[Go's os package](https://golang.org/pkg/os/) offers a wide range of methods and functions to allow programmers 
to exploit the host's OS capabilities. Using it leaves us with almost no need to invoke system commands. 
However, we need sometimes to execute a custom bash script or a system command not supported 
by some library or just invoke a 3rd party's executable file!

Go provides a pretty handy mechanism to manage - not just execute - system commands. The 
[os/exec package](https://golang.org/pkg/os/exec/) is the mechanism that Go provides to run external commands and 
we are going to examine it here!

> Unlike the "system" library call from C and other languages, the os/exec package intentionally does not invoke the 
> system shell and does not expand any glob patterns or handle other expansions, pipelines, or redirections typically 
> done by shells.

## Executing a command in Go
Executing a system command In Go can be initialized by invoking the os/exec's command `Command()`.
The most common way to execute the command is to use the `Run()` method:

```go
package main

import (
	"log"
	"os/exec"
)

func main() {
	cmd := exec.Command("date")
	err := cmd.Run()
	if err != nil {
		log.Fatalf("cmd.Run() failed with %s\n", err)
	}

}
```

In this example we are running the `date` command. The command is initialized through the `cmd := exec.Command("date")` 
command which returns an `exec.Cmd` struct to execute the named program with the given arguments. `cmd.Run()` 
executes the given command and the returned error is nil if the command runs, has no problems copying stdin, stdout, 
and stderr, and exits with a zero exit status.

## The cmd struct

The `exec.Cmd` struct is the return value of an `exec.Command()` function. 
Cmd represents an external command being prepared or run.
A Cmd cannot be reused after calling its Run, Output, or CombinedOutput methods which are presented at the following 
chapters. This struct contains all the information required for a command to be described before execution and its 
public fields are more or less self explanatory:

```go
type Cmd struct {
	Path string
	Args []string
	Env []string
	Dir string
	Stdin io.Reader
	Stdout io.Writer
	Stderr io.Writer
	ExtraFiles []*os.File
	SysProcAttr *syscall.SysProcAttr
	Process *os.Process
	ProcessState *os.ProcessState
}
```

In our previous example, using the `cmd := exec.Command("date")` we just initialized the Cmd struct with the `Path` of 
the date command and no data provided for the `Args`.

More details about the Cmd struct can be found [at the golang.org](https://golang.org/pkg/os/exec/#Cmd).

> Note - I’m writing this post on a MacOS using commands that may not work on other OS such as Linux or Windows.

## Capturing the command's output

In our example, after executing the `date` command we do not retrieve the returned data. It is really common task to 
retrieve the returned data after executing a command in order to process them appropriately.

```go
package main

import (
	"fmt"
	"log"
	"os/exec"
)

func main() {
	cmd := exec.Command("date")
	dt, err := cmd.Output()
	if err != nil {
		log.Fatalf("cmd.Run() failed with %s\n", err)
	}
	fmt.Println(string(dt))
}
```

The example above will print the current system's date upon running:
```
Mon  4 May 2021 22:46:33 EEST
```

The output returned by a `cmd.Output()` execution is a byte array that can the managed as a string for further 
processing.

## Providing arguments to the command

There are cases where we have to provide extra arguments to our command for execution. This is feasible using the same 
`exec.Command()` function where arguments can be provided after the command's name: 
`func Command(name string, arg ...string) *Cmd`. Each argument should be supplied as a different `arg`. For instance 
if we have to set up a command for the `ls -al /tmp/` command we have to invoke `exec.Command("ls", "-al", "/tam/")`.  

> exec.Command() is an example of a Variadic Function. Variadic Functions can be called with any number of trailing 
> arguments, therefore, we can pass in as many arguments to our initial command as we desire.

### Splitting command's args

Preparing to execute a command using Go's `exec.Command()` function can sometimes be painful - especially when we have 
to provide lots of arguments to our command. Thankfully we can prepare our command as one string and then use the 
`strings.fields()` function to split it to separate arguments.

```go
package main

import (
	"fmt"
	"log"
	"os/exec"
	"strings"
)

func main() {
	cmdStr := "ls -al /tmp/"
	cmdArgs := strings.Fields(cmdStr)
	cmd := exec.Command(cmdArgs[0], cmdArgs[1:]...)
	dt, err := cmd.Output()
	if err != nil {
		log.Fatalf("cmd.Run() failed with %s\n", err)
	}
	fmt.Println(string(dt))
}
```

In the example above we prepared our command `ls -al /tmp/` as a string, then split its fields, and used the first 
element as the command to execute while the rest of the fields used as arguments. 

> As Fields splits the string around each instance of one or more consecutive white space characters, we have to take 
> extra care when using it - especially when we have space characters within a single arguments.

## Other ways of executing a command

There are cases where we want to initiate a command's execution without having to wait until its completion. Especially 
when we have to execute long-running shell scripts where we don't expect some output we can initiate a command using the 
`Start()` method.

In some cases we just have to wait or completion of a command we can use the `Wait()` method. In addition to the `Run()` 
method, `Wait()` waits for the command to exit and waits for any copying to stdin or copying from stdout or stderr to 
complete.

We saw previously how we can capture a command's output using the `Output()` method. In cases we want to capture the 
standard error as well, then we can use the `CombinedOutput()` method. This method behaves exactly like the Output method 
and has the same signature.

## Using commands' pipes

In an `exec.Command()` we can provide the Stdin which specifies the process's standard input. The StdinPipe returns a 
pipe that will be connected to the command's standard input when the command starts. In the example below we can see 
how can we exploit this provided `Cmd.StdinPipe` in order to feed our `cat` command with some text upon startup:

```go
package main

import (
    "fmt"
    "io"
    "log"
    "os/exec"
)

func main() {
    cmd := exec.Command("cat")
    stdin, err := cmd.StdinPipe()
    if err != nil {
        log.Fatal(err)
    }
    go func() {
        defer stdin.Close()
        io.WriteString(stdin, "some random text")
    }()
    out, err := cmd.CombinedOutput()
    if err != nil {
        log.Fatal(err)
    }

    fmt.Printf("%s\n", out)
}
```

In this example we expect the output to be: `some random text`.  
Accordingly, we can exploit a command's `Cmd.StdoutPipe` in order to grab the output of the executed command as a stream.

## A hint 😉

As we saw there are various ways we are able to prepare a command for execution using Go's `exec.Command`. Most of the 
times we are adding parameters and conditional arguments while "preparing" a command for execution. Most of the times 
we have to log a command before execution. `exec.Command()` provides the `String()` method which allows us to print the 
final form of the command for execution.

```go
package main

import (
	"fmt"
	"os/exec"
	"strings"
)

func main() {
	command := "ls"
	path := "/tmp/"
	cmdStr := command + " -al " + path
	cmdArgs := strings.Fields(cmdStr)
	cmd := exec.Command(cmdArgs[0], cmdArgs[1:]...)
	fmt.Println(cmd.String())
}
```

The result of the program above should be: `ls -al /tmp/`. So, using the `String()` method of a command we can easily 
retrieve the command we are going to execute! 

## Sum Up

Using Go to manage and execute commands within our apps is a common use case. Go provides, through the exec package, a 
variety of functions and methods to support such tasks. By exploiting the Command struct we have a pretty powerful way 
to manage and execute commands according to our needs.
