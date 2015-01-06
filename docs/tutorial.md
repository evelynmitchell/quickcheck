# Assertion-based testing with Quickcheck





## Introduction

Quickcheck was originally a package for the language Haskell aiming to simplify the writing of tests. The main idea is the automatic generation of tests based on assertions a function needs to satisfy and the signature of that function. The idea spread to other languages and is now implemented in R with this package. Because of the differences in type systems between Haskell and other languages, the original idea morphed into something different for each language it was translated into, and R is no different. The main ideas retained are that tests are based on assertions and that the developer should not have to specify the inputs and output values of a test. The difference from Haskell is that the user needs to specify the type of each variable in an assertion with the optional possibility to fully specify its distribution. The main function in the package, `unit.test`, will randomly generate input values, execute the assertion and collect results. The advantages are multiple:

  - each test can be run multiple times on different data points, improving coverage and the ability to detect bugs;
  - test can run on large size inputs, possible but impractical in non-randomized testing;
  - assertions are more self-documenting than specific examples of the I/O relation. In fact, enough assertions can constitute a specification for the function being tested, but that's not necessary for testing to be useful.
  
## First example

Let's start with something very simple. Let's say we just wrote the function identity. Using the widely used testing package `testthat`, one would like to write a test like:


```r
library(testthat)
test_that("identity test", expect_identical(identity(x), x))
```

That in general doesn't work because `x` is not defined. What was meant was something like a quantifier *for all legal values of `x`*, but there isn't any easy way of implementing that. So a developer has to enter some values for `x`.


```r
for(x in c(1L, 1, list(1), NULL, factor(1)))
  test_that("identity test", expect_identical(identity(x),x))
```

But there is no good reason to pick those specific examples, testing on more data  points or larger values would increase the clutter factor, a developer may inadvertently inject unwritten assumptions in the choice of data points etc. `quickcheck` can solve or at lease alleviate all those problems


```r
library(quickcheck)
unit.test(assertion = function(x) identical(identity(x), x), generators = list(rinteger))
```

```
[1] "Pass  function (x)  \n identical(identity(x), x) \n"
```

```
[1] TRUE
```

We have supplied an assertion, that is a function returning a length-1 logical vector, where `TRUE` means *passed* and `FALSE` means *failed*, and a list of generators, one for each argument of the assertion -- use named or positional arguments as preferred.
What this means is that we have tested `identity` for this assertion on random integer vectors. We don't have to write them down one by one and later we will see how we can affect the distribution of such vectors, to make them potentially huge in size or value. We can also repeat the test multiple times on different values with the least amount of effort, in fact, we have already repeated the test 10 times, which is the default. But if 100 times is required, no problem:


```r
unit.test(assertion = function(x) identical(identity(x), x), generators = list(rinteger), sample.size = 100)
```

```
[1] "Pass  function (x)  \n identical(identity(x), x) \n"
```

```
[1] TRUE
```

Done! You see, if you had to write down those 100 integer vectors one by one, you'd never have time to. But, you may object, `identity` is not supposed to work only on integer vectors, why did we test only on those? That was just a starter indeed. Quickcheck contains a whole repertoire of random data generators, including `rinteger`, `rdouble`, `rcharacter` etc for most atomic types, and some also for non-atomic types such as `rlist` and `rdata.frame`. Notable omission is `rfunction` -- don't expect it any time soon. The library is easy to extend with your own generators (for instance, `rnorm` works out of the box) and offers a number of constructors for data generators such as `constant` and `mixture`. In particular, there is a generator `rany` that creates a mixture of all R types (in practice, the ones that `quickcheck` currently knows how to generate, but the intent is all of them). That is exactly what we need for our identity test.


```r
unit.test(assertion = function(x) identical(identity(x), x), generators = list(rany), sample.size = 100)
```

```
[1] "Pass  function (x)  \n identical(identity(x), x) \n"
```

```
[1] TRUE
```

Now we have more confidence that `identity` works for all types of R objects.

## Ways to define assertions

Unlike `testhat` where you need to construct specially defined *expectations*, `quickcheck` accepts run of the mill logical-valued functions, with the return value of length one. For example `function(x) all(x + 0 == x)` or `function(x) identical(x, rev(rev(x)))` are valid assertions -- independent of their success or failure. If an assertion returns TRUE, it is considered a success. If an assertion returns FALSE or generates an error, it is  considered a failure. For instance, `stop` is a valid assertion but always fails. How do I express the fact that this is its correct behaviour? Here is where the function `as.predicate` comes handy. It allows to convert a `testthat` expectation into a predicate, and expectations, which seem overkill to express the identity of two values,  are a powerful way of expressing less common requirements such as raising exceptions or generating warnings. In the case of `stop`, the appropriate expectations is `expect_error(stop())`. We can use it with `unit.test` as follows:


```r
unit.test(
  as.assertion(function(x) expect_error(stop(x))), 
  list(rcharacter))
```

```
[1] "Pass  function (...)  \n { \n     tryCatch({ \n         expectation(...) \n         TRUE \n     }, error = function(e) FALSE) \n } \n"
```

```
[1] TRUE
```

By executing this test successfully we have built confidence that the function `stop` will generate an error whenever called with any `character` argument.


## Ways to modify or define random data generators

There are built in random data generators for most built-in data types. They follow a simple naming conventions, "r" followed by the class name. For instance `rinteger` generates a random integer vector. Another characteristic of random data generators as defined in this package is that they have defaults for every argument, that is they can be called without arguments. Finally, the return value of different calls are statistically independent. For example


```r
rdouble()
```

```
[1] -1.0081  1.8832 -0.9290 -0.2942 -0.6150 -0.9471  0.5990 -1.5236
```

```r
rdouble()
```

```
[1] -0.57430 -1.39017 -0.07042 -0.43088 -0.59223  0.98112  0.53241 -0.09046
[9]  0.15649
```

As you can see, both elements and length change from one call to the next and in fact they are both random and independent. This is generally true for all generators, with the exception of the trivial generators created with `constant`. Most generators take two arguments, `element` and `size` which are meant to specify the distribution of the elements and size of the returned data structures and whose exact interpretation depends on the specific generator. In general, if element is a value it is construed as a desired expectation of the elements, if it is a function, it is called with a single argument to generate the elements of the random data structure. For example


```r
rdouble()
```

```
[1] -0.4302 -0.9261 -0.1771  0.4020 -0.7317  0.8304 -1.2081
```
generates some random double vector. The next expression does the same but with an expectation equal to 100

```r
rdouble(element = 100)
```

```
[1] 100.02  99.61  99.51  98.95  99.10 101.27
```
and finally this extracts the elements from a uniform distribution with all parameters at default values.

```r
rdouble(runif)
```

```
 [1] 0.78102 0.01115 0.94031 0.99375 0.35741 0.74764 0.79291 0.70586
 [9] 0.47583 0.49465 0.30805
```

The same is true for argument size. If not a function, it's construed as a length expectation, otherwise it is called with an single argument equal to 1 to generate a random length.

First form:

```r
rdouble(size = 1)
```

```
[1] 0.9261
```

```r
rdouble(size = 1)
```

```
[1] 0.4207
```

Second form:
```
rdouble(size = function(x) 10 * runif(x))
rdouble(size = function(x) 10 * runif(x))
```

Two dimensional data structures have the argument `size` replaced by `nrow` and `ncol`. Nested data structures have an argument `height`. All of these are intended to be averages and not deterministic values but can be replaced by a generator. If you need to define a test with a random vector of a specific length as input, use the generator constructor `constant`:

```r
rdouble(size = constant(3))
```

```
[1] -0.4002 -1.3702  0.9878
```

```r
rdouble(size = constant(3))
```

```
[1]  1.5197 -0.3087 -1.2533
```

The function returned by `constant(x)` is itself a generator, that we can use when we want to specify a deterministic value for a test


```r
unit.test(function(x, y) all(abs(x)/y == Inf), generators = list(rdouble, constant(0))) 
```

```
[1] "Pass  function (x, y)  \n all(abs(x)/y == Inf) \n"
```

```
[1] TRUE
```

Another simple constructor is `select` which creates a generator that picks randomly from a list, provided as argument -- not unlike `sample`, but consistent with the `quickcheck` definition of generator.


```r
select(1:5)()
```

```
[1] 5
```

```r
select(1:5)()
```

```
[1] 3
```

When passing generators to `unit.test`, one needs to pass a function, not a data set, so to provide custom arguments to generators one needs a little bit of functional programming, in the form of functions `Curry` from package `functional` or a more readable version there of, `fun` -- named as such because it returns a function but also because it is allegedly more fun to use. If `rdouble(element = 100)` generates data from the desired distribution, then a test would use it as follow


```r
unit.test(function(x) sum(x) > 100, list(Curry(rdouble, element = 100)))
```

```
Error: could not find function "Curry"
```

or the more readable


```r
unit.test(function(x) sum(x) > 100, list(fun(rdouble(element = 100))))
```

```
[1] "Pass  function (x)  \n sum(x) > 100 \n"
```

```
[1] TRUE
```

Note the the last two tests only pass with high probability. Sometimes accepting a high probability of passing is  a shortcut to writing an effective, simple test when a deterministic one is not available.

## Advanced topics

The alert reader may have already noticed how generators can be used to define other generators. For instance, a random list of double vectors can be generated with `rlist(rdouble)` and a list of list of the same with `rlist(fun(rlist(rdouble)))`. Composition can also be applied to tests, which can be used as assertions inside other tests. One application of this is developing a test that involves two random vectors of the same random length. There isn't a built in way in quickcheck to express this dependency between arguments, but the solution is not far using the composability of tests. We first pick a random length in the "outer" test, then using it to generate equal length vectors in the "inner" test.


```r
unit.test(
	function(l) 
		unit.test(
			function(x,y) 
				isTRUE(all.equal(x, x + y - y)), 
			list(
				fun(rdouble(size = constant(l))), 
				fun(rdouble(size = constant(l))))), 
	list(rinteger))
```

```
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (x, y)  \n isTRUE(all.equal(x, x + y - y)) \n"
[1] "Pass  function (l)  \n unit.test(function(x, y) isTRUE(all.equal(x, x + y - y)), list(fun(rdouble(size = constant(l))),  \n     fun(rdouble(size = constant(l))))) \n"
```

```
[1] TRUE
```