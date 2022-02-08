---
title: Building a REST and Web Socket API with Actix and Rust
date: 2021-03-21 00:00:00
layout: post
---

The world of web development really has come a long way over the years. In late 90s to early 2000s I learned off various websites how to build web pages with HTML, tables, random JavaScript snippets, etc. Over time we got more sophisiticated server rendering options like asp, php, and then into MVC frame works like Rails and Django. Now we're writing the backend side as full on REST apis, where all the server does is return data, and the client uses that data to populate the interface. The interface is built with a variety of technologies, one can still use tech like I did 6-7 years ago: server rendered pages in Rails and writing jQuery code to make things more dynamic. Which can work really well for a lot of applications, but when you need more dynamic control of a page, it becomes less scalable. This is where the modern frameworks like React & Angular come in.

"Why are you going on about this" you ask. "This post is about Actix & Rust!" Good question, reader! I bring this wonderful history of web development up because I feel like websockets are a really powerful tool. I don't think they're needed for every case out there, I imagine we run into them on all sorts of applications we use day to day. That said it's not a technology that I've leveraged very much in my career working at various agencies. There are a number of frameworks one can dig into to leverage web sockets. I have done so via [Primus](https://github.com/primus/primus) in the past. [ActionHeroJS](https://www.actionherojs.com/) is a NodeJS API framework that offers relatively seamless handling of HTTP & Websocket requests. Recently when working on a side project of mine I wanted to leverage both actix for its HTTP API capabitilies and integrate websockets for updating the user on state changes.

The application that I ended up building is one for users making predictions for a particular match or game. This in particular is for a StarCraft meetup I would go to, where we would have a raffle, and each game we would pick before hand who would be the one to win, as well as hit other first milestones in the game. For a sport you could equate this to first to score a goal, or first team to have a strikeout in baseball, etc. The idea was to model this in a similar way to the popular Jackbox games where users can easily join a game on their phone, select their picks and see a leaderboard. The benefits of web sockets for updating state of pick selection, the scoring, etc, came pretty clear to me.

That application has a bit more pieces to it than make sense for a tutorial like this. Instead I'm taking the lessons I learned while building that project, and we're going to build a piece of it. The user can view a list of questions, as well as create them. When questions are created, users connected to the websocket will recieve new questions. Relatively simple, but it covers a lot of the same concepts.

**Note**

This post expects that you have worked with Rust before, or at least have some working knowledge, and that you have worked on backend web APIs before. As I don't really dig into those concepts here. If Rust interests you, I recommend checking out the official book: [https://doc.rust-lang.org/book/](https://doc.rust-lang.org/book/). This tutorial on implementing linked lists is also fun [https://rust-unofficial.github.io/too-many-lists/index.html](https://rust-unofficial.github.io/too-many-lists/index.html), and does touch on some beginner concepts in Rust.

Secondly I've written this in such a way that the code doesn't always contain the full snippet. All the code you need will be written here, but when re-visiting a file, I don't show all the existing code that is there. I make comments in the code blocks to know when to remove things. I also make notes in both the code comments, and outside when to make additions. I have also added code comments explaining some Rust things, as well as how actix injects certain parameters, or how certain traits apply to the code.

**Actix versio 4 update**

This post was written for version 3, but version 4 has now hit RC. The post itself is not updated, but the completed app has been. You can see the changes I made to update the app here: [https://github.com/agmcleod/questions-app-rust-actix/commit/41784f9d99ab51113ca0de98a198eb9fa8ff83da](https://github.com/agmcleod/questions-app-rust-actix/commit/41784f9d99ab51113ca0de98a198eb9fa8ff83da)

## Let's get into the setup

First off here is the current version of the full server for my application: [https://github.com/agmcleod/sc-predictions-server](https://github.com/agmcleod/sc-predictions-server). It is not open source, but I'm happy for folks to use it a reference in their own projects and hobbies.

Secondly I want to give a huge shout out to this repository here: [https://github.com/ddimaria/rust-actix-example](https://github.com/ddimaria/rust-actix-example), which was instrumental in helping me figure out some of the authentication pieces, as well as error handling.

First create a new folder `questions-app`.

For building this application, I'm using docker for running the database, and then compiling & running the rust code on my host machine. If you use Linux, you can use docker to run the rust code. On windows & mac, I found the compilation time was about 20 times slower. Building in docker would take minutes, where as outside of docker it was more like 10 seconds. However if you run the application on your host machine, you will need postgres development tools installed on your machine.

If you wish to do the same, setup a `questions-app/docker-compose.yml` file like so:

```yaml
version: "3"

services:
  database:
    image: "postgres:10.5"
    ports:
      - "5434:5432"
    environment:
      POSTGRES_DB: my_database
      POSTGRES_PASSWORD: my_password
```

I'd suggest to use a newer version of postgres if you can, I first started on this project a few years ago. The port mapping is up to you as well, I set it to a different port as the installation of the postgres developer tools in windows starts a postgres server. I found this easier to avoid port collision.

Then create the database container and let it run in the background for us:

```bash
docker-compose up -d
```

You can stop this process whenever by running

```
docker-compose stop
```

The postgres base image for docker creates the database for us, so we should be good to go.

For the application server, we will split it into a few crates. Create a file called `Cargo.toml`, and let's populate it with our workspace information.

```
[workspace]

members = ["server"]
```

You can read more about [Cargo workspaces here](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html), but the idea is that we're stating a set of related binaries or libraries together. I do wish one could define a set of shared dependencies between them, but ah well!

Let's start on the server application side of things.

```
cargo new --bin server
```

Open `server/Cargo.toml` and add some dependencies to it:

```
[dependencies]
actix = "0.10.0"
actix-cors = "0.5"
actix-http = "2.1"
actix-identity = "0.3"
actix-service = "1.0.6"
actix-web = "3.2"
actix-web-actors = "3.0"
actix-rt = "1.1"
dotenv = "0.9.0"
env_logger = "0.8.2"
futures = "0.3.5"
futures-util = "0.3.5"
log = "0.4.0"
```

We won't need all of these right away, but we are specifying the base actix crates we need for this application, futures for working with async code. Both in the application and in tests. Handling environment variables, so we don't need to have keys and host names hardcoded. And some good ol fashioned logging.

Let's set things up in `server/src/main.rs`.

```rust
#[macro_use]
extern crate log;

use std::env;

use actix_cors::Cors;
use actix_rt;
use actix_web::{http, middleware::Logger, App, HttpServer};
use dotenv::dotenv;
use env_logger;

#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("debug"));

    // TBD
}
```

The first couple lines of code read the environment variables out of the .env file, as well as bash environment variables... which we haven't setup! Usually with docker I will pass environment variables through the compose file, but because the server is running on the host machine, i use a Makefile. If you're on windows, you'll need to have make installed first.

```Makefile
SHELL := /bin/bash

db_url := postgres://postgres:my_password@localhost:5434/my_database

run_server:
	DATABASE_URL=$(db_url) \
		RUST_BACKTRACE=full \
		cargo run --bin server

.PHONY: run_server
```

So now when we run `make run_server`, it will run with the variables we need. I snuck in a `RUST_BACKTRACE` so we can follow error stacks more easily. With that setup, we add the following to the rest of main()

```rust
#[actix_rt::main]
async fn main() -> std::io::Result<()> {
    dotenv().ok();
    env_logger::init_from_env(env_logger::Env::default().default_filter_or("debug"));
    HttpServer::new(move || {
        let cors = Cors::default()
            .allowed_origin(&env::var("CLIENT_HOST").unwrap())
            .allow_any_method()
            .allowed_headers(vec![
                http::header::AUTHORIZATION,
                http::header::ACCEPT,
                http::header::CONTENT_TYPE,
            ])
            .max_age(3600);

        App::new()
            .wrap(cors)
            .wrap(Logger::default())
            .wrap(Logger::new("%a %{User-Agent}i"))
    })
    .bind("0.0.0.0:8080")?
    .run()
    .await
}
```

We construct the web server, passing a number of services to it. First we allow CORS for our frontend's host only, and limit the headers allowed. Then we create the application server, setting up the logging, but no routes or middleware yet. Finally the server is started on port 8080.

Running `make run_server` should now compile and run the actix web server. It'll take sometime to compile and build all the dependencies for the first go. Once done, open a brower and visit [http://localhost:8080](http://localhost:8080) to verify it. You should see a blank page, and a 404 http response.

## Getting all questions

Now we need to start building out a GET endpoint that returns our list of questions.

Create our db crate:

```bash
cargo new --lib db
```

As the output suggests, add it to our workspace Cargo.toml file.

```
[workspace]

members = ["db", "server"]
```

For our database models, we're going to use [diesel](https://diesel.rs). As well as r2d2 for connection pooling, chrono for datetime, and serde for serializing the model into JSON.

Add the following to `db/Cargo.toml`

```
chrono = { version = "0.4.6", features = ["serde"] }
diesel = { version = "1.4.4", features = ["postgres", "r2d2", "uuid", "chrono"] }
env_logger = "0.5.13"
log = "0.4.0"
r2d2 = "0.8.2"
r2d2_postgres = "0.14.0"
serde = "1.0.80"
serde_derive = "1.0.115"
serde_json = "1.0.13"
```

Install diesel-cli on your host machine:

```
cargo install diesel_cli --no-default-features --features postgres
```

Then run the setup command, which creates our initial migration.

```
DATABASE_URL=postgres://postgres:my_password@localhost:5434/my_database diesel setup --migration-dir=db/migrations
```

Let's create a migration to create our questions table. Add a few usefule commands to our Makefile

```Makefile
create_migration:
	DATABASE_URL=$(db_url) diesel migration generate $(name) --migration-dir=db/migrations

migrate:
	DATABASE_URL=$(db_url) diesel migration run --migration-dir=db/migrations

redo_migrate:
	DATABASE_URL=$(db_url) diesel migration redo --migration-dir=db/migrations
```

Now create the migration

```bash
make create_migration name=create_questions
```

Diesel migrations come as a up.sql and down.sql. The former runs when you want to apply the change, the latter when you want to undo the change. Open the up.yml file that was just created, and populate it with:


```sql
CREATE TABLE questions (
  id SERIAL PRIMARY KEY,
  body TEXT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

SELECT diesel_manage_updated_at('questions');
```

The `diesel_manage_updated_at` function applies a trigger to ensure updated_at is populated for us whenever the record gets changed. Now populate the down.yml file with the drop table statement:

```sql
DROP TABLE questions;
```

Before running the migrations, open the `diesel.toml` file, and change the contents to:

```toml
# For documentation on how to configure this file,
# see diesel.rs/guides/configuring-diesel-cli

[print_schema]
file = "db/src/schema.rs"
```

This specifies the location we wish to use for the generated schema.rs file.

Run the migration!

```bash
make migrate
```

Upon completion, the db/src/schema.rs file will have been created. Do not edit this file, as diesel uses it to manage the state of the database schema.

To setup our database model, create some new files:

```bash
mkdir -p db/src/models
touch db/src/models/mod.rs
touch db/src/models/question.rs
```

Open `db/src/lib.rs` and initialize the modules we will be creating, as well as setup a type for our pool:

```rust
use std::env;

use diesel::pg::PgConnection;
use diesel::r2d2::{ConnectionManager, Pool, PooledConnection};

pub mod models;
pub mod schema;

pub type PgPool = Pool<ConnectionManager<PgConnection>>;

pub fn get_conn(pool: &PgPool) -> PooledConnection<ConnectionManager<PgConnection>> {
    pool.get().unwrap()
}

pub fn new_pool() -> PgPool {
    // again using our environment variable
    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let manager = ConnectionManager::<PgConnection>::new(database_url);

    Pool::builder()
        .build(manager)
        .expect("failed to create db pool")
}
```

For getting a connection, calling unwrap() right away isn't the greatest option, but we'll improve it later.

We define a re-usable type `PgPool` so we don't need to import those types in each of our route handlers. From there the new_pool() function uses the environment variable to establish a connection manager, and creates an r2d2 pool for managing connections.

Open `db/src/models/mod.rs` to define the question module.

```rust
mod question;

pub use self::question::*;
```

Defining a module this way, and then calling `pub use` allows us to re-export the definitions in the question module from the models module. In the app we will be able to import with `use db::models::Question`. Instead of `use db::models::question::Question`.

Open the model file and let's setup our model. First add the imports we will need:

```rust
// question.rs
use chrono::{DateTime, Utc};
use diesel::{PgConnection, QueryDsl, RunQueryDsl};
use serde::{Deserialize, Serialize};

use crate::schema::questions;
```

You may see a compiler error in reference to `crate::schema`, we'll fix that shortly!

Create the struct, deriving the types from diesel so it can support the type of queries we need & serialization.

```rust
#[derive(Debug, Identifiable, Serialize, Deserialize, Queryable)]
pub struct Question {
    pub id: i32,
    pub body: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}
```

An important thing to note, the fields here in the struct need to match the order they are defined in the schema.rs file. This will also match the order they were defined in the migration file. So if you were to remove `body` and add it back in a new migration, you would need to move `body` in the struct to the bottom. Probably the one thing that annoys me about diesel, but I realize given the trickyness of making this work, there's not an easy answer.

Next add a function to Question struct to get us all the records. First update `db/src/lib.rs` by adding a macro_use. This is so we can pull data out of schema module. Diesel will automatically load the schema.rs file, and this allows it to have the macros used in that file defined in our project scope.

```rust
// db/src/lib.rs
#[macro_use]
extern crate diesel;
```

Then open question.rs again

```rust
// Update the use list from before
use diesel::{PgConnection, QueryDsl, Queryable, RunQueryDsl};

impl Question {
    pub fn get_all(conn: &PgConnection) -> Result<Vec<Question>, diesel::result::Error> {
        use crate::schema::questions::dsl::{body, questions};

        let all_questions = questions.order(body).load::<Question>(conn)?;

        Ok(all_questions)
    }
}
```

The code here pulls in the dsl for querying the questions table, and the body field so we can sort by it. Using the ? operator to immediately return a diesel error if one ocurred. When things go well, we wrap the `Vec<Question>` in an Ok result.

Okay, so let's setup a route now to return questions. First add our db crate to the server's Cargo.toml

```
db = { path = "../db" }
```

```
mkdir -p server/src/routes/
touch server/src/routes/mod.rs
mkdir -p server/src/routes/questions
touch server/src/routes/questions/mod.rs
touch server/src/routes/questions/get_all.rs
```

Whew so many files! You can put everything in routes/mod.rs if you like, but I prefer to keep them separate so we don't crowd our route files.

Add our module definitions

```rust
// server/src/main.rs

mod routes;
```

```rust
// server/src/routes/mod.rs

pub mod questions;
```

```rust
// server/src/routes/questions/mod.rs

mod get_all;

// re-export everything under get_all to be part of questions
pub use self::get_all::*;
```

Open get_all.rs and let's create a route to return all questions in the database.

```rust
use actix_web::{
    error::Error,
    web::{block, Data, Json},
    Result,
};

use db::{get_conn, models::Question, PgPool};

pub async fn get_all(pool: Data<PgPool>) -> Result<Json<Vec<Question>>, Error> {
    let connection = get_conn(&pool);

    let questions = block(move || Question::get_all(&connection)).await?;

    Ok(Json(questions))
}
```

This brings together everything we've setup so far. We add the dependencies, and setup our route handler. It's important to note that the `pool: Data<PgPool>` argument works because of how actix allows app state & data to be shared. Coming up we will pass an instance of our pool to `App`, which allows it be accessed as parameters in our route handler functions. You can read more about this here [https://docs.rs/actix-web/3.3.2/actix_web/web/struct.Data.html](https://docs.rs/actix-web/3.3.2/actix_web/web/struct.Data.html)

It uses the Question model to get all records. Since we applied serde Serialize to the Question model, it can be easily converted into JSON. For our error type here we are just using the generic actix error type. The reason for block() is so that the database operation, which is IO blocking functionality, happens on a separate thread. block returns a future meaning we can use `await` to wait for it to complete before returning a response. Creating an Ok result, and wrapping as a Json will return our array of data as JSON through the API.

Create a routing config by opening `server/src/routes/mod.rs`, and add to it:

```rust
use actix_web::web;

pub fn routes(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/api")
            .service(web::scope("/questions").route("", web::get().to(questions::get_all))),
    );
}
```

In this function we'll be able to add all sorts of routes to it. This sets up a top level /api with no default, and then a /questions underneath it. We specify our get_all as a GET request.

Open our server/src/main.rs file, attach the routes to the App, and add a database pool.

```rust
// at the top of main()
let pool = db::new_pool();

// inside HttpServer::new block
App::new()
    .wrap(cors)
    .wrap(Logger::default())
    .wrap(Logger::new("%a %{User-Agent}i"))
    .data(pool.clone())         // <---- the news line here
    .configure(routes::routes)  // <----
```

Now that we have all these bits & pieces together, let's write some tests. Create a new file at `server/src/tests.rs`, so we can define some test helpers.

First setup a mod & our dependencies. We add the cfg(test) attribute so this module is only compiled in tests.

```rust
// server/src/main.rs
#[cfg(test)]
mod tests;
```

```rust
// server/src/tests.rs
use actix_http::Request;
use actix_service::Service;
use actix_web::{body::Body, dev::ServiceResponse, error::Error, test, App};
use serde::de::DeserializeOwned;

use crate::routes::routes;
```

These dependencies will be used for two functions. One to create the service to use for a single request, another to call our get request.

```rust
pub async fn get_service(
) -> impl Service<Request = Request, Response = ServiceResponse<Body>, Error = Error> {
    test::init_service(App::new().data(db::new_pool()).configure(routes)).await
}
```

`init_service` returns an instance of the Service trait, which requires for us to define a number of generic types. So for this we pull in a number of types from Actix that implement those traits. This function uses the init_service to create a new instance of the App with our routes.

Now add the following to the module:

```rust
pub async fn test_get<R>(route: &str) -> (u16, R)
where
    R: DeserializeOwned,
{
    let mut app = get_service().await;
    let req = test::TestRequest::get().uri(route);
    let res = test::call_service(&mut app, req.to_request()).await;

    let status = res.status().as_u16();
    let body = test::read_body(res).await;
    let json_body = serde_json::from_slice(&body).unwrap_or_else(|_| {
        panic!(
            "read_response_json failed during deserialization. response: {} status: {}",
            String::from_utf8(body.to_vec())
                .unwrap_or_else(|_| "Could not convert Bytes -> String".to_string()),
            status
        )
    });

    (status, json_body)
}
```

This uses the get_service to create the application service, and then creates our get request. To execute it, we then call the `call_service` function, which accepts our app & route to process the request against. The idea is that this runs the app only for a single request.

After that we get the http status code, as well as the response body. This is assuming that we only get json responses back, but does have some error handling using a panic to force a test failure if deserialization goes wrong.

You'll have a couple compiler errors as we'll need to add serde to the server/Cargo.toml file. While you're there, add diesel as we'll need it for the tests.

```
diesel = { version = "1.4.4", features = ["postgres", "r2d2", "uuid", "chrono"] }
serde = "1.0.80"
serde_json = "1.0.13"
```


With that setup open up `get_all.rs`, and add the following to the bottom.

```rust
#[cfg(test)]
mod tests{
    // not a type we're using directly, but we need to pull in this trait.
    // If you leave it out, you'll see a compiler error for calling the query to insert.
    // The Rust compiler is pretty good about telling you what traits you're missing.
    use diesel::RunQueryDsl;

    use db::{
        get_conn,
        models::{Question},
        new_pool,
        schema::{questions}
    };

    use crate::tests;

    #[actix_rt::test]
    async fn test_get_all_returns_questions() {
        let pool = new_pool();
        let conn = get_conn(&pool);

        diesel::insert_into(questions::table).values(Question {
            body: "one question".to_string(),
        })
        .execute(&conn).unwrap();
    }
}
```

We have a few things going on here, as well as a compiler error.

`use diesel::RunQueryDsl;`, is not a type we're using directly, but we need to pull in this trait. If you leave it out, you'll see a compiler error for a trait that is missing, on the call to insert. In those cases the compiler will tell you which trait is needed, so you can simply add the use statement. I really love this about the compiler, as I forget traits all the time.

The other one I want to call out is `#[actix_rt::test]`. If you've written tests for Rust code before, you have probably annotated them with `#[test]`. The reason we use this one here is that the `actix_rt::test` allows us to make them async. If you read the [crate documentation](https://docs.rs/actix-rt/2.2.0/actix_rt/), you'll see that it is a "Tokio-based single-threaded async runtime for the Actix ecosystem". The test attribute macro marks the async function to be run in an Actix system.

Okay, hmm. So when doing this we can't omit fields like ID and record timestamps, as the struct requires them. The values function accepts a number of ways to pass data, but I do like using a typed struct. So let's create a new one in our db crate. Open up `db/src/models/question.rs`

```rust
// add Insertiable
use diesel::{PgConnection, Insertable, QueryDsl, Queryable, RunQueryDsl};

#[derive(Debug, Insertable)]
#[table_name = "questions"]
pub struct NewQuestion {
    pub body: String,
}
```

We need to add the table_name attribute in this case, cause the type does not match our table name here. Otherwise the only parameter we really care about is the body, so we add that as a field.

Now back in our test.

```rust
#[cfg(test)]
mod tests{
    use diesel::RunQueryDsl;

    use db::{
        get_conn,
        models::{Question, NewQuestion},
        new_pool,
        schema::{questions}
    };

    use crate::tests;

    #[actix_rt::test]
    async fn test_get_all_returns_questions() {
        let pool = new_pool();
        let conn = get_conn(&pool);

        diesel::insert_into(questions::table).values(NewQuestion {
            body: "one question".to_string(),
        })
        .execute(&conn).unwrap();

        let res: (u16, Vec<Question>) = tests::test_get("/api/questions").await;
        assert_eq!(res.0, 200);
        assert_eq!(res.1.len(), 1);
        assert_eq!(res.1[0].body, "one question");
    }
}
```

We create a question using our fancy new struct, and then get the status + resulting array from our endpoint.

For running the tests, I created a separate docker-compose file, and then added new commands to the Makefile.

```yaml
// docker-compose.test.yml
version: "3"

services:
  database_test:
    image: "postgres:10.5"
    ports:
      - "5433:5432"
    environment:
      POSTGRES_USER: root
      POSTGRES_PASSWORD: ""
      POSTGRES_DB: my_database_test
      POSTGRES_HOST_AUTH_METHOD: trust
```

Start up the test docker postgres container: `docker-compose -f docker-compose.test.yml up -d` then add some commands to your Makefile.

```Makefile
test_prepare:
	DATABASE_URL=postgres://root@localhost:5433/my_database_test diesel migration run --migration-dir=db/migrations

test:
	docker-compose -f docker-compose.test.yml exec database_test psql -d my_database_test --c="TRUNCATE questions"
	DATABASE_URL=postgres://root@localhost:5433/my_database_test \
		cargo test $(T) -- --nocapture --test-threads=1

# Update the phony line:
.PHONY: run_server test test_prepare
```

Once the container is up and running, run `make test_prepare` to run the migrations, and then you can run `make test` to run your tests anytime. I like to use the truncate command to ensure a clear database, but up to you. It's a good idea to ensure any data you create in your tests are cleaned up once that test case finishes. Otherwise you can pollute your other tests.

If all goes well, you should see something like this for your output. With 1 test passing for the server target.

```
    Finished test [unoptimized + debuginfo] target(s) in 50.88s
     Running target\debug\deps\db-ad00c7968215382d.exe

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

     Running target\debug\deps\server-578da7493e677323.exe

running 1 test
test routes::questions::get_all::tests::test_get_all_returns_questions ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

   Doc-tests db

running 0 tests

test result: ok. 0
```

## Making errors better

I noted earlier that getting a database connection, and calling an immediate unwrap() is not great. In most cases this may be fine, but what if the connection fails for some reason? It would cause the rust application to panic, and therefore stop running. Also as you grow your application you may want to manually return different error codes and types. We're going to create our own Error type & handling so we can support these use cases.

Create a new crate:

```bash
cargo new --lib errors
```

Add it to the Cargo workspace

```
members = ["db", "errors", "server"]
```

Then open `errors/Cargo.toml` and add the dependencies we need.

```
[dependencies]
actix-web = "3.2"
derive_more = "0.99.9"
diesel = { version = "1.4.4", features = ["postgres", "r2d2", "uuid", "chrono"] }
env_logger = "0.5.13"
log = "0.4.0"
r2d2 = "0.8.2"
serde = "1.0.80"
serde_json = "1.0.13"
```

A similar set to our other crates, using actix-web, diesel & r2d2 for the database, logging support, and serde. This is so we can handle errors from these different external data types. Again I really want to thank `ddimaria` on github, who has a similar setup in the repo I listed at the top of the post.

Open up `errors/src/lib.rs`. You can delete the default test, and start adding the following.

```rust
#[macro_use]
extern crate log;

use actix_web::{
    error::{BlockingError, ResponseError},
    Error as ActixError, HttpResponse,
};
use derive_more::Display;
use diesel::result::{DatabaseErrorKind, Error as DBError};
use r2d2::Error as PoolError;
use serde::{Deserialize, Serialize};

#[derive(Debug, Display, PartialEq)]
pub enum Error {
    BadRequest(String),
    InternalServerError(String),
    Unauthorized,
    Forbidden,
    NotFound(String),
    PoolError(String),
    BlockingError(String),
}
```

We have our usual use statements to pull in dependents, but also we setup our Error enum. I've added all the types we're looking to specifically bubble up. You can add more types, like in my own application I have UnprocessableEntity for returning specific validation errors. Such as when joining a lobby, and the username is already taken for that lobby. What I like about this is the usage of Rust enums. We can define some of them to carry an error message, along with it being a specific type.

For our enum we want it to respond as an actix_web ResponseError, so implement it:

```rust
impl ResponseError for Error {
    fn error_response(&self) -> HttpResponse {
        match self {
            Error::BadRequest(error) => {
                HttpResponse::BadRequest().json::<ErrorResponse>(error.into())
            }
            Error::NotFound(message) => {
                HttpResponse::NotFound().json::<ErrorResponse>(message.into())
            }
            Error::Forbidden => HttpResponse::Forbidden().json::<ErrorResponse>("Forbidden".into()),
            _ => {
                error!("Internal server error: {:?}", self);
                HttpResponse::InternalServerError()
                    .json::<ErrorResponse>("Internal Server Error".into())
            }
        }
    }
}
```

Again this is fairly flexible, in the end we just want to return an HttpResponse of somekind. For the 40x error codes, we're handling them more deliberately, and anything else is treated as a 500. In this case logging the details, and just returning a generic error to the API. If you build this code, you will see an error.

```
cannot find type `ErrorResponse` in this scope
not found in this scope
```

For those 40x status codes, we are passing a custom data type to dictate the JSON structure, however we haven't defined it yet, let's do it now.

```rust
#[derive(Debug, Deserialize, Serialize)]
pub struct ErrorResponse {
    pub errors: Vec<String>,
}
```

We need it to Serialize & Deserialize via serde, otherwise it's pretty simple. Just contains an array of error messages that will be returned to the client.

Then, we will add the From trait for some types on our ErrorResponse, so the into() ErrorResponse will work. This is done by implementing the From trait for `&str`, `String`, and `Vec<String>` types.

```rust
impl From<&str> for ErrorResponse {
    fn from(error: &str) -> Self {
        ErrorResponse {
            errors: vec![error.into()],
        }
    }
}

impl From<&String> for ErrorResponse {
    fn from(error: &String) -> Self {
        ErrorResponse {
            errors: vec![error.into()],
        }
    }
}

impl From<Vec<String>> for ErrorResponse {
    fn from(error: Vec<String>) -> Self {
        ErrorResponse { errors: error }
    }
}
```

Given we get Result types containing types defined in other crates, we will want to wrap those in our own Error. As our error type will be used in the routing handlers as well as other places. Again let's apply the From trait, using external error types, and apply this to our Error.

```rust
// Convert DBErrors to our Error type
impl From<DBError> for Error {
    fn from(error: DBError) -> Error {
        // Right now we just care about UniqueViolation from diesel
        // But this would be helpful to easily map errors as our app grows
        match error {
            DBError::DatabaseError(kind, info) => {
                if let DatabaseErrorKind::UniqueViolation = kind {
                    let message = info.details().unwrap_or_else(|| info.message()).to_string();
                    return Error::BadRequest(message);
                }
                Error::InternalServerError("Unknown database error".into())
            }
            DBError::NotFound => Error::NotFound("Record not found".into()),
            _ => Error::InternalServerError("Unknown database error".into()),
        }
    }
}

// Convert PoolError to our Error type
impl From<PoolError> for Error {
    fn from(error: PoolError) -> Error {
        Error::PoolError(error.to_string())
    }
}

impl From<BlockingError<Error>> for Error {
    fn from(error: BlockingError<Error>) -> Error {
        match error {
            BlockingError::Error(error) => error,
            BlockingError::Canceled => Error::BlockingError("Thread blocking error".into()),
        }
    }
}

impl From<ActixError> for Error {
    fn from(error: ActixError) -> Error {
        Error::InternalServerError(error.to_string())
    }
}
```

The code for each error type is converting it into one of our enum types.

With that ready, let's integrate it into our application. Add the dependency to both our server & db crates.

```
# Cargo.toml
[dependencies]
errors = { path = "../errors" }
```

Open up `db/src/models/question.rs`, and update the return error type in our get_all.

```rust
use errors::Error;

impl Question {
    pub fn get_all(conn: &PgConnection) -> Result<Vec<Question>, Error> {
        use crate::schema::questions::dsl::{body, questions};

        let all_questions = questions.order(body).load::<Question>(conn)?;

        Ok(all_questions)
    }
}
```

Because we're using the ? operator, the From trait conversion is doing most of the lifting for us. Open `server/src/routes/questions/get_all.rs` and change the error type there too.

```rust
// remove actix_web::errors::Error
use actix_web::{
    web::{block, Data, Json},
    Result,
};

// add our type
use errors::Error;
```

This doesn't clean up the use of unwrap() in `get_conn()`, so let's address that. Open up `db/src/lib.rs`, updating dependencies & get_conn as follows.

```rust
#[macro_use]
extern crate log;

use r2d2::Error;

pub fn get_conn(pool: &PgPool) -> Result<PooledConnection<ConnectionManager<PgConnection>>, Error> {
    pool.get().map_err(|err| {
        error!("Failed to get connection - {}", err.to_string());
        err.into()
    })
}
```

We add the `macro_use` for log, so we can note when a db connection fails. We're still returning a external crate type error here, but we already handle pool errors in the errors crate. With this change we mainly want to log the error, and then bubble it up.

Making this change will cause some compiler errors, so let's open up get_all.rs and fix them. Update the get_conn() calls to use the ? operator at the end in the get_all function.

```rust
pub async fn get_all(pool: Data<PgPool>) -> Result<Json<Vec<Question>>, Error> {
    let connection = get_conn(&pool)?;
    // etc
```

Then in the test, call unwrap(). Our tests dont know how to handle the result type, and having a panic fail a test is okay.

```rust
#[actix_rt::test]
async fn test_get_all_returns_questions() {
    let pool = new_pool();
    let conn = get_conn(&pool).unwrap();
    // etc
```

Now run `make test` to be sure we didn't cause any regressions.

```
running 1 test
test routes::questions::get_all::tests::test_get_all_returns_questions ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Yay! Now let's allow the API consumer to create some questions.

## Create endpoint

Let's add a create endpoint that allows the caller to create a new question. First thing is to add a create method to our question model. Open up `db/src/models/question.rb`, and let's add a create function to our existing `impl Question`. This code will be quite similar to the code we used to create a question in our tests.

```rust
impl Question {
    pub fn create(conn: &PgConnection, body: &String) -> Result<Question, Error> {
        use crate::schema::questions::dsl::{questions};

        let question = diesel::insert_into(questions).values(NewQuestion {
            body: body.clone(),
        }).get_result::<Question>(conn)?;

        Ok(question)
    }
}
```

We pull in the questions dsl as our insert target, and then re-use the NewQuestion we created for the tests. Instead of calling execute, which calls the insert and returns no value, we call get_result to return the data from the insert. As diesel notes in the docs:

>>> When this method is called on an insert, update, or delete statement, it will implicitly add a RETURNING * to the query, unless a returning clause was already specified.

With that we use the ? operator again to return the right error type, and then return an Ok() result if everything is good.

Now for the POST endpoint.

```bash
touch server/src/routes/questions/create.rs
```

Update the routes/questions module

```rust
// server/src/routes/questions/mod.rs
mod create;
pub use self::create::*;
```

Open up the new create.rs file, and let's get started. You can add most of the same use statements we had in get_all. In addition we use the serde Serialize & Deserialize traits, as we need to define a request parameters type.

```rust
use actix_web::{
    web::{block, Data, Json},
    Result,
};
use serde::{Deserialize, Serialize};

use db::{get_conn, models::Question, PgPool};
use errors::Error;


#[derive(Clone, Deserialize, Serialize)]
pub struct CreateRequest {
    body: String,
}

pub async fn create(pool: Data<PgPool>, params: Json<CreateRequest>) -> Result<Json<Question>, Error> {
    if params.body == "" {
        return Err(Error::BadRequest("Body is required".to_string()));
    }

    let connection = get_conn(&pool)?;

    let question = block(move || Question::create(&connection, &params.body)).await?;

    Ok(Json(question))
}
```

The create function looks pretty familiar, but we've added a second parameter for the request params. We declare it as JSON, with a given set of fields by stating the struct to use. If the data coming in is not JSON, or it does not match the structure of `CreateRequest`, actix will return a bad request error.

Upon the request looking okay, we should check that the body is not empty. If it is we return a bad request with a detail string. This is fine for the purposes of the tutorial, but for implementing more validations, I would suggest to look at [https://github.com/Keats/validator](https://github.com/Keats/validator).

After that we do the same operations as before by getting a connection from the pool, and calling our new create function.

Now update `server/src/routes/mod.rs` adding our new route.

**Note** be sure to have the `.route()` chain after the first one so it is part of the `scope()` call. When building this I had attached it to the closing paranthesis of the `service()` call. The route was 404ing on me!

```rust
use actix_web::web;

pub mod questions;

pub fn routes(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/api")
            .service(web::scope("/questions")
                .route("", web::get().to(questions::get_all))
                .route("", web::post().to(questions::create))),
    );
}
```

In order to test it, we'll need a new helper function since this is a POST request.

```rust
// server/src/test.rs

// update our serde use statement
use serde::{de::DeserializeOwned, Serialize};

pub async fn test_post<T: Serialize, R>(
    route: &str,
    params: T,
) -> (u16, R)
where
    R: DeserializeOwned,
{
    let mut app = get_service().await;

    let req = test::TestRequest::post().set_json(&params).uri(route);

    let res = test::call_service(&mut app, req.to_request()).await;

    let status = res.status().as_u16();
    let body = test::read_body(res).await;
    let json_body = serde_json::from_slice(&body).unwrap_or_else(|_| {
        panic!(
            "read_response_json failed during deserialization. response: {} status: {}",
            String::from_utf8(body.to_vec())
                .unwrap_or_else(|_| "Could not convert Bytes -> String".to_string()),
            status
        )
    });

    (status, json_body)
}
```

This works fairly similarly to our test_get. We setup the service, create a post request adding the parameters as JSON. We then return the status & the response body.

```rust
// create.rs

#[cfg(test)]
mod tests {
    use diesel::{self, RunQueryDsl};

    use db::{
        models::{NewQuestion, Question},
        get_conn,
        new_pool,
        schema::questions
    };

    use crate::tests;

    #[actix_rt::test]
    pub async fn test_create_question() {
        let pool = new_pool();
        let conn = get_conn(&pool).unwrap();

        let res: (u16, Question) = tests::test_post("/api/questions", NewQuestion {
            body: "A new question".to_string()
        }).await;

        assert_eq!(res.0, 200);
        assert_eq!(res.1.body, "A new question");

        let result_questions = questions::dsl::questions.load::<Question>(&conn).unwrap();
        assert_eq!(result_questions.len(), 1);
        assert_eq!(result_questions[0].body, "A new question");

        diesel::delete(questions::dsl::questions).execute(&conn).unwrap();
    }
}
```

Upon adding this you will get a compiler error:

```
the trait bound `db::models::question::NewQuestion: routes::questions::create::_::_serde::Serialize` is not satisfied
the trait `routes::questions::create::_::_serde::Serialize` is not implemented for `db::models::question::NewQuestion`
```

What this is saying is that NewQuestion is missing the serde Serialize trait to pass it through a request that is requiring JSON. So open up `db/src/models/question.rs` and add the Serialize derive to the struct.

```rust
#[derive(Debug, Insertable, Serialize)] // add Serialize here
#[table_name = "questions"]
pub struct NewQuestion {
    pub body: String,
}
```

The test is pretty similar to the one we have for get_all. We get a pool & connection so we can inspect the database after creating the record, as well as clean it up. Checking the status & response body working the same way here.

Run `make test`, and we can see the test case succeeds!

```
test routes::questions::create::tests::test_create_question ... ok
```

Now let's test our validation code.

```rust
// add above with the other use statements in the test module
use errors::ErrorResponse;

#[actix_rt::test]
pub async fn test_create_body_required() {
    let pool = new_pool();
    let conn = get_conn(&pool).unwrap();

    // note here how the deserialize type is changed
    let res: (u16, ErrorResponse) = tests::test_post("/api/questions", NewQuestion {
        body: "".to_string()
    }).await;

    assert_eq!(res.0, 400);
    assert_eq!(res.1.errors, vec!["Body is required"]);

    let result_questions = questions::dsl::questions.load::<Question>(&conn).unwrap();
    assert_eq!(result_questions.len(), 0);
}
```

We call the same endpoint with an empty string, specifying our `ErrorResponse` type as the response body. From there checking the http status, the value of the error message, and that no database records got written.

Run `make test` again and you should see:

```
running 3 tests
test routes::questions::create::tests::test_create_body_required ... ok
test routes::questions::create::tests::test_create_question ... ok
test routes::questions::get_all::tests::test_get_all_returns_questions ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

That covers our create endpoint, lets get into websockets.

## Websockets

In the title I mentioned the use of websockets. Actix gives us the ability to work with a websocket server, and to handle requests by using Actix's actor system. What we're going to do is create an HTTP endpoint for users to connect to the websocket. Then when a question is created, broadcast the question details to all users connected.

Create the files & folders we will need:

```
mkdir -p server/src/websocket
touch server/src/websocket/mod.rs
touch server/src/websocket/server.rs
```

```rust
// server/src/main.rs
mod websocket;
```

```rust
// server/src/websocket/mod.rs
mod server;
pub use self::server::*;
```

Let's start with `server.rs`.

```rust
use std::collections::HashMap;

use actix::prelude::{Message as ActixMessage, Recipient};
use serde::{Deserialize, Serialize};
use serde_json::Value;

#[derive(ActixMessage)]
#[rtype(result = "()")]
pub struct Message(pub String);

#[derive(ActixMessage, Deserialize, Serialize)]
#[rtype(result = "()")]
pub struct MessageToClient {
    pub msg_type: String,
    pub data: Value,
}

impl MessageToClient {
    pub fn new(msg_type: &str, data: Value) -> Self {
        Self {
            msg_type: msg_type.to_string(),
            data,
        }
    }
}

pub struct Server {
    sessions: HashMap<String, Recipient<Message>>
}

impl Server {
    pub fn new() -> Self {
        Server {
            sessions: HashMap::new(),
        }
    }
}
```

Going top to bottom, we define Message, to have a struct wrapping around a String. This is the basics of a message coming over the websocket. They need to have the return type defined, so we do this using an attribute. `#[rtype(result = "()")]`. For this and all our types, we're just returning an empty value.

With `MessageToClient` we declare the type of message via a string key. So in our case we will just pass "newquestion", as we will be sending the new question data. The data field is one of Value from serde_json. So any type that can be converted into a Value type. From there we setup a basic constructor.

Finally we setup the struct for our WebSocket Server. This will store the current sessions using a String as a key, which we will generate UUIDs for. The value being the `Recipient` that sent the original `Message`. So when a user connects we receive the `Recipient<Message>` from the HTTP request, and store it in the Server.

Add a send_message function:

```rust
use serde_json::{error::Result as SerdeResult, Value}; // add the Result use here

impl Server {
    fn send_message(&self, data: SerdeResult<String>) {
        match data {
            Ok(data) => {
                for recipient in self.sessions.values() {
                    match recipient.do_send(Message(data.clone())) {
                        Err(err) => {
                            error!("Error sending client message: {:?}", err);
                        }
                        _ => {}
                    }
                }
            }
            Err(err) => {
                error!("Data did not convert to string {:?}", err);
            }
        }
    }
}
```

To make calls from the handlers a little easier, we pass a SerdeResult type and let our server handle the error handling if any. You could opt to do this as the route handler instead. Which may be a better way to go if you want to respond to the user that something went wrong with sending the websocket message.

We will now setup a set of messages to listen for:

```rust
// a number of new actix & serde imports
use actix::prelude::{Actor, Context, Handler, Message as ActixMessage, Recipient};
use serde_json::{error::Result as SerdeResult, to_string, Value};

#[derive(ActixMessage)]
#[rtype(result = "()")]
pub struct Connect {
    pub addr: Recipient<Message>,
    pub id: String,
}

impl Handler<Connect> for Server {
    type Result = ();

    fn handle(&mut self, msg: Connect, _: &mut Context<Self>) {
        self.sessions.insert(msg.id.clone(), msg.addr);
    }
}

#[derive(ActixMessage)]
#[rtype(result = "()")]
pub struct Disconnect {
    pub id: String,
}

impl Handler<Disconnect> for Server {
    type Result = ();

    fn handle(&mut self, msg: Disconnect, _: &mut Context<Self>) {
        self.sessions.remove(&msg.id);
    }
}

impl Handler<MessageToClient> for Server {
    type Result = ();

    fn handle(&mut self, msg: MessageToClient, _: &mut Context<Self>) -> Self::Result {
        self.send_message(to_string(&msg));
    }
}
```

We've created a few new types based on `ActixMessage`. For the `Connect` message we will get that initial ID as well as the `Recipient<Message>`. Likewise we have a `Disconnect` which removes its identitifier from the HashMap. And then we implement the `Handler` for `MessageToClient`, which calls our helper function we just previously implemented, serializing the object as JSON data. The reason for the Handler trait, is so that we can pass any of these types to our websocket Server.

A number of compiler errors will show along the lines of

```
the trait bound `websocket::server::Server: actix::actor::Actor` is not satisfied
the trait `actix::actor::Actor` is not implemented for `websocket::server::Server`
```

Let's add the trait.

```rust
impl Actor for Server {
    type Context = Context<Self>;
}
```

What this does is allows the Server to recieve messages (via the Handler trait) as an Actor. With these types applied, we can register the Server as an Actor with actix, which means we can pull it in to our route handlers, similar to the postgres connection pool, and send messages to it. Actix will then send these messages asynchronously.

Open up `server/src/websocket/mod.rs` so we can setup the HTTP route, plus some other key pieces.

```rust
use std::time::{Duration, Instant};

use uuid::Uuid;

use actix::{
    prelude::{Actor, Addr}
};

// same as before
mod server;
pub use self::server::*;

const HEARTBEAT_INTERVAL: Duration = Duration::from_secs(5);
const CLIENT_TIMEOUT: Duration = Duration::from_secs(30);

pub struct WebSocketSession {
    id: String,
    hb: Instant,
    server_addr: Addr<Server>,
}


impl WebSocketSession {
    fn new(server_addr: Addr<Server>) -> Self {
        Self {
            id: Uuid::new_v4().to_string(),
            hb: Instant::now(),
            server_addr,
        }
    }

    fn send_heartbeat(&self, ctx: &mut <Self as Actor>::Context) {
        ctx.run_interval(HEARTBEAT_INTERVAL, |act, ctx| {
            if Instant::now().duration_since(act.hb) > CLIENT_TIMEOUT {
                info!("Websocket Client heartbeat failed, disconnecting!");
                act.server_addr.do_send(Disconnect { id: act.id.clone() });
                // stop actor
                ctx.stop();

                // don't try to send a ping
                return;
            }
            ctx.ping(b"");
        });
    }
}
```

We start off by setting some constants of a heartbeat rate to keep the connection alive, as well as what we consider to be a timeout. We then create a new struct that tracks the id (uuid) of a session, along with the heartbeat, and the server address. The latter is how the session can communicate with the websocket server.

The send_heartbeat function we will setup to be called soon, but the idea is that it uses an actor context to run at a set interval. Provided the heartbeat doesn't get refreshed, it will send a disconnect to the server, so our server clears this session. It will then also stop the context of this actor, so the session gets cleaned up.

uuid is a new dependency, so open up `server/Cargo.toml` to add uuid as a dependency.

```
uuid = { version = "0.5", features = ["serde", "v4"] }
```

We have a compiler error, similar to the one we have on the websocket Server.

```
the trait bound `websocket::WebSocketSession: actix::actor::Actor` is not satisfied
the trait `actix::actor::Actor` is not implemented for `websocket::WebSocketSession`
```

The fix for the Server was pretty straight forward, but this one is a little more involved.

```rust
// update use statements again
use actix::{
    fut,
    prelude::{Actor, Addr},
    ActorContext, AsyncContext
};
use actix_web_actors::ws;

impl Actor for WebSocketSession {
    type Context = ws::WebsocketContext<Self>;

    fn started(&mut self, ctx: &mut Self::Context) {
        self.send_heartbeat(ctx);

        let session_addr = ctx.address();
        self.server_addr
            .send(Connect {
                addr: session_addr.recipient(),
                id: self.id.clone(),
            })
            .into_actor(self)
            .then(|res, _act, ctx| {
                match res {
                    Ok(_res) => {}
                    _ => ctx.stop(),
                }
                fut::ready(())
            })
            .wait(ctx);
    }
}
```

Similar to the websocket Server, we set the context as a websocket context, which gives us the capability to have the heartbeat interval. We then setup a `started` function, which is called when the actor starts up. We tell the context to connect the heartbeat function we just wrote, and then send an initial Connect message to the server. Provided it goes well, we return a future::ready, which is similar to a `Promise.resolve()` in JavaScript. With this we're returning an Ok() future. Otherwise if things go wrong we have the context stop right away.

We're getting errors about the WebSocketSession not implementing Handler, so let's fix this.

```rust
use actix::{
    fut,
    prelude::{Actor, Addr, Handler},
    ActorContext, ActorFuture, AsyncContext, ContextFutureSpawner, WrapFuture
};

impl Handler<Message> for WebSocketSession {
    type Result = ();

    fn handle(&mut self, msg: Message, ctx: &mut Self::Context) {
        ctx.text(msg.0);
    }
}
```

A bit simpler this time! We just pass the message String to the actor context as a text message. This is how we pass down the messages from the websocket Server to the client. If you'll recall, the `MessageToClient` struct would pass its data to the Server's `send_message`. This function would create an instance of `Message` and pass to the recipient context. Which then comes back to this `WebSocketSession`.

Now we need to implement the `StreamHandler` trait, which will allow us to handle a stream of events coming to the Actor. Using the ping/pong to update the heartbeat, close events, errors, etc.

```rust
// yep, this again
use actix::{
    fut,
    prelude::{Actor, Addr, Handler, StreamHandler},
    ActorContext, ActorFuture, AsyncContext, ContextFutureSpawner, WrapFuture
};

impl StreamHandler<Result<ws::Message, ws::ProtocolError>> for WebSocketSession {
    fn handle(&mut self, msg: Result<ws::Message, ws::ProtocolError>, ctx: &mut Self::Context) {
        match msg {
            Ok(ws::Message::Ping(msg)) => {
                self.hb = Instant::now();
                ctx.pong(&msg);
            }
            Ok(ws::Message::Pong(_)) => {
                self.hb = Instant::now();
            }
            Ok(ws::Message::Binary(bin)) => ctx.binary(bin),
            Ok(ws::Message::Close(reason)) => {
                self.server_addr.do_send(Disconnect {
                    id: self.id.clone(),
                });
                ctx.close(reason);
                ctx.stop();
            }
            Err(err) => {
                warn!("Error handling msg: {:?}", err);
                ctx.stop()
            }
            _ => ctx.stop(),
        }
    }
}
```

The Ping & Pong events are for a back and forth basic sending of data to confirm the connection is still alive. They update the heartbeat to the current timestamp, so the heartbeat interval will know not to close the connection. We also capture any binary requests, not really doing a whole lot with them. If a close request comes through, we send a Disconnect to the server and close the WebSocketSession.

Last bit of code for this file, we need to add the http request handler to kick off a new session.

```rust
use actix_web::{web, HttpRequest, HttpResponse};

use errors::Error;

pub async fn ws_index(
    req: HttpRequest,
    stream: web::Payload,
    server_addr: web::Data<Addr<Server>>,
) -> Result<HttpResponse, Error> {
    let res = ws::start(
        WebSocketSession::new(server_addr.get_ref().clone()),
        &req,
        stream,
    )?;

    Ok(res)
}
```

This uses the actix websocket crate to start a new actor. We use the server actor address from the data to pass it through, passing request & stream info for the actor to get created.

Open `server/src/main.rs` and connect the web server!

```rust
use actix::Actor;

// inside main()
let server = websocket::Server::new().start(); // new line

App::new()
    .wrap(cors)
    .wrap(Logger::default())
    .wrap(Logger::new("%a %{User-Agent}i"))
    .data(pool.clone())
    .data(server.clone()) // new line here
    .configure(routes::routes)
```

Connect the websocket route

```rust
use actix_web::web;

use crate::websocket;

pub mod questions;

pub fn routes(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::resource("/ws/").route(web::get().to(websocket::ws_index))
    ).service(
        web::scope("/api")
            .service(web::scope("/questions")
                .route("", web::get().to(questions::get_all))
                .route("", web::post().to(questions::create))),
    );
}
```

Open up `server/src/routes/questions/create.rs` and lets pass the new question to the websocket.

```rust
// new use statements
use actix::Addr;
use serde_json::to_value;

use crate::websocket::{MessageToClient, Server};

// updates to create()
pub async fn create(
    pool: Data<PgPool>,
    websocket_srv: Data<Addr<Server>>,
    params: Json<CreateRequest>,
) -> Result<Json<Question>, Error> {
    if params.body == "" {
        return Err(Error::BadRequest("Body is required".to_string()));
    }

    let connection = get_conn(&pool)?;

    let question = block(move || Question::create(&connection, &params.body)).await?;

    if let Ok(question) = to_value(question.clone()) {
        let msg = MessageToClient::new("newquestion", question);
        websocket_srv.do_send(msg);
    }

    Ok(Json(question))
}
```

We add the websocket server as a data paramter. Wrapping it in the Addr type because it's an Actor. After creating the question we call the websocket server, first converting the question into a Value, then sending it as a MessageToClient. You will get some compiler errors, so let's fix it.

First is that our Question does not implement `clone()` so open up `db/src/models/question.rs`.

```rust
#[derive(Clone, Debug, Identifiable, Serialize, Deserialize, Queryable)]
pub struct Question {
    pub id: i32,
    pub body: String,
    pub created_at: DateTime<Utc>,
    pub updated_at: DateTime<Utc>,
}
```

Here we add Clone to the derive list, so our struct can be cloned. We use this as passing it without a clone to `to_value` would cause a move, and we wouldnt be able to return the question in the JSON response.

Now we'll update the tests to support the websocket. Open up `server/src/tests.rs`.

```rust
// add the use statements so we can add the Server
use actix::Actor;

use crate::websocket::Server;

// update get_service to include the websocket server
pub async fn get_service(
) -> impl Service<Request = Request, Response = ServiceResponse<Body>, Error = Error> {
    test::init_service(
        App::new()
            .data(db::new_pool())
            .data(Server::new().start())
            .configure(routes),
    )
    .await
}
```

Run `make test` to ensure our current tests pass.

## Testing the websocket

With our past tests we've used the `test_service`, but that doesn't really work for the websocket Server, as we need to have the actor running to recieve messages from it. So let's go for a different methodology. Open up `server/src/tests.rs`.

```rust
use actix_web_actors::ws;

use crate::websocket::{MessageToClient, Server}; // add MessageToClient here

pub fn get_test_server() -> test::TestServer {
    test::start(|| {
        App::new()
            .data(db::new_pool())
            .data(Server::new().start())
            .configure(routes)
    })
}

pub fn get_websocket_frame_data(frame: ws::Frame) -> Option<MessageToClient> {
    match frame {
        ws::Frame::Text(t) => {
            let bytes = t.as_ref();
            let data = String::from_utf8(bytes.to_vec()).unwrap();
            let value: MessageToClient = serde_json::from_str(&data).unwrap();
            return Some(value);
        }
        _ => {}
    }

    None
}
```

We have two functions here. The first one using pretty much the same contents as `get_service` but instead it returns a running web server. The second is a helper function that will be useful for getting frames from the websocket connection stream, and turning them into json data that we can read.

Open up the `create.rs` file again, and let's update the happy path test.

```rust
#[cfg(test)]
mod tests {
    use actix_web::client::Client;
    use futures::StreamExt;
    use serde_json;
    // etc
```

We add some important use statements here. `Client` is a web client that we can use to establish a websocket connection. `StreamExt` is one we will need for streaming the websocket responses. `serde_json` will we use for converting the data from the websocket into our types.

Jump down to the `test_create_question` function and empty it, we're replacing most of it.

```rust
#[actix_rt::test]
pub async fn test_create_question() {
    let pool = new_pool();
    let conn = get_conn(&pool).unwrap();

    let srv = tests::get_test_server();

    let client = Client::default();
    let ws_conn = client.ws(srv.url("/ws/")).connect().await.unwrap();

    let mut res = srv
        .post("/api/questions")
        .send_json(&NewQuestion {
            body: "A new question".to_string(),
        })
        .await
        .unwrap();
```

As before, we create the database connection pool & get a connection. We use our new function to start the test server. We then create a new web client and connect to the websocket server via the URL we specified in route configuration. We then replace our test_post() call with calling post() directly on the test server.

```rust
    assert_eq!(res.status().as_u16(), 200);

    let question: Question = res.json().await.unwrap();
    assert_eq!(question.body, "A new question");

    let mut stream = ws_conn.1.take(1);
    let msg = stream.next().await;

    let data = tests::get_websocket_frame_data(msg.unwrap().unwrap());
    if data.is_some() {
        let msg = data.unwrap();
        assert_eq!(msg.msg_type, "newquestion");
        let question: Question = serde_json::from_value(msg.data).unwrap();
        assert_eq!(question.body, "A new question");
    } else {
        assert!(false, "Message was not a string");
    }
```

Similar to our old test we check the status & response body from the HTTP request. We then get a stream from the websockect connection, just asking for a total of 1 message via `take()`. Pulling the data out of the frame, we use the Option type to handle if anything went wrong. Provided the data is the type we need we then check the fields from the websocket frame.

```rust
    drop(stream);

    let result_questions = questions::dsl::questions.load::<Question>(&conn).unwrap();
    assert_eq!(result_questions.len(), 1);
    assert_eq!(result_questions[0].body, "A new question");

    srv.stop().await;

    diesel::delete(questions::dsl::questions)
        .execute(&conn)
        .unwrap();
}
```

Finish up the test by stopping the stream. We use the standard drop() for this, as the stream implements the Drop trait. This will get called for us when the variable is out of scope, but we also need to stop the server, so that is why we must manually call drop here. Afterwards we have our previous test code of checking the database & cleaning it up, then the server stop code.

Run `make test` and you should be good to go!

## Wrap up

Can find the completed source to reference [here](https://github.com/agmcleod/questions-app).

That's it for the functionality implementation. There are improvements that could be made here. One being our use of `do_send` in the `create` route handler. What if the message failed to send? We should at least log the error, or better yet return an error to the user.

If you want to learn more, here's a few topics I'd suggest looking at:

- [Futures](https://docs.rs/futures/0.3.14/futures/)
- [Actix Actors](https://docs.rs/actix/0.11.1/actix/prelude/trait.Actor.html)
- [Actix Identity](https://docs.rs/actix-identity/0.3.1/actix_identity/). We didn't talk about authentication at all, but I do leverage this crate in my project: [https://github.com/agmcleod/sc-predictions-server](https://github.com/agmcleod/sc-predictions-server). Can look at the auth crate I setup in there, and how it's integrated in both the websockets & HTTP apis.
