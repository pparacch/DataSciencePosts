# Non-Standard Evaluation in R
Pier Lorenzo Paracchini, `r format(Sys.time(), '%d.%m.%Y')`  



The content of this blog is based on notes collected and experiments performed while reading the __["Non-standard Evaluation"](http://adv-r.had.co.nz/Computing-on-the-language.html) chapter__ in [__"Advanced R"__](http://adv-r.had.co.nz/) by __Hadley Wickham__ [1]. The supporting [R markdown TBD]() used for generating these notes/ experiments can be find in the following [repository](https://github.com/pparacch/DataSciencePosts).

## Introduction

    "In most programming languages, you can only access the values of a function’s arguments. In R, you can also access the code used to compute them. This makes it possible to evaluate code in non-standard ways: to use what is known as non-standard evaluation, or NSE for short. NSE is particularly useful for functions when doing interactive data analysis because it can dramatically reduce the amount of typing." [1]

__NSE__ is about __accessing the code used to specify any valid R expression__ and used it in a __non standard way__.

## How to to capture expression

### The `substitute()` function

The `substitute(exp, env)` function in the `base` package is what makes NSE possible.

From `?substitute`

    "substitute returns the parse tree for the (unevaluated) expression expr, substituting any variables bound in env. ... [substitution] If it is an ordinary variable, its value is substituted, unless env is .GlobalEnv in which case the symbol is left unchanged." R Documentation

Something to note about this function   

    "... substitute() works because function arguments are represented by a special type of object called a promise. A promise captures the expression needed to compute the value and the environment in which to compute it. You’re not normally aware of promises because the first time you access a promise its code is evaluated in its environment, yielding a value." [1]


```r
#A simple function using SE
g <- function(x){
    x
}

#The same simple function using the substitute() function
f <- function(x){
    substitute(x)
}
g(1:10)
##  [1]  1  2  3  4  5  6  7  8  9 10
f(1:10)
## 1:10

x <- 10
g(x)
## [1] 10

f(x)
## x
typeof(f(x))
## [1] "symbol"

x <- 10
y <- 13
g(x + y^2)
## [1] 179

f(x + y^2)
## x + y^2
typeof(f(x + y^2))
## [1] "language"
```


```r
#Create a new environmnet
#create some bindings in the env
my_env <- new.env()
my_env$a = 1
my_env$b = 2
my_env$c = 3

#substituting any variables bound in env
substitute((a+x) + (b+y) + (c+z), my_env)
## (1 + x) + (2 + y) + (3 + z)
```


```r
#A tricky situation...

f <- function(x){
    substitute((x))
}
g <- function(x){
    #When calling f() x is the expression passed on
    #not its value.
    deparse(f(x))
}

g(1:10)
## [1] "(x)"
g(x)
## [1] "(x)"
g(x + y^2)
## [1] "(x)"
```

### The `quote()` function

Another way to capture expression is using the `quote()` function in the `base` package

    "simply returns its argument. The argument is not evaluated and can be any R expression." R Documentation

This function capture the provided expression as is, without performing any transformations.


```r
#The same simple function using the substitute() function
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

## How to use captured expressions

`The `deparse()` function is often used with the `substitute()` function. It turns unevaluated expressions into character strings.


```r
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

When using `deparse()` function, one argument to be aware is `width_cutoff` cause it can gives unexpected results. It defines the cut at which a line-breaking is tried. See example below


```r
f <- function(x){
    deparse(substitute(x), width.cutoff = 20L)
}

f(a + b + c + d + e + f + g + h + i + j + k + l + m +
  n + o + p + q + r + s + t + u + v + w + x + y + z)
```

```
## [1] "a + b + c + d + e + f + " "    g + h + i + j + k + "
## [3] "    l + m + n + o + p + " "    q + r + s + t + u + "
## [5] "    v + w + x + y + z"
```

```r
#A possible way to remove the possible splitting of the expression in more than
#one line is to write a wrapper around it 
g <- function(x){
    paste(gsub("^\\s+|\\s+$", "", deparse(substitute(x), width.cutoff = 20L)), collapse = " ")
}

g(a + b + c + d + e + f + g + h + i + j + k + l + m +
  n + o + p + q + r + s + t + u + v + w + x + y + z)
```

```
## [1] "a + b + c + d + e + f + g + h + i + j + k + l + m + n + o + p + q + r + s + t + u + v + w + x + y + z"
```

### Some applications


```r
#print out the implementing code
as.Date.default

#to create an error message (from the code)
stop(gettextf("do not know how to convert '%s' to class %s", 
        deparse(substitute(x)), dQuote("Date")), domain = NA)
```


```r
#print out the implementing code
pairwise.t.test

#to create the data name to be returned 
DNAME <- paste(deparse(substitute(x)), "and", deparse(substitute(g)))
```

## `subset()`, an example of NSE

The `subset()` function returns a subset of a provided dataframe (not only) which meet the provided conditions, minimizing the typing involved. See example below


```r
a_dataframe <- data.frame(a = 1:5, b = 5:1, c = c(6,7,8,9,10))
a_dataframe
##   a b  c
## 1 1 5  6
## 2 2 4  7
## 3 3 3  8
## 4 4 2  9
## 5 5 1 10

#lets select obs (row) where a is greater or equal to
subset(a_dataframe, a >=4)
##   a b  c
## 4 4 2  9
## 5 5 1 10

#equivalent to the SE
a_dataframe[a_dataframe$a >=4,]
##   a b  c
## 4 4 2  9
## 5 5 1 10
```

The implementation of the `subset` function for a dataframe can be found below


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
## <bytecode: 0x7fce3912cd18>
## <environment: namespace:base>
```

### The `eval()` function

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

### A simple `subset` implementation


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

## Scoping issues when using NSE

When using NSE, using expressions instead of values, things can go wrongs in different ways. When evaluating an expression using `eval` the look up for variables is done in

* (first) `envir`, the environment where `expr` is going to be evaluated.  
* if not found in `envir` the search will continue in `enclos` (enclosure).

Some examples when scoping goes wrong ....



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
#A simple example
subset2(a_dataframe, a == 4)
## a == 4
##   a b z
## 4 4 2 4

#################
#Another simple example that should give
#the same result as before

#When evaluating the expression y is not defined in the
#dataframe (envir), so it will be searched in the enclosure
#the calling environment when using a dataframe (as env)
#so y is found and the expression connected with the condition
#is evaluated.
y <- 4
subset2(a_dataframe, a == y)
## a == y
##   a b z
## 4 4 2 4

#################
#Another simple example that should give
#the same result as before

#this time x unfortunately is defined as one of the arguments of the function
#so the wrong value is picked up when the expression connected with the condition
#is evaluate. x is actually the dataframe itself
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

#another similar example
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

### Scoping with `parent.frame()`

See which environmnet is used for `env` when using the default value or setting the env specifically (always using `parent.frame()`).


```r
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
## <environment: 0x7fce3ccbcf88>
## <environment: 0x7fce3ccbcf88>


#When explicitly setting env will
#point to the calling environment, in this case
#the environment associated with the env 
#calling the function g1()
g1 <- function(){
    print(environment())
    f(envir = parent.frame())
}

g1()
## <environment: 0x7fce3cdcc2a0>
## <environment: R_GlobalEnv>
```


# References
[1] "Advanced R" by Hadley Wickham, ["Non-standard evaluation"](http://adv-r.had.co.nz/Computing-on-the-language.html) chapter 
