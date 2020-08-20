---
title: "Separating Tests in Go"
date: 2018-04-15T20:11:28+02:00
draft: false
categories:
  - Programming
tags:
  - Go
  - Tests
description: Different methods of separating tests in Go projects.
---

There are many different types of testing which you can do in order to ensure that your software is working correctly. These tests can
vary a lot in complexity and the time that it takes to complete them. 

Although the go testing package along with the `go tool` command provides us support in order to make automated tests for Go packages,
it is not immediately clear on how we should separate different tests for different testing scenarios.
This blog post will cover several great patterns to separate tests using the tools at our disposal.

### Build tags

A build tag, or a [build constraint](https://golang.org/pkg/go/build/#hdr-Build_Constraints) as it is described in the `build` package documentation,
is a line comment that begins with a `// +build` and then follow the conditions under which a file should be included in the package. You can have 
multiple build constraints for one file. The tags can then be passed to the `go build` command, when building the package, like so: 

`go build -tags <tags>`

This will allow us to separate the integration tests from the unit tests by firstly separating them in different files and then
giving each file a special build tag, let's say `integration` for our integration tests and `unit` for our unit tests.

⚠️ Make sure to add a **blank line** after the build tag, otherwise the test file will be omitted.

Example code:

```go
// +build integration

func TestIntegrationStuff(t *testing.T) {
    //...
}
```

The `test` command can also take tags just like the build command, so when we run our integration tests, we can type the command:


`go test -tags=integration`


### Short mode

The second method of separating tests that we can use is the short mode. The `go test` command provides us a flag for [short mode](https://golang.org/pkg/testing/#Short).
When we want to run our unit tests, we would use the `-short` flag, and omit it for running our integration tests or long running tests. Tests can check the
value of `testing.Short()` function in order to determine whether or not the code to be executed.

Example code:

```go
func TestIntegrationStuff(t *testing.T) {
    if testing.Short() {
        t.Skip()
    }

    //rest of the code..
}
```

This flag could also be used in the `TestMain` function in order to determine whether we should initialize our database or other external
services and do a clean up after we run our tests:

```go
func TestMain(m *testing.M) {
    flag.Parse()
    if !testing.Short() {
        setup()
    }

    res := m.Run()

    if !testing.Short() {
        teardown()
    }

    os.Exit(res)
}
```

The `-short` flag is useful for simple cases, but falls, *ahem*, short when we want to run different types of test scenarios and also 
we would need to pass the flag every time we want to avoid running our integration tests, which can be an inconvenience.

### The -run flag

Another great way of separating our tests is the `-run` flag which provides much greater flexibility than the `-short` flag. 
This flag takes a *regexp* as a value, which then the test command uses in order for it to run only the tests whose name matches against it.

For this to work, we would need a naming convention when we name our test functions and examples. For example, we could name our unit tests like `TestUnitFoo` and 
our integration tests `TestIntegrationFoo`. When we want to run our unit tests, we would then use `go test -run=Unit` and `go test -run=Integration` for our integration tests.

⚠️ Since -run involves regexp, take caution when naming your test functions, otherwise some of your tests will not run if they are not named properly.

You can also use these patterns together, based on your own needs, for example using -short as well as the -run flag, for an even more flexible solution.

There you go, three simple methods or patterns if you will, for separating your tests in Go. Here are some other useful links:

+ https://golang.org/pkg/testing/
+ https://golang.org/pkg/go/build/
+ https://github.com/stretchr/testify 


