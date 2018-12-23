# DBI and SQL (11c)

## This chapter:
> 
> * Introduces more DBI functions and demonstrates techniques for submitting SQL to the dbms
> * Illustrates some of the differences between writing `dplyr` commands and SQL
> * Suggests some strategies for dividing the work between your local R session and the dbms

### Setup

The following packages are used in this chapter:

```r
library(tidyverse)
library(DBI)
library(RPostgres)
library(dbplyr)
require(knitr)
library(bookdown)
library(sqlpetr)
```
Assume that the Docker container with PostgreSQL and the dvdrental database are ready to go. If not go back to [the previous Chapter][Build the pet-sql Docker Image]

```r
sp_docker_start("sql-pet")
```
Connect to the database:

```r
con <- sp_get_postgres_connection(
  user = Sys.getenv("DEFAULT_POSTGRES_USER_NAME"),
  password = Sys.getenv("DEFAULT_POSTGRES_PASSWORD"),
  dbname = "dvdrental",
  seconds_to_test = 30
)
```

## SQL in R Markdown

When you create a report to run repeatedly, you might want to put that query into R markdown. See the discussion of [multiple language engines in R Markdown](https://bookdown.org/yihui/rmarkdown/language-engines.html#sql). That way you can also execute that SQL code in a chunk with the following header:

  {`sql, connection=con, output.var = "query_results"`}


```sql
SELECT "staff_id", COUNT(*) AS "n"
FROM "rental"
GROUP BY "staff_id";
```
Rmarkdown stored that query result in a tibble:

```r
query_results
```

```
##   staff_id    n
## 1        2 8004
## 2        1 8040
```
## DBI Package

In this chapter we touched on a number of functions from the DBI Package.  The table in file 96b shows other functions in the package.  The Chapter column references a section in the book if we have used it.


```r
film_table <- tbl(con, "film")
```

### Retrieve the whole table

SQL code that is submitted to a database is evaluated all at once^[From R's perspective. Actually there are 4 steps behind the scenes.].  To think through an SQL query, either use dplyr to build it up step by step and then convert it to SQL code or an IDE such as [pgAdmin](https://www.pgadmin.org/). DBI returns a data.frame, so you don't have dplyr's guardrails.

```r
res <- dbSendQuery(con, 'SELECT "title", "rental_duration", "length"
FROM "film"
WHERE ("rental_duration" > 5.0 AND "length" > 117.0)')

res_output <- dbFetch(res)
str(res_output)
```

```
## 'data.frame':	202 obs. of  3 variables:
##  $ title          : chr  "African Egg" "Alamo Videotape" "Alaska Phantom" "Alley Evolution" ...
##  $ rental_duration: int  6 6 6 6 6 7 6 7 6 6 ...
##  $ length         : int  130 126 136 180 181 179 119 127 170 162 ...
```

```r
dbClearResult(res)
```

### Or a chunk at a time


```r
res <- dbSendQuery(con, 'SELECT "title", "rental_duration", "length"
FROM "film"
WHERE ("rental_duration" > 5.0 AND "length" > 117.0)')

set.seed(5432)

chunk_num <- 0
while (!dbHasCompleted(res)) {
  chunk_num <- chunk_num + 1
  chunk <- dbFetch(res, n = sample(7:13,1))
  # print(nrow(chunk))
  chunk$chunk_num <- chunk_num
  if (!chunk_num %% 9) {print(chunk)}
}
```

```
##                 title rental_duration length chunk_num
## 1      Grinch Massage               7    150         9
## 2     Groundhog Uncut               6    139         9
## 3       Half Outfield               6    146         9
## 4       Hamlet Wisdom               7    146         9
## 5       Harold French               6    168         9
## 6        Hedwig Alter               7    169         9
## 7     Holes Brannigan               7    128         9
## 8     Hollow Jeopardy               7    136         9
## 9  Holocaust Highball               6    149         9
## 10          Home Pity               7    185         9
## 11     Homicide Peach               6    141         9
## 12    Hotel Happiness               6    181         9
##                     title rental_duration length chunk_num
## 1        Towers Hurricane               7    144        18
## 2                Town Ark               6    136        18
## 3       Trading Pinocchio               6    170        18
## 4 Trainspotting Strangers               7    132        18
## 5          Uncut Suicides               7    172        18
## 6    Unforgiven Zoolander               7    129        18
## 7         Uprising Uptown               6    174        18
## 8             Vanilla Day               7    122        18
## 9         Vietnam Smoochy               7    174        18
```

```r
dbClearResult(res)
```

## Dividing the work between R on your machine and the DBMS

They work together.

### Make the server do as much work as you can

* show_query as a first draft of SQL.  May or may not use SQL code submitted directly.

### Criteria for choosing between `dplyr` and native SQL

This probably belongs later in the book.

* performance considerations: first get the right data, then worry about performance
* Trade offs between leaving the data in PostgreSQL vs what's kept in R: 
  + browsing the data
  + larger samples and complete tables
  + using what you know to write efficient queries that do most of the work on the server

Where you place the `collect` function matters.
Here is a typical string of dplyr verbs strung together with the magrittr `%>%` command that will be used to tease out the several different behaviors that a lazy query has when passed to different R functions.  This query joins three connection objects into a query we'll call `Q`:


```r
rental_table <- dplyr::tbl(con, "rental")
staff_table <- dplyr::tbl(con, "staff") 
# the 'staff' table has 2 rows
customer_table <- dplyr::tbl(con, "customer") 
# the 'customer' table has 599 rows

Q <- rental_table %>%
  left_join(staff_table, by = c("staff_id" = "staff_id")) %>%
  rename(staff_email = email) %>%
  left_join(customer_table, by = c("customer_id" = "customer_id")) %>%
  rename(customer_email = email) %>%
  select(rental_date, staff_email, customer_email)
```


```r
Q %>% show_query()
```

```
## <SQL>
## SELECT "rental_date", "staff_email", "customer_email"
## FROM (SELECT "rental_id", "rental_date", "inventory_id", "customer_id", "return_date", "staff_id", "last_update.x", "first_name.x", "last_name.x", "address_id.x", "staff_email", "store_id.x", "active.x", "username", "password", "last_update.y", "picture", "store_id.y", "first_name.y", "last_name.y", "email" AS "customer_email", "address_id.y", "activebool", "create_date", "last_update", "active.y"
## FROM (SELECT "TBL_LEFT"."rental_id" AS "rental_id", "TBL_LEFT"."rental_date" AS "rental_date", "TBL_LEFT"."inventory_id" AS "inventory_id", "TBL_LEFT"."customer_id" AS "customer_id", "TBL_LEFT"."return_date" AS "return_date", "TBL_LEFT"."staff_id" AS "staff_id", "TBL_LEFT"."last_update.x" AS "last_update.x", "TBL_LEFT"."first_name" AS "first_name.x", "TBL_LEFT"."last_name" AS "last_name.x", "TBL_LEFT"."address_id" AS "address_id.x", "TBL_LEFT"."staff_email" AS "staff_email", "TBL_LEFT"."store_id" AS "store_id.x", "TBL_LEFT"."active" AS "active.x", "TBL_LEFT"."username" AS "username", "TBL_LEFT"."password" AS "password", "TBL_LEFT"."last_update.y" AS "last_update.y", "TBL_LEFT"."picture" AS "picture", "TBL_RIGHT"."store_id" AS "store_id.y", "TBL_RIGHT"."first_name" AS "first_name.y", "TBL_RIGHT"."last_name" AS "last_name.y", "TBL_RIGHT"."email" AS "email", "TBL_RIGHT"."address_id" AS "address_id.y", "TBL_RIGHT"."activebool" AS "activebool", "TBL_RIGHT"."create_date" AS "create_date", "TBL_RIGHT"."last_update" AS "last_update", "TBL_RIGHT"."active" AS "active.y"
##   FROM (SELECT "rental_id", "rental_date", "inventory_id", "customer_id", "return_date", "staff_id", "last_update.x", "first_name", "last_name", "address_id", "email" AS "staff_email", "store_id", "active", "username", "password", "last_update.y", "picture"
## FROM (SELECT "TBL_LEFT"."rental_id" AS "rental_id", "TBL_LEFT"."rental_date" AS "rental_date", "TBL_LEFT"."inventory_id" AS "inventory_id", "TBL_LEFT"."customer_id" AS "customer_id", "TBL_LEFT"."return_date" AS "return_date", "TBL_LEFT"."staff_id" AS "staff_id", "TBL_LEFT"."last_update" AS "last_update.x", "TBL_RIGHT"."first_name" AS "first_name", "TBL_RIGHT"."last_name" AS "last_name", "TBL_RIGHT"."address_id" AS "address_id", "TBL_RIGHT"."email" AS "email", "TBL_RIGHT"."store_id" AS "store_id", "TBL_RIGHT"."active" AS "active", "TBL_RIGHT"."username" AS "username", "TBL_RIGHT"."password" AS "password", "TBL_RIGHT"."last_update" AS "last_update.y", "TBL_RIGHT"."picture" AS "picture"
##   FROM "rental" AS "TBL_LEFT"
##   LEFT JOIN "staff" AS "TBL_RIGHT"
##   ON ("TBL_LEFT"."staff_id" = "TBL_RIGHT"."staff_id")
## ) "tvnvuviyiw") "TBL_LEFT"
##   LEFT JOIN "customer" AS "TBL_RIGHT"
##   ON ("TBL_LEFT"."customer_id" = "TBL_RIGHT"."customer_id")
## ) "dkimtwhtoo") "dkadgsqpgd"
```

Here is the SQL query formatted for readability:
```
SELECT "rental_date", 
       "staff_email", 
       "customer_email" 
FROM   (SELECT "rental_id", 
               "rental_date", 
               "inventory_id", 
               "customer_id", 
               "return_date", 
               "staff_id", 
               "last_update.x", 
               "first_name.x", 
               "last_name.x", 
               "address_id.x", 
               "staff_email", 
               "store_id.x", 
               "active.x", 
               "username", 
               "password", 
               "last_update.y", 
               "picture", 
               "store_id.y", 
               "first_name.y", 
               "last_name.y", 
               "email" AS "customer_email", 
               "address_id.y", 
               "activebool", 
               "create_date", 
               "last_update", 
               "active.y" 
        FROM   (SELECT "TBL_LEFT"."rental_id"     AS "rental_id", 
                       "TBL_LEFT"."rental_date"   AS "rental_date", 
                       "TBL_LEFT"."inventory_id"  AS "inventory_id", 
                       "TBL_LEFT"."customer_id"   AS "customer_id", 
                       "TBL_LEFT"."return_date"   AS "return_date", 
                       "TBL_LEFT"."staff_id"      AS "staff_id", 
                       "TBL_LEFT"."last_update.x" AS "last_update.x", 
                       "TBL_LEFT"."first_name"    AS "first_name.x", 
                       "TBL_LEFT"."last_name"     AS "last_name.x", 
                       "TBL_LEFT"."address_id"    AS "address_id.x", 
                       "TBL_LEFT"."staff_email"   AS "staff_email", 
                       "TBL_LEFT"."store_id"      AS "store_id.x", 
                       "TBL_LEFT"."active"        AS "active.x", 
                       "TBL_LEFT"."username"      AS "username", 
                       "TBL_LEFT"."password"      AS "password", 
                       "TBL_LEFT"."last_update.y" AS "last_update.y", 
                       "TBL_LEFT"."picture"       AS "picture", 
                       "TBL_RIGHT"."store_id"     AS "store_id.y", 
                       "TBL_RIGHT"."first_name"   AS "first_name.y", 
                       "TBL_RIGHT"."last_name"    AS "last_name.y", 
                       "TBL_RIGHT"."email"        AS "email", 
                       "TBL_RIGHT"."address_id"   AS "address_id.y", 
                       "TBL_RIGHT"."activebool"   AS "activebool", 
                       "TBL_RIGHT"."create_date"  AS "create_date", 
                       "TBL_RIGHT"."last_update"  AS "last_update", 
                       "TBL_RIGHT"."active"       AS "active.y" 
                FROM   (SELECT "rental_id", 
                               "rental_date", 
                               "inventory_id", 
                               "customer_id", 
                               "return_date", 
                               "staff_id", 
                               "last_update.x", 
                               "first_name", 
                               "last_name", 
                               "address_id", 
                               "email" AS "staff_email", 
                               "store_id", 
                               "active", 
                               "username", 
                               "password", 
                               "last_update.y", 
                               "picture" 
                        FROM   (SELECT "TBL_LEFT"."rental_id"    AS "rental_id", 
                                       "TBL_LEFT"."rental_date"  AS 
                                       "rental_date", 
                                       "TBL_LEFT"."inventory_id" AS 
                                       "inventory_id", 
                                       "TBL_LEFT"."customer_id"  AS 
                                       "customer_id", 
                                       "TBL_LEFT"."return_date"  AS 
                                       "return_date", 
                                       "TBL_LEFT"."staff_id"     AS "staff_id", 
                                       "TBL_LEFT"."last_update"  AS 
                                       "last_update.x", 
                                       "TBL_RIGHT"."first_name"  AS "first_name" 
                                       , 
                       "TBL_RIGHT"."last_name"   AS "last_name", 
                       "TBL_RIGHT"."address_id"  AS "address_id", 
                       "TBL_RIGHT"."email"       AS "email", 
                       "TBL_RIGHT"."store_id"    AS "store_id", 
                       "TBL_RIGHT"."active"      AS "active", 
                       "TBL_RIGHT"."username"    AS "username", 
                       "TBL_RIGHT"."password"    AS "password", 
                       "TBL_RIGHT"."last_update" AS "last_update.y", 
                       "TBL_RIGHT"."picture"     AS "picture" 
                                FROM   "rental" AS "TBL_LEFT" 
                                       LEFT JOIN "staff" AS "TBL_RIGHT" 
                                              ON ( "TBL_LEFT"."staff_id" = 
                                                   "TBL_RIGHT"."staff_id" )) 
                               "ymdofxkiex") "TBL_LEFT" 
                       LEFT JOIN "customer" AS "TBL_RIGHT" 
                              ON ( "TBL_LEFT"."customer_id" = 
                                 "TBL_RIGHT"."customer_id" )) 
               "exddcnhait") "aohfdiedlb" 
```

Hand-written SQL code to do the same job will probably look a lot nicer and could be more efficient, but functionally dplyr does the job.


```r
GQ <- dbGetQuery(
  con,
  "select r.rental_date, s.email staff_email,c.email customer_email  
     from rental r
          left outer join staff s on r.staff_id = s.staff_id
          left outer join customer c on r.customer_id = c.customer_id
  "
)
```

But because `Q` hasn't been executed, we can add to it.  This behavior is the basis for a useful debugging and development process where queries are built up incrementally.

Where you place the `collect` function matters.

```r
dbDisconnect(con)
sp_docker_stop("sql-pet")
```
