Assignment 04 - HPC and SQL
================
Shawn Ye

The learning objectives are to conduct data scraping and perform text
mining.

\#HPC \#\#Problem 1: Make sure your code is nice Rewrite the following R
functions to make them faster. It is OK (and recommended) to take a look
at Stackoverflow and Google

``` r
# Total row sums
fun1 <- function(mat) {
  n <- nrow(mat)
  ans <- double(n) 
  for (i in 1:n) {
    ans[i] <- sum(mat[i, ])
  }
  ans
}

fun1alt <- function(mat) {
  rowSums(mat)
}

# Cumulative sum by row
fun2 <- function(mat) {
  n <- nrow(mat)
  k <- ncol(mat)
  ans <- mat
  for (i in 1:n) {
    for (j in 2:k) {
      ans[i,j] <- mat[i, j] + ans[i, j - 1]
    }
  }
  ans
}

fun2alt <- function(mat) {
  n <- nrow(mat)
  ans <- mat
  for (i in 1:n) {
    ans[i,]=cumsum(mat[i,])
  }
  ans
}


# Use the data with this code
set.seed(2315)
dat <- matrix(rnorm(200 * 100), nrow = 200)

# Test for the first
microbenchmark::microbenchmark(
  fun1(dat),
  fun1alt(dat), unit = "relative", check = "equivalent"
)
```

    ## Unit: relative
    ##          expr      min       lq     mean   median       uq     max neval
    ##     fun1(dat) 8.317939 8.693522 7.405735 8.978052 8.002227 2.20299   100
    ##  fun1alt(dat) 1.000000 1.000000 1.000000 1.000000 1.000000 1.00000   100

``` r
# Test for the second
microbenchmark::microbenchmark(
  fun2(dat),
  fun2alt(dat), unit = "relative", check = "equivalent"
)
```

    ## Unit: relative
    ##          expr      min       lq     mean   median       uq       max neval
    ##     fun2(dat) 5.781128 3.967031 2.323984 3.925745 2.463122 0.2727468   100
    ##  fun2alt(dat) 1.000000 1.000000 1.000000 1.000000 1.000000 1.0000000   100

The last argument, check = “equivalent”, is included to make sure that
the functions return the same result.

\#\#Problem 2: Make things run faster with parallel computing The
following function allows simulating PI

``` r
sim_pi <- function(n = 1000, i = NULL) {
  p <- matrix(runif(n*2), ncol = 2)
  mean(rowSums(p^2) < 1) * 4
}

# Here is an example of the run
set.seed(156)
sim_pi(1000) # 3.132
```

    ## [1] 3.132

In order to get accurate estimates, we can run this function multiple
times, with the following code:

``` r
# This runs the simulation a 4,000 times, each with 10,000 points
set.seed(1231)
system.time({
  ans <- unlist(lapply(1:4000, sim_pi, n = 10000))
  print(mean(ans))
})
```

    ## [1] 3.14124

    ##    user  system elapsed 
    ##   3.912   1.298   5.283

Rewrite the previous code using parLapply() to make it run faster. Make
sure you set the seed using clusterSetRNGStream():

``` r
library(parallel)

system.time({
  cl <- makePSOCKcluster(4L)
  clusterSetRNGStream(cl, 1231)
  ans <-  unlist(parLapply(cl, 1:4000, sim_pi, n = 10000))
  print(mean(ans))
  stopCluster(cl)
})
```

    ## [1] 3.141578

    ##    user  system elapsed 
    ##   0.016   0.013   4.310

\#SQL Setup a temporary database by running the following chunk

``` r
# install.packages(c("RSQLite", "DBI"))

library(RSQLite)
library(DBI)

# Initialize a temporary in memory database
con <- dbConnect(SQLite(), ":memory:")

# Download tables
film <- read.csv("https://raw.githubusercontent.com/ivanceras/sakila/master/csv-sakila-db/film.csv")
film_category <- read.csv("https://raw.githubusercontent.com/ivanceras/sakila/master/csv-sakila-db/film_category.csv")
category <- read.csv("https://raw.githubusercontent.com/ivanceras/sakila/master/csv-sakila-db/category.csv")

# Copy data.frames to database
dbWriteTable(con, "film", film)
dbWriteTable(con, "film_category", film_category)
dbWriteTable(con, "category", category)
```

When you write a new chunk, remember to replace the r with sql,
connection=con. Some of these questions will reqruire you to use an
inner join. Read more about them here
<https://www.w3schools.com/sql/sql_join_inner.asp>

\#\#Question 1 How many many movies is there avaliable in each rating
catagory.

``` sql
SELECT rating, COUNT(*) 
FROM film
GROUP BY rating
```

<div class="knitsql-table">

| rating | COUNT(\*) |
| :----- | --------: |
| G      |       180 |
| NC-17  |       210 |
| PG     |       194 |
| PG-13  |       223 |
| R      |       195 |

5 records

</div>

\#\#Question 2 What is the average replacement cost and rental rate for
each rating category.

``` sql
SELECT rating, avg(replacement_cost),avg(rental_rate)
FROM film
GROUP BY rating
```

<div class="knitsql-table">

| rating | avg(replacement\_cost) | avg(rental\_rate) |
| :----- | ---------------------: | ----------------: |
| G      |               20.12333 |          2.912222 |
| NC-17  |               20.13762 |          2.970952 |
| PG     |               18.95907 |          3.051856 |
| PG-13  |               20.40256 |          3.034843 |
| R      |               20.23103 |          2.938718 |

5 records

</div>

\#\#Question 3 Use table film\_category together with film to find the
how many films there are witth each category ID

``` sql
select category_id , count(*)
from film_category a join film b
on a.film_id=b.film_id
group by category_id
```

<div class="knitsql-table">

| category\_id | count(\*) |
| :----------- | --------: |
| 1            |        64 |
| 2            |        66 |
| 3            |        60 |
| 4            |        57 |
| 5            |        58 |
| 6            |        68 |
| 7            |        62 |
| 8            |        69 |
| 9            |        73 |
| 10           |        61 |

Displaying records 1 - 10

</div>

\#\#Question 4 Incorporate table category into the answer to the
previous question to find the name of the most popular category.

``` sql
SELECT name AS category, a.category_id, COUNT(a.film_id) AS count
FROM film_category a 
  JOIN film b on a.film_id=b.film_id
  JOIN category c on a.category_id=c.category_id 
GROUP BY a.category_id
ORDER BY count DESC
```

<div class="knitsql-table">

| category    | category\_id | count |
| :---------- | -----------: | ----: |
| Sports      |           15 |    74 |
| Foreign     |            9 |    73 |
| Family      |            8 |    69 |
| Documentary |            6 |    68 |
| Animation   |            2 |    66 |
| Action      |            1 |    64 |
| New         |           13 |    63 |
| Drama       |            7 |    62 |
| Sci-Fi      |           14 |    61 |
| Games       |           10 |    61 |

Displaying records 1 - 10

</div>

Sports category is the most popular category with 74 films.

``` r
dbDisconnect(con)
```

PM566: Introduction to Health Data Science - PM 566 (Fall 2020)

University of Southern California

Department of Preventive Medicine

Meredith Franklin, George Vega-Yon, Emil Hvitfeldt

<meredith.franklin@usc.edu>

All content licensed under a Creative Commons
Attribution-NonCommercial-NoDerivatives 4.0 International License.

View the source at GitHub.
