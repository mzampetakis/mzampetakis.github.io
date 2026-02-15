---
title: Testing When Time Matters
description: Mocking out time sensitivity from unit tests.
image:
  path: /assets/posts/Testing-When-Time-Matters/time.jpg
  show_in_post: true
date: 2020-09-26 20:55:00 +0300
categories: []
tags: [golang,testing,time]
---

Software testing undeniably can provide objective, independent information about the quality of software. However, the testing development process can sometimes be not straightforward. There are cases - not so rare - that the development of a test requires a significant amount of effort and time. One of these cases is when we have to test code that is dependant on the current time.

## The issue

Developing processes that depend on the current time or elapsed time is not a rare case. There are many cases we need to use the current time and diverse a program's execution depending on time, duration, or a period. A few examples are schedulers, expirations, repetitive tasks, timeouts, and more...

A sample case of a library in Go that is dependant on time is a key-value store that uses expiration on each pair. Such a Store should have the following structure

```go
type DataStore struct {
	Data            map[string]mapValue
	DeletePrecision time.Duration
}

type MapValue struct {
	Expiration time.Time
	Value      interface{}
}
```

Each store has a `deletePrecision` duration which signifies the amount of time that it checks for expired data. Also `mapValue.expiration` is a `time.Time` which signifies the expiration of each `mapValue.value` that is added in the `DataStore`.

A function that checks and deletes expired `mapValues` could Periodically run every `deletePrecision` duration as seen bellow:

```go
func (ds *DataStore) DeleteExpiredKeys() {
	checkIntervalTicker := time.NewTicker(ds.DeletePrecision)
	for {
		<- checkIntervalTicker.C    // every ds.deletePrecision duration
		ds.checkAndDeleteExpiredKeys()
	}

}

func (ds *DataStore) checkAndDeleteExpiredKeys() {
	for key, data := range ds.Data {
		now := time.Now()
		if data.Expiration.Before(now) {
			delete(ds.Data, key)
		}
	}
}
```

So, when we need to test `checkAndDeleteExpiredKeys()` function we face an issue. Our function depends on an uncontrollable resource:  

**Current Time**  

This means that when we have to write a test for the `checkAndDeleteExpiredKeys()` function we should appropriately set up a mapValue object with an expiration, check the time elapsed and then run the `checkAndDeleteExpiredKeys()` function before any assertion. This leaves us to have a test with some sleeping time in order to avoid any false negatives in our test:

```go
func TestCheckAndDeleteExpiredKeys(t *testing.T) {
    ds := DataStore{
		Data:            map[string]mapValue{},
		DeletePrecision: precision,
    }
    ds.Data["akey"] = mapValue{
        expiration: ds.Clock.Now().Add(500*time.Millisecond),
		value:      "avalue",
    }
    time.Sleep(300 * time.Millisecond)
    ds.checkAndDeleteExpiredKeys()
    if _, exists := ds.Data["akey"] ; !exists {
        t.Errorf("Key-value not found but should")
    }
    time.Sleep(300 * time.Millisecond)
    ds.checkAndDeleteExpiredKeys()
    if _, exists := ds.Data["akey"] ; exists {
        t.Errorf("Key-value found but shouldn't")
    }
}
```

Looking into this test it is clear that, adding a delay in our test to verify the correctness of a function, is not a desirable process at all. Apart from the fact that our unit test now adds an extra delay to the testing process, it is clear that trying to minimize this delay might affect our test. Also, this way of testing makes our test less easy to maintain and more prone to errors.

### Some theory

As seen above our function is tightly coupled with a resource we don't have control on. **Current time**. Decoupling our function from this external resource is essential for unit testing. 

The most common way to achieve this decoupling is by using Dependency Injection (DI). 

According to [Wikipedia](https://en.wikipedia.org/wiki/Dependency_injection):

>In software engineering, dependency injection is a technique in which an object receives other objects that it depends on. These other objects are called dependencies. In the typical "using" relationship the receiving object is called a client and the passed (that is, "injected") object is called a service. The code that passes the service to the client can be many kinds of things and is called the injector. Instead of the client specifying which service it will use, the injector tells the client what service to use. The "injection" refers to the passing of a dependency (a service) into the object (a client) that would use it.
>Dependency injection is one form of the broader technique of inversion of control. A client who wants to call some services should not have to know how to construct those services. Instead, the client delegates the responsibility of providing its services to external code (the injector).

But the intent of this technique is this: 

>The intent behind dependency injection is to achieve separation of concerns of construction and use of objects. This can increase readability and code reusability.

## The solution

Coming back to our case, we should now follow the separation of concerns through DI in our scenario, in order to give us the ability to test our function. This means we should provide each DataStore object with an interface that implements a `Now()` function. This should return the current time in all cases but when we are testing our `checkAndDeleteExpiredKeys()` function we could override it.

So, `Datastore` should now have the following form:

```go
type DataStore struct {
    Data            map[string]mapValue
    DeletePrecision time.Duration
    Clock           Clock
}

// Clock is an interface that provides a single function
// to return the current time which is used for checking expiration.
type Clock interface {
	Now() time.Time
}
```

The default SystemClock provided below could be used by default in each New `Datastore`:

```go
// SystemClock implements Clock interface that uses time.Now().
var SystemClock = systemClock{}

type systemClock struct{}

func (t systemClock) Now() time.Time {
	return time.Now()
}
```

However, during our unit test, we can inject an implementation of the `Clock interface` that we could control the `Now()` function's results so that we can alter the "current" time based on the test!  
So our test can now be formed like this:

```go
type pastClock struct{}

var PastClock = pastClock{}

func (t pastClock) Now() time.Time {
	pt, _ := time.Parse(time.RFC3339, "2000-01-01T00:00:00+00:00")
	return pt
}

type futureClock struct{}

var FutureClock = futureClock{}

func (t futureClock) Now() time.Time {
	pt, _ := time.Parse(time.RFC3339, "3000-01-01T00:00:00+00:00")
	return pt
}

func TestCheckAndDeleteExpiredKeys(t *testing.T) {
    ds := DataStore{
		Data:            map[string]mapValue{},
        DeletePrecision: precision,
        Clock: PastClock
    }
    ds.Data["akey"]=mapValue{
		expiration: ds.Clock.Now().Add(500*time.Millisecond),
		value:      "avalue",
    }
    ds.checkAndDeleteExpiredKeys()
    if _, exists := ds.Data["akey"] ; !exists {
        t.Errorf("Key-value not found but should")
    }
    ds.Clock = FutureClock
    ds.checkAndDeleteExpiredKeys()
    if _, exists := ds.Data["akey"] ; exists {
        t.Errorf("Key-value found but shouldn't")
    }
}
```

In the test above, we generated two different implementors of the `Clock` interface: one providing a past time and one providing a future time. This provides to our test the ability to change between those two in between the assertions leaving the test with no possibility to create a false positive or negative and also without adding any extra delay.

> We could also use more fine-grained results in `Now()` functions of the two implementations in order to give a more precise result (in our case 500ms are enough!) between those two implementations.

## Other Approaches

It might be a little weird to Inject a Dependency in our library that will only be used while unit testing some of the provided functionalities. However, this is an essential process when we want to achieve such a decoupling.

Another commonly process that is using when we have to test a function which requires a resource that we do not control in **Monkey Patching** (or mocking). 

>Mocking is primarily used in unit testing. An object under test may have dependencies on other (complex) objects. To isolate the behavior of the object you want to replace the other objects by mocks that simulate the behavior of the real objects. This is useful if the real objects are impractical to incorporate into the unit test.
>
>In short, mocking is creating objects that simulate the behavior of real objects.

In our case we could use one of the available mocking libraries. Some of theme are listed bellow:
* [mock](https://github.com/golang/mock)
* [testify](https://github.com/stretchr/testify)
* [monkey](https://github.com/bouk/monkey)

## Wrap up

Decoupling a function from it's external resources is essential for unit testing. Dependency Injection is the technique that we usually exploit to achieve this decoupling. In our case we showed how to test a function that uses current time and injected it to the function under test. Monkey patching is another commonly pattern used when it comes to test resources we do not own. So, choose your preferred one and keep testing!
