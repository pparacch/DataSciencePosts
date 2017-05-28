# Non-Standard Evaluation in R
Pier Lorenzo Paracchini, `r format(Sys.time(), '%d.%m.%Y')`  



<style type="text/css">
  .r code {
    font-size: 12px;
  }
</style>

## Acknowledgments

The content of this blog is based on notes collected and experiments performed while reading the __["Non-standard Evaluation"](http://adv-r.had.co.nz/Computing-on-the-language.html) chapter__ in [__"Advanced R"__](http://adv-r.had.co.nz/) by __Hadley Wickham__ [1]. The supporting [R markdown TBD]() used for generating these notes/ experiments can be find in the following [repository](https://github.com/pparacch/DataSciencePosts).

## Introduction

__Non-standard evaluation__(NSE) is about accessing the code used to specify any valid R expression and used it in a __non standard way__. While NSE is a great time-saver when working interactively, it can become a source of great headaches when using NSE functions in a standard way.

    "In most programming languages, you can only access the values of a function’s arguments. In R, you can also access the code used to compute them. This makes it possible to evaluate code in non-standard ways: to use what is known as non-standard evaluation, or NSE for short. NSE is particularly useful for functions when doing interactive data analysis because it can dramatically reduce the amount of typing." [1]

One of the biggest downside of NSE, as stated by __Hadley Wickham__ in [1], is that functions implemented using NSE are no longer _"[referentially transparent](https://en.wikipedia.org/wiki/Referential_transparency)"_

    "An expression is said to be referentially transparent if it can be replaced with its corresponding value without changing the program's behavior. As a result, evaluating a referentially transparent function gives the same value for same arguments. Such functions are called pure functions. An expression that is not referentially transparent is called referentially opaque.", Wikipedia

Using NSE makes a function __referentially opaque__ increasing the complexity when writing code e.g. functions that uses NSE functions or building R packages.

    "Non-standard evaluation allows you to write functions that are extremely powerful. However, they are harder to understand and to program with. As well as always providing an escape hatch, carefully consider both the costs and benefits of NSE before using it in a new domain." [1]

## How to implement NSE

### How to Capture Expressions

Two functions can be used to capture expressions, the `substitute()` and the `quote()` functions both part of the `base` package.

#### The `substitute()` function

The `substitute(exp, env)` function is what makes NSE possible.

From `?substitute` ...

    "substitute returns the parse tree for the (unevaluated) expression expr, substituting any variables bound in env. ... [substitution] If it is an ordinary variable, its value is substituted, unless env is .GlobalEnv in which case the symbol is left unchanged." R Documentation

Something to note about this function   

    "... substitute() works because function arguments are represented by a special type of object called a promise. A promise captures the expression needed to compute the value and the environment in which to compute it. You’re not normally aware of promises because the first time you access a promise its code is evaluated in its environment, yielding a value." [1]

Some examples ...


```r
rm(list = ls())
#A simple function using the substitute() function
#it returns the expression (not its value)
f <- function(x){
    substitute(x)
}
f(1:10)
## 1:10
typeof(f(1:10))
## [1] "language"

f(x)
## x
typeof(f(x))
## [1] "symbol"

f(x + y^2)
## x + y^2
typeof(f(x + y^2))
## [1] "language"
```


```r
rm(list=ls())
#substitute(), actually 
#substitutes any variables bound in the provided environment
#with its value (an exception for the Global Environment)

#Create a new environmnet
#create some bindings in the env
my_env <- new.env()
my_env$a = 1
my_env$b = 2
my_env$c = 3

#substituting any variables bound in the provided
#environment
substitute((a+x) + (b+y) + (c+z), env = my_env)
## (1 + x) + (2 + y) + (3 + z)
substitute(a + b + c, env = list(a = 1))
## 1 + b + c
```


```r
#A tricky situation with nested functions...
f <- function(x){
    substitute(x)
}
g <- function(x){
    #When calling f() x is the expression passed on
    #not its value.
    f(x)
}

g(1:10)
## x
g(x)
## x
g(x + y^2)
## x
#The expression is always x cause for the fuction f()
#x is the argument passed on from the function g()
```

#### The `quote()` function

Another way to capture expression is using the `quote()` function. It has a similar behaviour to the `substitute()` function.

From `?quote`...

    "... simply returns its argument. The argument is not evaluated and can be any R expression." R Documentation

This function capture the provided expression as is, without performing any transformations.


```r
rm(list=ls())
#The same simple function using the quote() function
f <- function(x){
    quote(x)
}
f(1:10)
## x
typeof(f(x))
## [1] "symbol"

f(x)
## x
typeof(f(x))
## [1] "symbol"

f(x + y^2)
## x
typeof(f(x + y^2))
## [1] "symbol"
```

### How to Use Captured Expressions

#### Create labels

The `deparse()` function is often used together with the `substitute()` function. The function turns __unevaluated expressions__ into __character strings__.


```r
rm(list = ls())
f <- function(x){
    deparse(substitute(x))
}

f(x)
## [1] "x"
typeof(f(x))
## [1] "character"

f(x + y^2)
## [1] "x + y^2"
typeof(f(x + y^2))
## [1] "character"
```

When using the `deparse()` function, one function argument to be aware of is `width_cutoff` cause it can gives unexpected results. It defines the cut at which a line-breaking is tried. See example below


```r
rm(list = ls())
f <- function(x){
    deparse(substitute(x), width.cutoff = 20L)
}

f(a + b + c + d + e + f + g + h + i + j + k + l + m +
  n + o + p + q + r + s + t + u + v + w + x + y + z)
## [1] "a + b + c + d + e + f + " "    g + h + i + j + k + "
## [3] "    l + m + n + o + p + " "    q + r + s + t + u + "
## [5] "    v + w + x + y + z"
#return a veactor of characters of length 5....


#A possible way to remove the possible splitting of the expression in more than
#one line is to write a wrapper around it 
g <- function(x){
    paste(gsub("^\\s+|\\s+$", "", deparse(substitute(x), width.cutoff = 20L)), collapse = " ")
}

g(a + b + c + d + e + f + g + h + i + j + k + l + m +
  n + o + p + q + r + s + t + u + v + w + x + y + z)
## [1] "a + b + c + d + e + f + g + h + i + j + k + l + m + n + o + p + q + r + s + t + u + v + w + x + y + z"
```

##### Some examples

Create an error message with info about teh provided data as in `as.Date.default` in `stop(gettextf("do not know how to convert '%s' to class %s", deparse(substitute(x)), dQuote("Date")), domain = NA)` ...


```r
#print out the implementing code
as.Date.default
## function (x, ...) 
## {
##     if (inherits(x, "Date")) 
##         return(x)
##     if (is.logical(x) && all(is.na(x))) 
##         return(structure(as.numeric(x), class = "Date"))
##     stop(gettextf("do not know how to convert '%s' to class %s", 
##         deparse(substitute(x)), dQuote("Date")), domain = NA)
## }
## <bytecode: 0x7f9cd3e59228>
## <environment: namespace:base>
```

Create some internal data structures as in `pairwise.t.test` in `DNAME <- paste(deparse(substitute(x)), "and", deparse(substitute(g)))` ...


```r
#print out the implementing code
pairwise.t.test
## function (x, g, p.adjust.method = p.adjust.methods, pool.sd = !paired, 
##     paired = FALSE, alternative = c("two.sided", "less", "greater"), 
##     ...) 
## {
##     if (paired & pool.sd) 
##         stop("pooling of SD is incompatible with paired tests")
##     DNAME <- paste(deparse(substitute(x)), "and", deparse(substitute(g)))
##     g <- factor(g)
##     p.adjust.method <- match.arg(p.adjust.method)
##     alternative <- match.arg(alternative)
##     if (pool.sd) {
##         METHOD <- "t tests with pooled SD"
##         xbar <- tapply(x, g, mean, na.rm = TRUE)
##         s <- tapply(x, g, sd, na.rm = TRUE)
##         n <- tapply(!is.na(x), g, sum)
##         degf <- n - 1
##         total.degf <- sum(degf)
##         pooled.sd <- sqrt(sum(s^2 * degf)/total.degf)
##         compare.levels <- function(i, j) {
##             dif <- xbar[i] - xbar[j]
##             se.dif <- pooled.sd * sqrt(1/n[i] + 1/n[j])
##             t.val <- dif/se.dif
##             if (alternative == "two.sided") 
##                 2 * pt(-abs(t.val), total.degf)
##             else pt(t.val, total.degf, lower.tail = (alternative == 
##                 "less"))
##         }
##     }
##     else {
##         METHOD <- if (paired) 
##             "paired t tests"
##         else "t tests with non-pooled SD"
##         compare.levels <- function(i, j) {
##             xi <- x[as.integer(g) == i]
##             xj <- x[as.integer(g) == j]
##             t.test(xi, xj, paired = paired, alternative = alternative, 
##                 ...)$p.value
##         }
##     }
##     PVAL <- pairwise.table(compare.levels, levels(g), p.adjust.method)
##     ans <- list(method = METHOD, data.name = DNAME, p.value = PVAL, 
##         p.adjust.method = p.adjust.method)
##     class(ans) <- "pairwise.htest"
##     ans
## }
## <bytecode: 0x7f9cd3984b08>
## <environment: namespace:stats>
```

#### Evaluate Expressions

The `eval()` function is used to evaluate an R expression in a specified environment. By default the calling environment (`envir = parent.frame()`) is used when evaluating an expression (if not esplicitly provided)


```r
#clean up current environment
rm(list = ls())

#variable x is created and set to 1 in the current env
eval(quote(x <- 1))
eval(quote(x))
## [1] 1
```


```r
#clean up current environment
rm(list = ls())

#If x is not found (current environment on) then
#an error is thrown
eval(quote(x))
## Error in eval(expr, envir, enclos): object 'x' not found
```


```r
#nested eval() with nested quote()
#each eval removes one quote level.
eval(quote(quote(quote(quote(2+2)))))
## quote(quote(2 + 2))
eval(quote(quote(quote(2+2))))
## quote(2 + 2)
eval(eval(quote(quote(quote(2+2)))))
## 2 + 2
eval(eval(eval(quote(quote(quote(2+2))))))
## [1] 4
```

__Providing a specific environment (as an `environment`, a `list`, a `data.frame`) for evaluating an expression...__


```r
rm(list = ls())

#Env provided as a list
x <- 10
eval(quote(x), envir = list(x = 55))
## [1] 55

y <- "a string"
#Env provided as an env
my_env <- new.env()
my_env$y <- 15
eval(quote(y), envir = my_env)
## [1] 15

eval(quote(x), envir = data.frame(x = c(55,56,57,60)))
## [1] 55 56 57 60
```

__A common mistake is to use `eval` without passing an expression...__


```r
rm(list = ls())
a <- 10
a_dataframe <- data.frame(a = 1:5, b = 5:1, c = c(6,7,8,9,10), z = 1:5)

#passing a R expression, envir is used to evaluate the expression
#a (feature) is found into the dataframe
eval(quote(a), envir = a_dataframe)
## [1] 1 2 3 4 5

#passing a variable, the current environment is used to find that value
#a is found into the current environment (envir is not used)
eval(a, envir = a_dataframe)
## [1] 10

#passing a variable, the current environment is used to find that value
#z is not found into the current environment (envir is not used)
#an error is thrown
eval(z, envir = a_dataframe)
## Error in eval(z, envir = a_dataframe): object 'z' not found
```


### An example of NSE implementation: `subset()` 

The `subset()` function returns a subset of a provided dataframe which meet the provided conditions, minimizing the typing involved. The function is implemented using NSE. See the example below


```r
a_dataframe <- data.frame(a = 1:5, b = 5:1, c = c(6,7,8,9,10))
a_dataframe
##   a b  c
## 1 1 5  6
## 2 2 4  7
## 3 3 3  8
## 4 4 2  9
## 5 5 1 10

# (NSE) lets select obs (row) where a is greater or equal to...
subset(a_dataframe, a >=4)
##   a b  c
## 4 4 2  9
## 5 5 1 10

#(SE) equivalent to...
a_dataframe[a_dataframe$a >=4,]
##   a b  c
## 4 4 2  9
## 5 5 1 10
```

The implementation of the `subset()` function for a dataframe can be found below


```r
getAnywhere(subset.data.frame())
## A single object matching 'subset.data.frame' was found
## It was found in the following places
##   package:base
##   registered S3 method for subset from namespace base
##   namespace:base
## with value
## 
## function (x, subset, select, drop = FALSE, ...) 
## {
##     r <- if (missing(subset)) 
##         rep_len(TRUE, nrow(x))
##     else {
##         e <- substitute(subset)
##         r <- eval(e, x, parent.frame())
##         if (!is.logical(r)) 
##             stop("'subset' must be logical")
##         r & !is.na(r)
##     }
##     vars <- if (missing(select)) 
##         TRUE
##     else {
##         nl <- as.list(seq_along(x))
##         names(nl) <- names(x)
##         eval(substitute(select), nl, parent.frame())
##     }
##     x[r, vars, drop = drop]
## }
## <bytecode: 0x7f9cd32d33c8>
## <environment: namespace:base>
```


#### A simple `subset()` re-implementation

Reusing the concepts of captured expressions and evaluating expression it is possible to re-implement a simplified NSE version of the function... 


```r
rm(list = ls())
a_dataframe <- data.frame(a = 1:5, b = 5:1, c = c(6,7,8,9,10), z = 1:5)

subset2 <- function(x, condition){
    condition_exp <- substitute(condition)
    condition_evaluated <- eval(condition_exp, envir = x)
    x[condition_evaluated,]
}

subset2(a_dataframe, a >=2)
##   a b  c z
## 2 2 4  7 2
## 3 3 3  8 3
## 4 4 2  9 4
## 5 5 1 10 5
```

## Challenges when using NSE functions in a SE way

### Scoping issues when using NSE

When using NSE, using expressions instead of values, things can go wrongs in different ways. When evaluating an expression using `eval` the look up for variables is done in

* (first) `envir`, the environment where `expr` is going to be evaluated.  
* if not found in `envir` the search will continue in `enclos` (enclosure).

Some examples when scoping goes wrong can be found in the following blocks of code...



```r
rm(list = ls())

a_dataframe <- data.frame(a = 1:5, b = 5:1, z = 1:5)

subset2 <- function(x, condition){
    condition_exp <- substitute(condition)
    print(condition_exp)
    condition_evaluated <- eval(condition_exp, envir = x)
    x[condition_evaluated,]
}

#################
#A working-as-expected example
subset2(a_dataframe, a == 4)
## a == 4
##   a b z
## 4 4 2 4

#################
#Another simple example that should give
#the same result as before working-as-expected

#Explanation
#When evaluating the expression y is not defined in the
#dataframe (envir), so it will be searched in the enclosure
#the calling environment when using a dataframe (as env)
#so y is found and the expression connected with the condition
#is evaluated giving the expected outcome (being lucky???)
#Y is found not in teh calling environment but in its parent
y <- 4
subset2(a_dataframe, a == y)
## a == y
##   a b z
## 4 4 2 4

#################
#Another simple example that should give
#the same result as before but not working-as-expected

#Explanation
#this time x unfortunately is defined as one of the arguments of the function
#so it is defined in the calling environment. Then the wrong value is 
#picked up when the expression connected with the condition
#is evaluate (x is the dataframe itself within the calling environment)
#(x <- 4 is not used)
x <- 4
subset2(a_dataframe, a == x)
## a == x
##       a  b  z
## 1     1  5  1
## 2     2  4  2
## 3     3  3  3
## 4     4  2  4
## 5     5  1  5
## NA   NA NA NA
## NA.1 NA NA NA
## NA.2 NA NA NA
## NA.3 NA NA NA
## NA.4 NA NA NA
## NA.5 NA NA NA

#another similar example like the one before but
#with a more criptical error
condition <- 4
subset2(a_dataframe, a == condition)
## a == condition
## Error in eval(expr, envir, enclos): object 'a' not found
```

We can tell `eval` to use the calling environment for trying to find the missing variables using the `enclos` argument.


```r
rm(list = ls())
a_dataframe <- data.frame(a = 1:5, b = 5:1, z = 1:5)

subset2 <- function(x, condition){
    condition_exp <- substitute(condition)
    print(condition_exp)
    condition_evaluated <- eval(condition_exp, envir = x, enclos = parent.frame())
    x[condition_evaluated,]
}

x <- 4
subset2(a_dataframe, a == x)
## a == x
##   a b z
## 4 4 2 4
```

#### A note on scoping with `parent.frame()`

The environmnet used for `envir` when using the default setting or setting the env specifically to `parent.frame()` are actually different, see example below.


```r
rm(list = ls())
#envir is set to parent.frame() (default) if not esplicitally provided.
f <- function(envir = parent.frame()){
    print(envir)
}

g <- function(){
    print(environment())
    f()
}
#When using the default setting env will
#point to the calling environment, in this case
#the environment associated with the function g
#calling f()
g()
## <environment: 0x7f9cd372a520>
## <environment: 0x7f9cd372a520>


#When explicitly setting env will
#point to the calling environment, in this case
#the environment associated with the env 
#calling the function g1()
g1 <- function(){
    print(environment())
    f(envir = parent.frame())
}

g1()
## <environment: 0x7f9cd40d2e70>
## <environment: R_GlobalEnv>
```

### Calling NSE functions from other functions

__Using NSE functions within function (non interactively) can have unpredictable behaviours.__ While `subset()` is a great function when working interactively through the console, it’s actually difficult to use non-interactively e.g. calling it within a function. In order to understand the complexity just have a look at the following example


```r
rm(list = ls())
a_dataframe <- data.frame(a = 1:5, b = 5:1, z = 1:5)

subset2 <- function(x, condition){
  print("--> subset2 (current. env, parent.frame)")
  print(environment())
  print(parent.frame())
  condition_exp <- substitute(condition)
  print(condition_exp)
  condition_evaluated <- eval(expr = condition_exp, envir = x, enclos = parent.frame())
  x[condition_evaluated,]
}

scramble <- function(x){
  print("--> scramble (current. env, parent.frame)")
  print(environment())
  print(parent.frame())
  x[sample(nrow(x)),]
}

subscramble <- function(x, condition){
  print("--> subscramble (current. env, parent.frame)")
  print(environment())
  print(parent.frame())
  scramble(subset2(x, condition))
}

#An error is generated by the next line of code...
#a <- 5:9 #Comment out to remove the error
subscramble(a_dataframe, a >= 4)
## [1] "--> subscramble (current. env, parent.frame)"
## <environment: 0x7f9cd3072e70>
## <environment: R_GlobalEnv>
## [1] "--> scramble (current. env, parent.frame)"
## <environment: 0x7f9cd3074ce8>
## <environment: 0x7f9cd3072e70>
## [1] "--> subset2 (current. env, parent.frame)"
## <environment: 0x7f9cd3120098>
## <environment: 0x7f9cd3072e70>
## condition
## Error in eval(expr, envir, enclos): object 'a' not found

#The reasong behind the error is that substitute set condition_exp to
#condition. condition is not set in the provided dataframe so it will
#be searched for in the calling environment (inside the subscrample function)
#and it will be found and evaluated but a is not found in the calling environment (up)
#so the error is thrown. Things could get very unpredictable if a is defined
#comment out the code setting a for surprises.
```


    "This is an example of the general tension between functions that are designed for interactive use and functions that are safe to program with. A function that uses substitute() might reduce typing, but it can be difficult to call from another function. As a developer, you should always provide an escape hatch: an alternative version of the function that uses standard evaluation." [1]

## How to use NSE functions within your code

Most functions that use NSE provide an alternative version of the function that uses SE (Standard Evaluation), this version of the functions should be used.

__But__ sometimes there are functions that do not provide this alternative version so __what is the best approach in this case?__ 

* Implement an alternative version to be used for SE or 
* not use such function. 

### The `substitute()` example

`substitute()` is a function that uses NSE and does not have an alternative version to be used for SE. When using `substitute()`, the variables bound in the environment `env` are replaced in the expression using the following rules:

(1) an __ordinary variable__, it's replaced by its value
(2) a __promise__ (a function argument), it's replaced by expression associated with the promise
(3) `...`, it's replaced by the content of `...`
(4) lest as is.

__Some examples using ordinary variables...__


```r
rm(list = ls())
f <- function(){
    a <- 1
    b <- 2
    #Rule 1. applied
    substitute(a + b + z)
}
f()
## 1 + 2 + z
```


```r
rm(list = ls())
f <- function(){
    a <- quote(mpg)
    b <- quote(disp)
    data <- quote(mtcars)
    #Rule 1. applied
    substitute(lattice::xyplot(a ~ b, data = data))
}
f()
## lattice::xyplot(mpg ~ disp, data = mtcars)
```


```r
rm(list = ls())
#Rule 1. applied
substitute(a + b, env = list(a = "y"))
## "y" + b
substitute(a + b, env = list(a = quote(y)))
## y + b
substitute(a + b, env = list(a = quote(y())))
## y() + b
substitute(a + b, env = list("+" = quote(f)))
## f(a, b)
substitute(a + b, env = list("+" = quote(`*`)))
## a * b
```

__Examples using promises (function arguments)...__


```r
rm(list = ls())
f <- function(x, y, data){
    #Rule 2. applied - working on promises
    #A promise is replaced with an expression associated with the promise
    substitute(lattice::xyplot(x ~ y, data = data))
}

f(mpg, disp, data = mtcars)
## lattice::xyplot(mpg ~ disp, data = mtcars)
```

__Examples using `...`...__


```r
rm(list = ls())
f <- function(x,y, ...){
    #Rule 3. applied (together with Rule 2.)
    substitute(lattice::xyplot(x ~ y, ...))
}

f(mpg, disp, data = mtcars, col = "red")
## lattice::xyplot(mpg ~ disp, data = mtcars, col = "red")
```

#### Adding an escape hatch

# References
[1] "Advanced R" by Hadley Wickham, ["Non-standard evaluation"](http://adv-r.had.co.nz/Computing-on-the-language.html) chapter 
