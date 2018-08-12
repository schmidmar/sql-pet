SQL Pet
=======

# Goals

The use case for this repo is:

* You are running R through Rstudio and want to experiment with some of the intricacies of working with an SQL database that has:
    + a moderately complex and unfamiliar structure. 
    + requires passwords and other features found in an organizational environment
    + mostly read but sometimes write to the database

    This example use [the Postgres version of "dvd rental" database](http://www.postgresqltutorial.com/postgresql-sample-database/), which can be  [downloaded here](http://www.postgresqltutorial.com/wp-content/uploads/2017/10/dvdrental.zip).  Here's a glimpse of it's structure:
    
    ![Entity Relationship diagram for the dvdrental database](fig/dvdrental-er-diagram.png)

* You want to run PostgresSQL on a Docker container, avoiding any OS or system dependencies  that might come up. 

# Instructions

## Download the repo

First step: download [this repo](https://github.com/smithjd/sql-pet).  It contains source code to build a Docker container that has the dvdrental database in Postgress and shows how to interact with the database from R.

## Docker & Postgres

Noam Ross's "[Docker for the UseR](https://nyhackr.blob.core.windows.net/presentations/Docker-for-the-UseR_Noam-Ross.pdf)" suggests the following use-cases for useRs:

* Make a fixed working environment
* Access a service outside of R **(e.g., Postgres)**
* Create an R based service
* Send our compute job somewhere else

There's a lot to learn about Docker and many uses for it, here we just cut to the chase. (Later you might come back to study this [ROpensci Docker tutorial](https://ropenscilabs.github.io/r-docker-tutorial/))

* Install Docker.  

  + [On a Mac](https://docs.docker.com/docker-for-mac/install/)
  + [On Windows](https://docs.docker.com/docker-for-windows/install/)
  + [On UNIX flavors](https://docs.docker.com/install/#supported-platforms)
  

* Verify that it's running on your machine with

     `$ docker -v`

* From the directory that contains this repo, open a terminal, run `docker-compose`. Your container will be named `sql-pet_postgres9_1`: 

     `$ docker-compose up`

* Use `test_postgres.Rmd` to demonstrate that you have a persistent database by uploading `mtcars` to Postgres, then stopping the Docker container, restarting it, and finally determining that `mtcars` is still there.

* In another terminal session, use the `stop` command to stop the container (and the Postgres database).  You should get a 0 return code.  If you forgot to disconnect from Postgres in R, you will get a 137 return code.

    `$ docker-compose stop`

* When you have time explore the Postgres environment by browsing around inside the Docker command with a shell

    `$ docker exec -ti sql-pet_postgres9_1 sh`

  + To exit Docker enter:

    `# exit`

  + Inside Docker, you can enter the Postgres command-line utility psql by entering 

    `# psql -U postgres`

    Handy commands inside psql include:

    + `postgres=# \h`          # psql help
    + `postgres=# \dt`         # list Postgres tables
    + `postgres=# \c dbname`   # connect to databse dbname
    + `postgres=# \l`          # list Postgres databases
    + `postgres=# \conninfo`   # list Postgres databases
    + `postgres=# \q`          # exit psql


## DVD Rental database installation

* Download the backup file for the dvdrental test database and convert it to a .tar file with:

   [./src/get-dvdrental-zipfile.Rmd](./src/get-dvdrental-zipfile.Rmd)

* Create the dvdrental database in Postgres and restore the data in the .tar file with:

   [./src/install-dvdrental-in-postgres.Rmd](./src/install-dvdrental-in-postgres.Rmd)

## Interacting with Postgres from R

* passwords
* overview investigation
* dplyr queries
* what goes on the database side vs what goes on the R side
* examining dplyr queries
* rewriting SQL
* performance considerations

# Resources

* Picking up ideas and tips from Ed Borasky's [Data Science pet containers]( https://github.com/hackoregon/data-science-pet-containers).  This repo creates a framework based on that Hack Oregon example.
* A very good [introductory Docker tutorial](https://docker-curriculum.com/)
* Usage examples of [Postgres with Docker](https://amattn.com/p/tutorial_postgresql_usage_examples_with_docker.html)
* Loading the [dvdrental database into Postgres](http://www.postgresqltutorial.com/load-postgresql-sample-database/)
