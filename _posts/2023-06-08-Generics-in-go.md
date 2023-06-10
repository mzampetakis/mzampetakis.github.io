---
title: Generics in Go
description:
image:
  path: /assets/posts/Generics-In-Go/way-selections.jpg
  show_in_post: false
date: 2023-06-08 18:10:00 +0300
categories: []
tags: [golang]
---

A few days ago we had a discussion about generics in Go and how we use them in the development process. 
So, the reason for this post is to explore the way that generics work and present some use cases where generics can ease the development process and the maintenance of a project.

## Generics

From [wikipedia](https://en.wikipedia.org/wiki/Generic_programming):

> Generic programming is a style of computer programming in which algorithms are written in terms of types to-be-specified-later that are then instantiated when needed for specific types provided as parameters.

We can see that generics allows programmers to write algorithms that are loosely specify the types of the variables used. This allows our codebase to have less duplicate functions for similar functionalities.

## Go and generics

A great example for this is a Sum function that will be able to add every numeric value provided. And by numeric we mean: integers, floats etc.

In a traditional way we should implement different functions each one for each available type:

```go
func Sum(numbers []int) int {
	var sum int
	for _, number := range numbers {
		sum += number
  }
	return sum
}
```

With generics we can write the same code but this time it will be able to accept more than one of the available types:

```go
func Sum[T int | int64 | float64](numbers []T) T {
	var sum T
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

This function declares a new type `T` which can be one of `int | int64 | float64`. Argument should be an array of type `T` but also return value should be of type `T`. 
 
> `T` is a commonly used type name for generics but we could use anything else e.g. `Numeric`

We can also declare this type outside the function's scope like this:

```go
type Numerics interface {
	int | int64 | float64
}

func Sum[T Numerics](numbers []T) T {
	var sum T
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

In Generics arguments and return values should be known at compile time. We are not allowed to decide the return value at runtime. For instance the following function will fail at compile time:

```go
func CheckNumber[T int | error](number int) T {
  if number < 0 {
    return errors.New("Number should be positive")
  }
  return number
}
```

Another misconception is that we can mix different types of arguments even though the type makes it look possible. For instance this code block will fail at compile time:

```go
Add(1, 1.4)

func Add[T int | float64](number1 T, number2 T) T {
	return number1 + number2
}
```

There are lots of use cases that we can use generics. It might not always clear for everyone when we should use generics. If you find yourself writing the exact same code multiple times, where the only difference between the copies is that the code uses different types, consider whether you can use a type parameter.

> Generics introduced in Go v.1.18

### Underlying types

There are cases where we want to create a generic function but we would like to support custom types which are usually define within our programs. Lets say we have this generic function which returns the minimum from two given integer values:

```go
func Min[T uint | int | int64](a, b T) T {
	if a < b {
		return a
	}
	return b
}
```

This is totally fine. However we might want to use this with another type such as `time.Duration`. Duration is declared as `type Duration int64` within the `time` package. Even thought, durations is an int64, we are now allowed to use our `Min` function with this.

```go
minDuration := Min(time.Minute*59, time.Hour)
```

Will give:
> time.Duration does not satisfy uint, int, int64 (possibly missing ~ for int64 in uint, int, int64) 
{: .prompt-danger }

Go offers another wildcard here for the rescue. The tilde `~` character is used in order to tell the compiler that uint, int, int64, or any type whose underlying type is one of those types can be used.

> ~T means the set of all types whose underlying type is T

So, for our case we could change the min function as 
```go
func Min[T ~uint | ~int | ~int64](a, b T) T {
	if a < b {
		return a
	}
	return b
}
```

This will allow `Min` to compile even with types that their underlying type is one of those declared.

## Reflection

Go has run time reflection. Reflection permits a kind of generic programming. It permits us to write code that works with any type.

If some operation has to support even types that don’t have methods (so that interface types don’t help), and if the operation is different for each type (so that type parameters aren’t appropriate), we should use reflection. Reflection generally results in code that is much harder to maintain nad loses out on some compile-time type safety.

A function that calculates the minimum between two numeric values using reflection is like this:

```go 
func Min(a, b interface{}) interface{} {
	va := reflect.ValueOf(a)
	vb := reflect.ValueOf(b)

	switch va.Kind() {
	case reflect.Int, reflect.Int64:
		if va.Int() < vb.Int() {
			return a
		}
	case reflect.Uint, reflect.Uint64:
		if va.Uint() < vb.Uint() {
			return a
		}
	case reflect.Float32, reflect.Float64:
		if va.Float() < vb.Float() {
			return a
		}
	default:
		return nil
	}

	return b
}
```

## Wrap up

Generic programming is all about abstracting algorithms and data structures. 

Typically in Go, if you want to be able to use two different types for the same variable, you’d need to use either a specific interface or use interface{}, which allows any value to be used. Generics allow us to declare and use functions or types that are written to work with any of a set of types provided by calling code.
