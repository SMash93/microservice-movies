# Microservices with React and Node

This post focuses on setting up a React app to communicate with a number of Node microservices.

ADD INTRO
ADD IMAGE
ADD LINK TO PREVIOUS BLOG POST
UPDATE WORKING DIR

This post assumes prior knowledge of the following topics. Refer to the resources for more info:

| Topic            | Resource |
|------------------|----------|
| Docker           | [Get started with Docker](https://docs.docker.com/engine/getstarted/) |
| Docker Compose   | [Get started with Docker Compose](https://docs.docker.com/compose/gettingstarted/) |
| Node/Express API | [Testing Node and Express](http://mherman.org/blog/2016/09/12/testing-node-and-express) |
| React | [React Intro](https://github.com/mjhea0/node-workshop/blob/master/w2/lessons/03-react.md)
| TestCafe | [Developing and Testing Microservices with Docker](http://mherman.org/blog/2017/04/18/developing-and-testing-microservices-with-docker)

## Contents

1. Objectives
1. Architecture
1. Project Setup
1. Docker Config
1. User Service
1. Web Service - part 1
1. Movies Service
1. Web Service - part 2
1. Workflow
1. Test Setup
1. Swagger Setup
1. Next Steps

## Objectives

ADD OBJECTIVES

By the end of this tutorial, you should be able to...

1. Configure and run a set of microservices locally with Docker and Docker Compose
1. Utilize [volumes](https://docs.docker.com/engine/tutorials/dockervolumes/) to mount your code into a container
1. Run unit and integration tests inside a Docker container
1. Set up a separate container for functional tests
1. Debug a running Docker container
1. Utilize [links](https://docs.docker.com/compose/compose-file/#links) for inter-container communication (AJAX)
1. Secure your services via JWT-based authentication

## Architecture

| Name             | Service | Container | Tech          |
|------------------|---------|-----------|---------------|
| Web              | Web     | web       | React, Redux  |
| Movies API       | Movies  | movie     | Node, Express |
| Movies DB        | Movies  | movie-db  | Postgres      |
| Swagger          | Movies  | swagger   | Swagger UI    |
| User API         | User    | user      | Node, Express |
| User DB          | User    | user-db   | Postgres      |
| Functional Tests | Test    | test      | TestCafe      |

In a microservice architecture, units are less complex while the system is more complex.

## Project Setup

ADD SETUP INSTRUCTIONS - clone, view project tree, current set up info

Before we added Docker, feel free to test each service...

*Users:*

- Navigate to "services/users"
- `npm install`
- `npm start`
- Open [http://localhost:3000/users/ping](http://localhost:3000/users/ping) in your browser

*Web:*

- Navigate to "services/web"
- `npm install`
- `npm start`
- Open [http://localhost:3006](http://localhost:3006) in your browser. You should see the log in page.

## Docker Config

Add a *docker-compose.yml* file to the project root. This file is used by Docker Compose to link multiple services together.

Then add a *.dockerignore* to the "services/movies", "services/movies/src/db", "services/movies/src/swagger", "services/users", "services/users/src/db", "services/web", "tests" directories:

```
.git
.gitignore
README.md
docker-compose.yml
node_modules
```

With that, let's get each service going, making sure to test as we go...

## Users Service

We'll start with the database since the API is dependent on it being up...

### Database

First, add a *Dockerfile* to "services/users/src/db":

```
FROM postgres

# run create.sql on init
ADD create.sql /docker-entrypoint-initdb.d
```

Then update the *docker-compose.yml* file:

```
version: '2.1'

services:

  users-db:
    container_name: users-db
    build: ./services/users/src/db
    ports:
      - '5433:5432' # expose ports - HOST:CONTAINER
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
    healthcheck:
      test: exit 0
```

This config will create a container called `movies-db`, from the *Dockerfile* found in "services/users/src/db". Once spun up, environment variables will be added and an exit code of `0` will be sent after it's successfully up and running. Postgres will be available on port `5433` on the host machine and on port `5432` for other services.

> **NOTE:** Use `expose` if you just want Postgres available to other services but not the host machine:
> ```
> expose:
    - "5432"
> ```

Fire up the container:

```sh
$ docker-compose up --build -d
```

Once up, let's run a quick sanity check. Enter the shell:

```sh
$ docker-compose run users-db bash
```

Then run `env` to ensure that the proper environment variables are set. `exit` when done.

### API

Turning to the API, add a *Dockerfile* to "services/users", making sure to review the comments:

```
FROM node:latest

# set working directory
RUN mkdir /src
WORKDIR /src

# install app dependencies
ENV PATH /src/node_modules/.bin:$PATH
ADD package.json /src/package.json
RUN npm install

# start app
CMD ["npm", "start"]
```

Then add the `users-service` to the *docker-compose.yml* file:

```
users-service:
  container_name: users-service
  build: ./services/users/
  volumes:
    - './services/users:/src'
    - './services/users/package.json:/src/package.json'
  ports:
    - '3000:3000' # expose ports - HOST:CONTAINER
  environment:
    - DATABASE_URL=postgres://postgres:postgres@users-db:5432/users_dev
    - DATABASE_TEST_URL=postgres://postgres:postgres@users-db:5432/users_test
    - NODE_ENV=${NODE_ENV}
    - TOKEN_SECRET=changeme
  depends_on:
    users-db:
      condition: service_healthy
  links:
    - users-db
```

What's happening here?

- `volumes`: [volumes](https://docs.docker.com/engine/tutorials/dockervolumes/) are used to mount a directory inside a container so that you can make modify the code without having to rebuild the image. This should be a default in your local development environment so you can get quick feedback on code changes.
- `depends_on`: [depends_on](https://docs.docker.com/compose/compose-file/#dependson) specifies the order in which to start services. In this case, the `users-service` will wait for the `users-db` to fire up successfully (with an exit code of `0`) before it starts.
- `links`: With [links](https://docs.docker.com/compose/compose-file/#links) you can link to services running in other containers. So, with this config, code inside the `users-service` will be able to access the database via `users-db:5432`.

> **NOTE:** Curious about the difference between `depends_on` and `links`? Check out the following [Stack Overflow discussion](http://stackoverflow.com/a/39658359/1799408) for more info.

Set the `NODE_ENV` environment variable:

```sh
$ export NODE_ENV=development
```

Build the image and spin up the container:

```sh
$ docker-compose up --build -d users-service
```

Once up, create a new file in the project root called *init_db.sh* and add the Knex migrate and seed commands:

```sh
#!/bin/sh

docker-compose run users-service knex migrate:latest --env development --knexfile app/knexfile.js
docker-compose run users-service knex seed:run --env development --knexfile app/knexfile.js
```

Then apply the migrations and add the seed:

```sh
$ sh init_db.sh
Using environment: development
Batch 1 run: 1 migrations
/src/src/db/migrations/20170504191016_users.js
Using environment: development
Ran 1 seed files
/src/src/db/seeds/users.js
```

Test:

| Endpoint        | HTTP Method | CRUD Method | Result        |
|-----------------|-------------|-------------|---------------|
| /users/ping     | GET         | READ        | `pong`        |
| /users/register | POST        | CREATE      | add a user    |
| /users/login    | POST        | CREATE      | log in a user |
| /users/user     | GET         | READ        | get user info |

```sh
$ http POST http://localhost:3000/users/register username=michael password=herman
$ http POST http://localhost:3000/users/login username=michael password=herman
```

> **NOTE:** `http` in the above commands is part of the [HTTPie](https://httpie.org/) library, which is a wrapper on top of cURL.

Finally, run the unit and integration tests:

```sh
$ docker-compose run users-service npm test
```

You should see:

```
routes : index
  GET /does/not/exist
    ✓ should throw an error

routes : users
  POST /users/register
    ✓ should register a new user (178ms)
  POST /users/login
    ✓ should login a user (116ms)
    ✓ should not login an unregistered user
    ✓ should not login a valid user with incorrect password (125ms)
  GET /users/user
    ✓ should return a success (114ms)
    ✓ should throw an error if a user is not logged in

auth : helpers
  comparePass()
    ✓ should return true if the password is correct (354ms)
    ✓ should return false if the password is correct (315ms)
    ✓ should return false if the password empty (305ms)

auth : local
  encodeToken()
    ✓ should return a token
  decodeToken()
    ✓ should return a payload


12 passing (4s)
```

Check the test specs for more info. That's it! Let's move on to the web service...

## Web Service - part 1

With our users service up and running, we can turn our attention to the client-side and spin up the React app inside a container to test authentication.

> **NOTE:** The React code is ported from [intro-react-redux-omdb](https://github.com/blackstc/intro-react-redux-omdb) and [communikey](https://github.com/etmoore/communikey) written by [Charlie Blackstock](https://www.linkedin.com/in/charlieblackstock/) and [Evan Moore](https://www.linkedin.com/in/etmoore1/), respectively - two of my former students.

Add a *Dockerfile* to "services/web":

```
FROM node:latest

# set working directory
RUN mkdir /src
WORKDIR /src

# install app dependencies
ENV PATH /src/node_modules/.bin:$PATH
ADD package.json /src/package.json
RUN npm install

# start app
CMD ["npm", "start"]
```

And update the *docker-compose.yml* file like so:

```
web-service:
  container_name: web-service
  build: ./services/web/
  volumes:
    - './services/web:/src'
    - './services/web/package.json:/src/package.json'
  ports:
    - '3007:3006' # expose ports - HOST:CONTAINER
  environment:
    - NODE_ENV=${NODE_ENV}
  depends_on:
    users-service:
      condition: service_started
  links:
    - users-service
```

Build the image and fire up the container:

```sh
$ docker-compose up --build -d web-service
```

Open your browser and navigate to [http://localhost:3007](http://localhost:3007). You should see the login page:

ADD IMAGE

Log in with -

- username: `michael`
- password: `herman`

Once logged in you should see:

ADD IMAGE

ADD BIT ABOUT AJAX CALL

Next, let's spin up the movies service so that end users can save movies to a collection...

## Movies Service

Set up for the movies service is nearly the same as the users service. Try this on your own to check your understanding:

1. Database
    - add a *Dockerfile*
    - update the *docker-compose.yml*
    - spin up the container
    - test
1. API
    - add a *Dockerfile*
    - update the *docker-compose.yml*
    - spin up the container
    - apply migrations and seeds
    - test

GULP ISSUES?

> **NOTE:** Need help? Grab the code from the [microservices-movies](ADD URL) repo.

With the container up, let's test out the endpoints...

| Endpoint      | HTTP Method | CRUD Method | Result                    |
|---------------|-------------|-------------|---------------------------|
| /movies/ping  | GET         | READ        | `pong`                    |
| /movies/user  | GET         | READ        | get all movies by user    |
| /movies       | POST        | CREATE      | add a single movie       |

Start with opening the browser to [http://localhost:3001/movies/ping](http://localhost:3001/movies/ping). You should see `pong`! Try [http://localhost:3001/movies/user](http://localhost:3001/movies/user):

```json
{
  "status": "Please log in"
}
```

Since you need to be authenticated to access the other routes, let's test them out by running the integration tests:

```sh
$ docker-compose run movies-service npm test
```

You should see:

```sh
routes : index
  GET /does/not/exist
    ✓ should throw an error

Movies API Routes
  GET /movies/ping
    ✓ should return "pong"
  GET /movies/user
    ✓ should return saved movies
  POST /movies
    ✓ should create a new movie


4 passing (818ms)
```

Check the test specs for more info.

RACE CONDITION NPM TEST, TABLE LOCK?
ADD HOW TO GET AROUND AUTH?

## Web Service - part 2

Turn to the *docker-compose.yml* file. Update the `links` and `depends_on` keys for the `web-service`:

```
depends_on:
  users-service:
    condition: service_started
  movies-service:
    condition: service_started
links:
  - users-service
  - movies-service
```

Why?

Next, update the container:

```sh
$ docker-compose up -d web-service
```

Let's test this out in the browser! Open [http://localhost:3007/](http://localhost:3007/). Log in and then add some movies to the collection. Be sure to view the collection as well.

ADD IMAGES
ADD INFO ABOUT AJAX REQUEST: Take a look at the AJAX request in the GET / route in web/src/routes/index.js. Why does the uri point to locations-service and not localhost? Well, localhost refers back to the container itself, so you need to set up a link in the Docker compose - which we’ve already done.

## Workflow

how to work with create react app?

## Test Setup

Thus far we've only tested each individual microservice with unit and integration tests. Let's turn our attention to functional, e2e tests to test the entire system. For this, we'll use [TestCafe](https://testcafe.devexpress.com/).

Again, add a *Dockerfile* to "tests":

```sh
```

## Swagger Setup


## Next Steps

SOURCES:

https://medium.com/@patriciolpezjuri/using-create-react-app-with-react-router-express-js-8fa658bf892d
https://github.com/fabiopaiva/docker-create-react-app/blob/master/hello-world/package.json
http://blog.cloud66.com/microservices-with-node-js/
