---
title: Testing the Tests
description: If we use tests to test code, how can we test our tests?
image:
  src: /assets/posts/Testing-the-Tests/mobius.jpg
  show_in_post: false
date: 2021-09-10 21:15:00 +0300
categories: []
tags: [golang,coding,testing]
---

If we use tests to test code, how can we test our tests? We write tests in order to assure the quality of the software 
we are writing. Software developers are usually trying to reach a certain level of code coverage through their tests. 
However, the quality of the tests is not usually measured or examined through this process. It is possible that someone 
could achieve a 100% code coverage even without any assertion at all!

## Is it Fully Tested?
In a simple example we have a [fizzbuzz](https://en.wikipedia.org/wiki/Fizz_buzz) implementation where given an integer we return the desired string result:

```go
package fizzbuzz

import "strconv"

// CalculateFizzBuzz returns
// `fizz` if the given number is divisible by 3
// `buzz` if the given number is divisible by 5
// `fizzbuzz` if the given number is divisible by 3 and 5
// empty string if it is non positive
// else the number itself.
func CalculateFizzBuzz(num int) string {
	ret := ""
	if num <= 0 {
		return ret
	}
	if num%3 == 0 {
		ret += "fizz"
	}
	if num%5 == 0 {
		ret += "buzz"
	}
	if num%3 != 0 && num%5 != 0 {
		ret = strconv.Itoa(num)
	}
	return ret
}
```

Testing this function seems straight forward. We have to test all the `if` statements  of the function. 
Using table testing technique we can have the following result for our test cases:

```go
package fizzbuzz

import "testing"

func TestCalculateFizzBuzz(t *testing.T) {
	var tt = []struct {
		name     string
		given    int
		expected string
	}{
		{
			name:     "Given -1 expects empty string.",
			given:    -1,
			expected: "",
		},
		{
			name:     "Given 3 expects fizz.",
			given:    3,
			expected: "fizz",
		},
		{
			name:     "Given 5 expects buzz.",
			given:    5,
			expected: "buzz",
		},
		{
			name:     "Given 15 expects fizzbuzz.",
			given:    15,
			expected: "fizzbuzz",
		},
		{
			name:     "Given 7 expects 7.",
			given:    7,
			expected: "7",
		},
	}
	for _, tc := range tt {
		t.Run(tc.name, func(t *testing.T) {
			//Test
			got := CalculateFizzBuzz(tc.given)
			//Assert
			if got != tc.expected {
				t.Fatalf("expected: %s, got: %s, for CalculateFizzBuzz of %d", tc.expected, got, tc.given)
			}
		})
	}
}
```

Test coverage can be seen by executing these commands:
```
go test -coverprofile cover.out 
go tool cover -html=cover.out
```

and the result is:

![Coverage #1](/assets/posts/Testing-the-Tests/coverage1.png)

As we see our code is 100% covered by tests. Does this mean that we are safe to do any change to our code with the
safety net of our tests? Absolutely not! 

Let's see a simple example. Changing the line `if num <= 0 {` to `if num < 0 {` will still let all the tests to pass
successfully, however, we just changed our function's behavior with unexpected results in the execution.

Lets see another example. Removing these test cases:
```go
{
	name:     "Given 5 expects buzz.",
	given:    5,
	expected: "buzz",
},
{
	name:     "Given 15 expects fizzbuzz.",
	given:    15,
	expected: "fizzbuzz",
},
```
will also leave our coverage untouched and up to 100%. However, we have missed out to test specific cases of our 
function.

There is one certain technique that allows us to detect such cases. This technique involves the process to apply minor 
changes to our function and then check if the existing tests fail on each one of those! This process can be automated 
through mutation testing!

## Mutation to the rescue!
Mutation testing is used to design new software tests and evaluate the quality of existing software tests. Mutation 
testing involves modifying a program in small ways. Each mutated version is called a mutant and tests detect and reject 
mutants by causing the behavior of the original version to differ from the mutant. This is called killing the mutant. 
Test suites are measured by the percentage of mutants that they kill. New tests can be designed to kill additional 
mutants. Mutants are based on well-defined mutation operators that either mimic typical programming errors (such as 
using the wrong operator or variable name) or force the creation of valuable tests (such as dividing each expression by 
zero). The purpose is to help the tester develop effective tests or locate weaknesses in the test data used for the 
program or in sections of the code that are seldom or never accessed during execution. Mutation testing is a form of 
white-box testing. (https://en.wikipedia.org/wiki/Mutation_testing)

The mutation testing can be automated through a mutation library. Such libraries mutate the source code in minor ways through a specific rule-set and then they run the same tests against the produced mutant. In our case we used the 
[go-mutesting library](https://github.com/zimmski/go-mutesting) to run such a process.

```
>> go-mutesting/fizbuzz.go
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-718515268/fizzbuzz/fizbuzz.go.0" with checksum 8f404742ff5b871908bdf580620787c2
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-718515268/fizzbuzz/fizbuzz.go.1" with checksum df615b5ba4c9d06bb54cf605730d89b8
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-718515268/fizzbuzz/fizbuzz.go.2" with checksum 15e3b2a8de455eed5e05c73dbbf1263f
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-718515268/fizzbuzz/fizbuzz.go.3" with checksum 933cd06e194200a78f53199c1d3ddfb6
--- fizzbuzz/fizbuzz.go	2021-09-07 11:48:32.000000000 +0300
+++ /var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-718515268/fizzbuzz/fizbuzz.go.4	2021-09-07 22:37:29.000000000 +0300
@@ -10,7 +10,7 @@
 // else the number itself.
 func CalculateFizzBuzz(num int) string {
 	ret := ""
-	if num <= 0 {
+	if num < 0 {
 		return ret
 	}
 	if num%3 == 0 {

FAIL "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-718515268/fizzbuzz/fizbuzz.go.4" with checksum 9970f3c67a1e9211513823f2d4549314
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-718515268/fizzbuzz/fizbuzz.go.5" with checksum a26848fafc97ee7d8ef7d7a497a12f82
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-718515268/fizzbuzz/fizbuzz.go.6" with checksum 802ed71e97906862491cfaf7a506754d
The mutation score is 0.857143 (6 passed, 1 failed, 3 duplicated, 0 skipped, total is 7)
```

We can see that `go-mutesting` produced 7 mutants of our source code and just one of those failed.
In the failed one the altered line is mentioned as 
```go
-	if num <= 0 {
+	if num < 0 {
```

We can fix this by altering our first test case to:
```
{
	name:     "Given 0 expects empty string.",
	given:    0,
	expected: "",
},
```

Here is a demo of how our source code updated while running the mutation testing:

![mutating source code](/assets/posts/Testing-the-Tests/mutating.gif)

Also if we remove these test cases from our test (which also leaves code's test coverage up to 100%):
```go
{
	name:     "Given 5 expects buzz.",
	given:    5,
	expected: "buzz",
},
{
	name:     "Given 15 expects fizzbuzz.",
	given:    15,
	expected: "fizzbuzz",
},
```

and then run again the mutation testing we get this result:
```
 >> go-mutesting fizzbuzz/fizbuzz.go
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-254713350/fizzbuzz/fizbuzz.go.0" with checksum 8f404742ff5b871908bdf580620787c2
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-254713350/fizzbuzz/fizbuzz.go.1" with checksum df615b5ba4c9d06bb54cf605730d89b8
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-254713350/fizzbuzz/fizbuzz.go.2" with checksum 15e3b2a8de455eed5e05c73dbbf1263f
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-254713350/fizzbuzz/fizbuzz.go.3" with checksum 933cd06e194200a78f53199c1d3ddfb6
PASS "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-254713350/fizzbuzz/fizbuzz.go.4" with checksum 9970f3c67a1e9211513823f2d4549314
--- fizzbuzz/fizbuzz.go	2021-09-07 11:48:32.000000000 +0300
+++ /var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-254713350/fizzbuzz/fizbuzz.go.5	2021-09-07 22:53:28.000000000 +0300
@@ -19,7 +19,7 @@
 	if num%5 == 0 {
 		ret += "buzz"
 	}
-	if num%3 != 0 && num%5 != 0 {
+	if true && num%5 != 0 {
 		ret = strconv.Itoa(num)
 	}
 	return ret

FAIL "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-254713350/fizzbuzz/fizbuzz.go.5" with checksum a26848fafc97ee7d8ef7d7a497a12f82
--- fizzbuzz/fizbuzz.go	2021-09-07 11:48:32.000000000 +0300
+++ /var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-254713350/fizzbuzz/fizbuzz.go.6	2021-09-07 22:53:28.000000000 +0300
@@ -19,7 +19,7 @@
 	if num%5 == 0 {
 		ret += "buzz"
 	}
-	if num%3 != 0 && num%5 != 0 {
+	if num%3 != 0 && true {
 		ret = strconv.Itoa(num)
 	}
 	return ret

FAIL "/var/folders/jh/nh1wg3r93xv94jd2bwz8lzc80000gn/T/go-mutesting-254713350/fizzbuzz/fizbuzz.go.6" with checksum 802ed71e97906862491cfaf7a506754d
The mutation score is 0.714286 (5 passed, 2 failed, 3 duplicated, 0 skipped, total is 7)
```

As we still have 100% test coverage, mutation testing revealed cases where our tests fail to catch such alterations to the source code.

> The mutation score is calculated as the percentage ration of (number of killed mutants)/(number of total mutants).

## Mutation variations
Each mutation testing library can produce mutants by applying specific changes on the source code based on various 
rules. The most popular ones are:
* Branch mutators
* Expression mutators
* Statement mutators

It makes sense that mutation testing is pretty resource-intensive as there is a huge number of mutants that are usually generated for every code block.

Some mutations can cause a test not to finish, so generally, mutation libraries allow each test to run within a ten 
times compared to the original execution time.

Also many tests might generate false positives, so most of the libraries allows us to blacklist specific mutants in order to avoid noise in future runs.

## Sum Up
Even though we try to assure the quality of the software we are writing through the test, nothing validates the 
efficiency of the tests. Even code coverage of tests might be a misleading qualifier sometimes.

Mutation testing involves modifying a program in small ways so that to verify if tests cover each one of this variation (mutant). This way we can qualify if our tests are able to kill all of the mutants effectively. 

The process of mutating a source code and running the test suite is a pretty resource-intensive process so we should 
consider how often we can run the mutation testing towards our codebase.
