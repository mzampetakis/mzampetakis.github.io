---
title: Deep Dive in Go Channels
description: A brief deep dive in Go Channels with examples and common error cases.
image: /assets/posts/Deep-Dive-in-Go-Channels/channels.jpg
show_image_post: false
date: 2020-10-31 08:55:00 +0300
categories: []
tags: [golang,coding]
---

One of the biggest advantages of Go is undoubtedly it's concurrency management. Goroutines are the main feature that Go uses to achieve this. Goroutines wouldn't be so easy if there wasn't for channels.

> A goroutine is a lightweight thread managed by the Go runtime.

Channels are the pipes that connect concurrent goroutines. You can send values into channels from one goroutine and receive those values into another goroutine. 


## How channels work

Go provides a simple and straightforward mechanism to manage channels that allows goroutines to send and receive values from other gorourtines.

A new channel can be created by invoking the make command like this `make(chan value-type)`. value-type is the type of values that the generated channel will hold and transfer. 

Sending a value through a channel can be done by `a-channel <- a-value`.
Receiving a value from a channel can be done by `a-value := <- a-channel`.

The following example illustrates a simple scenario where a goroutine send a value to a channel while 

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	messages := make(chan string)
	go func() {
		time.Sleep(time.Second * 5)
		messages <- "ping"
	}()
	msg := <-messages
	fmt.Println(msg)
}
```

In our example, the main functions create a channel of type string and the spawned goroutine waits for 5" before sending a "ping" message through this channel. Main function blocks at receiving data from the channel at line 14 until the data is received and then prints the received data ("ping") and exits.

## Buffered and unbuffered channels

The example we saw exploited the functionality of an unbuffered channel. The size of the channel was `0` which shows the capacity of a channel. For an unbuffered channel, we can only send data if there is a corresponding receive statement - otherwise, it blocks.

To illustrate this example we can see that running the following example, `pong` message can be sent after 5" when the 2nd receive is initiated.

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	messages := make(chan string)
	go func() {
		messages <- "ping"
		fmt.Println("ping sent")
		messages <- "pong"
		fmt.Println("pong sent")
	}()
	fmt.Println(<-messages)
	time.Sleep(time.Second * 5)
	fmt.Println(<-messages)
}
```

The output we get from this example is:
```
>/ go run main.go
ping sent
ping
# here we have a 5" pause
pong
pong sent
```

We can see that send is blocked until the recipient is present.

Let's see now buffered channels in go. A buffered channel in go accepts a limited number of values without a corresponding receiver for those values. A buffered channel is created also by calling `make` but now an extra argument has to be provided that indicates the buffer size of the channel  `make(chan value-type, buffer-size)`.

Changing in the previous example the channel construction line to `messages := make(chan string, 2)` will produce the following output:

```
/> go run main.go
ping sent
pong sent
ping
# here we have a 5" pause
pong
```

## Bidirectional channels

We have seen that the same channel is used to send and receive data. However, when passing a channel to a function we can specify if the function is going to send or receive message(s) from the channel. This feature increases the type-safety of the program.

The way of defining this feature is at the functions header. A sender only function will be:

```go
func sender(messages chan<- string)
```

a receiver only function will be:

```go
func receiver(messages <-chan string)
```

## Closing a channel

Closing a channel signifies that no more data will be sent through this. This is a really useful feature that is used to inform receivers that channel's sending has been completed.

Closing a channel can be done through `close(messages)` and while receiving we have to check if the channel has closed. To do so we can add one more return value to the receiver of our channel:

```go
message, more := <-messages
if more {
    fmt.Println("received message", j)
} else {
    fmt.Println("received all messages")
}
```

## A real-world scenario

In a more real-word scenario now let's examine three two goroutines should handle sending and receiving data as a sequential process.

Let's assume we need 3 services that serially process data. The first one generates numbers (`Counter`) the second one doubles them (`Doubler`) and the last one prints the result (`Printer`). This is presented in the following schema:


    +---------------+            +---------------+            +---------------+
    |               |   1,2,3    |               |   2,4,6    |               |
    |    Counter    |+---------->|    Doubler    |+---------->|    Printer    |
    |               |  numbers   |               |  doubles   |               |
    +---------------+            +---------------+            +---------------+

All data exchange between goroutines should be explicitly done through different channels. Each goroutine should only send or receive to/from each channel.

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"sync"
	"syscall"
)

func main() {
	numbers := make(chan int)
	doubles := make(chan int)
	var wg sync.WaitGroup

	sigs := make(chan os.Signal, 1)
	signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)
	done := make(chan bool, 1)
	go func() {
		sig := <-sigs
		fmt.Println()
		fmt.Println(sig)
		done <- true
	}()
	
	wg.Add(3)
	go counter(&wg, numbers, done)
	go doubler(&wg, numbers, doubles)
	go printer(&wg, doubles)
	wg.Wait()
}

func counter(wg *sync.WaitGroup, numbers chan<- int, done <-chan bool) {
	defer wg.Done()
	totalSent := 0
	for x := 0; ; x++ {
		numbers <- x
		totalSent++
		select {
		case _, ok := <-done:
			if ok {
				close(numbers)
				fmt.Printf("Closing numbers. Total Sent: %d\n", totalSent)
				return
			}
		default:
		}
	}
}

func doubler(wg *sync.WaitGroup, numbers <-chan int, doubles chan<- int) {
	defer wg.Done()
	for {
		x, more := <-numbers
		if !more {
			close(doubles)
			fmt.Println("Closing doubles")
			return
		}
		doubles <- (2 * x)
	}
}

func printer(wg *sync.WaitGroup, doubles <-chan int) {
	defer wg.Done()
	totalReceived := 0
	for {
		d, more := <-doubles
		if !more {
			fmt.Printf("Closing Printer. Total Received %d\n", totalReceived)
			return
		}
		totalReceived++
		fmt.Println(d)
	}
}
```

We can see that all goroutines either consume or produce data to each given channel. `counter()` just sends to `numbers` channel, `doubler()` receives from `numbers` and sends to `doubles` and `printer()` just receives from `doubler` channel.

A `sync.WaitGroup` is used to allow our main function to wait for all three goroutines to end before exiting. This allows all threads to gracefully finish their jobs

Lines 14 to 23 are used to exit our application and gorourines. When a `SIGINT` or `SIGTERM` signals are received a special (for our application!) channel named `done` is used to signal the `counter()` function that it should stop producing new numbers. Closing the `numbers` allows us in the `doubler()` function to close the `doubles` channel as soon as all numbers have been processed. Then `printer()` can print all doubles and return.

## Sum Up

We have seen some basic concepts of channels in go. Channels allow our goroutines to exchange data in a thread-safe way. Lots of options are provided to meet an application's need. We've seen how to create channels. the difference between buffered and unbuffered channels, how to close a channel, and how to perceive that a channel has closed!

Channels are widely used in go apps especially when exploiting go's concurrency mechanisms. Hope that this post will give you a better understanding of channels in go.
