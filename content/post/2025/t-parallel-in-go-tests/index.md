---
title: T.Parallel() In Go Tests
tags: [go]
author: "Yunier"
date: "2025-11-05"
description: "How to use T.Parallel in G tests."
---

Earlier this year I learned that I could make my Go unit test faster, it turned out I wasn't using the t.Parallel method to its full potential. Say you have the following package.

```go
package example

func PrintHello() string {
    return "Hello, World!"
}

func PrintGoodbye() string {
    return "Goodbye, World!"
}
```

With the following tests

```go
package example_test

import (
    "example"
    "fmt"
    "testing"
)

func Test(t *testing.T) {
    fmt.Println("Starting tests...")
    t.Run("PrintHello", func(t *testing.T) {
        fmt.Println("Running PrintHello test")
        result := example.PrintHello()
        expected := "Hello, World!"

        if result != expected {
            t.Errorf("Expected %q, got %q", expected, result)
        }
    })

    t.Run("PrintGoodbye", func(t *testing.T) {
        fmt.Println("Running PrintGoodbye test")
        result := example.PrintGoodbye()
        expected := "Goodbye, World!"

        if result != expected {
            t.Errorf("Expected %q, got %q", expected, result)
        }
    })
}
```

If I run the go test command on my terminal as follows.

```sh
go test
```

I got this result.

```sh
Starting tests...
Running PrintHello test
Running PrintGoodbye test
PASS
ok      example 0.495s
```

These tests are executed in 0.495 seconds. One way to speed the execution of these test is to to throw a t.Paralle at the top of the test function, like this.

```go
func Test(t *testing.T) {
    t.Parallel() // enable test parallelism
    fmt.Println("Starting tests...")
    t.Run("PrintHello", func(t *testing.T) {
        // omitted for brevity
    })

    t.Run("PrintGoodbye", func(t *testing.T) {
        // omitted for brevity
    })
}
```

Now when I run the go test command I get the following results.

```sh
Starting tests...
Running PrintHello test
Running PrintGoodbye test
PASS
ok      example 0.269s
```

Nice, the tests are faster, the problem with this and what I ended up learning is that we can make even further improvements by making the sub-tests use the t.Parallel function. For some reason I always thought that using t.Parallel at the top level made all the sub-tests run in parallel. Turns out I was mistaken.

```go
func Test(t *testing.T) {
    t.Parallel()
    fmt.Println("Starting tests...")
    t.Run("PrintHello", func(t *testing.T) {
        t.Parallel()
        // rest of the code omitted for brevity
    })

    t.Run("PrintGoodbye", func(t *testing.T) {
        t.Parallel()
        // rest of the code omitted for brevity 
    })
}
```

With each sub-test now using t.Parallel I get the following result when I execute the go test in my terminal.

```sh
Starting tests...
Running PrintHello test
Running PrintGoodbye test
PASS
ok      example 0.162s
```

Great, the tests are even faster. Now I know, these are small gains, but this is just an example, in a large repo where hundreds or even thousands of test are executed a change like this could improve your test execution time.

Now I have a few notes on t.Parallel.

1) Any top level function that does not call t.Parallel will block until all of its sub-test have completed, even if the sub-test does call t.Parallel.
2) t.Parallel is a blocking method, it makes the function stop and only resume when all non parallel methods have completed.
3) The maximum number of tests to run in parallel is determined by the GOMAXPROCS environment variable.
