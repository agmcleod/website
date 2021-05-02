---
title: Building a REST and Web Socket API with Actix and Rust
date: 2021-03-21 00:00:00
layout: post
---

The world of web development really has come a long way over the years. In late 90s to early 2000s I learned off various websites how to build web pages with HTML, tables, random JavaScript snippets, etc. Over time we got more sophisiticated server rendering options like asp, php, and then into MVC frame works like Rails and Django. Now we're writing the backend side as full on REST apis, where all the server does is return data, and the client uses that data to populate the interface. The interface is built with a variety of technologies, one can still use tech like I did 6-7 years ago: server rendered pages in Rails and writing jQuery code to make things more dynamic. But it's not the most scalable & organized code, so we now have these awesome frameworks like Angular & React to build flexibile and powerful web interfaces.

"Why are you going on about this" you ask. "This post is about Actix & Rust!" Good question, reader! I bring this wonderful history of web development up because I feel like websockets are a really powerful tool. I don't think they're needed for every case out there, I imagine we run into them on all sorts of applications we use day to day. That said it's not a technology that I've leveraged very much in my career working at various agencies. There are a number of frameworks one can dig into to leverage web sockets. I have done so via [Primus](https://github.com/primus/primus) in the past. [ActionHeroJS](https://www.actionherojs.com/) is a NodeJS API framework that offers relatively seamless handling of HTTP & Websocket requests. Though when building out a side project of mine I wanted to leverage both actix for its HTTP API capabitilies and integrate websockets for updating the user on state changes.

The application that I ended up building is one for users making predictions for a particular match or game. This in particular is for a StarCraft meetup I would go to, where we would have a raffle, and each game we would pick before hand who would be the one to win, as well as hit other first milestones in the game. For a sport you could equate this to first to score a goal, or first team to have a strikeout in baseball, etc. The idea was to model this in a similar way to the popular Jackbox games where users can easily join a game on their phone, select their picks and see a leaderboard. The benefits of web sockets for updating state of pick selection, the scoring, etc, came pretty clear to me.

This application has a bit more pieces to it than make sense for a tutorial like this. Instead we're going to build a piece of it as a questions repository. The user can view a list of questions, as well as create them. When questions are created, users connected will see the updated list. Relatively simple, but it covers a lot of the same concepts.

## Let's get into the setup

First off here is the current version of the full server for my application: [https://github.com/agmcleod/sc-predictions-server]. It is not open source, but I'm happy for folks to use it a reference in their own projects and hobbies.

Secondly I want to give a huge shout out to this repository here: [https://github.com/ddimaria/rust-actix-example](https://github.com/ddimaria/rust-actix-example), which was instrumental in helping me figure out some of the authentication pieces, as well as error handling.

For building this application, I'm using docker for running the database, and then compiling & running the rust code on my host machine. If you use Linux, you can use docker to run the rust code. On windows & mac, I found the compilation time was about 20 times slower. Building in docker would take minutes, where as outside of docker it was more like 10 seconds. However if you run the application on your host machine, you will need postgres development tools installed on your machine.

If you wish to do the same, setup a docker.compose.yml file like so:

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

We won't need all of these right away, but essentially we are providing the base actix crates we need for this application, futures for working with async code. Both in the application and in tests. Handling environment variables, so we don't need to have keys and host names hardcoded. And some good ol fashioned logging.

Let's set things up in `server/src/main.rs`.

```rust
#[macro_use]
extern crate log;

use std::env;

use actix_cors::Cors;
use actix_rt;
use actix_web::{http, middleware::Logger, web, App, HttpResponse, HttpServer};
use dotenv::dotenv;
use env_logger;

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

The first couple lines of code read the environment variables out of the .env file, as well as bash environment variables... which we haven't setup! Usually with docker I will pass environment variables through the compose file, but because the server is running on the host machine, i use a Makefile. If you're on windows, you'll need to have make installed first.

```Makefile
SHELL := /bin/bash

db_url := postgres://postgres:my_password@localhost:5434/my_database

run_server:
	CLIENT_HOST=http://localhost:3000 \
		RUST_BACKTRACE=full \
		cargo run --bin server

.PHONY: run_server
```

So now when we run `make run_server`, it will run with the variables we need. I snuck in a `RUST_BACKTRACE` so we can follow error stacks more easily. With that setup, we continue going over main().

```rust
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
```

We construct the web server, passing a number of services to it. First we allow CORS for our frontend's host only, and limit the headers allowed. Then we create the application server, setting up the logging, but no routes or middleware yet. Finally the server is started on port 8080.

Running `make run_server` should now compile and run the actix web server. It'll take sometime to compile and build all the dependencies for the first go. Once done, open a brower and visit [http://localhost:8080](http://localhost:8080) to verify it. You should see a blank page, and a 404 http response.

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

Add the following to db/Cargo.toml

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

Then run the setup command

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

The `diesel_manage_updated_at` function applies a trigger to ensure updated_at is populated for us whenever the record gets changed. Also populate the down.yml file with the drop table statement:

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
pub mod models;
pub mod schema;

use std::env;

pub type PgPool = Pool<ConnectionManager<PgConnection>>;

pub fn new_pool() -> PgPool {
    // again using our environment variable
    let database_url = env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let manager = ConnectionManager::<PgConnection>::new(database_url);

    Pool::builder()
        .build(manager)
        .expect("failed to create db pool")
}

```

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

Next add a function to Question struct to get us all the records. First update `db/src/lib.rs` by adding a macro_use. This is so we can pull data out of schema module.

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

The code here pulls in the dsl for querying the questions table, and the body field so we can sort by it. Using the ? operator to immediately return a diesel error if one ocurred. Wrapping the `Vec<Question>` in an Ok result.

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

Whew so many files! But I like to keep them separate so we don't crowd our route files code & tess.

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

pub use self::get_all::*;
```

Now let's make a module for our get all endpoint.

```bash
touch server/src/routes/queestions/get_all.rs
```

Open get_all.rs and let's create a route to return all questions in the database.

```rust
use actix_web::{
    error::Error,
    web::{block, Data, Json},
    Result,
};

use db::{models::Question, PgPool};

pub async fn get_all(pool: Data<PgPool>) -> Result<Json<Vec<Question>>, Error> {
    let connection = pool.get().unwrap();

    let questions = block(move || Question::get_all(&connection)).await?;

    Ok(Json(questions))
}
```

Bringing together everything we've setup so far. Leverages the Question model with its ability to convert into JSON, pulling data out of the db, and returning as JSON. Using a generic actix error type. The reason for block() is so that the database operation, which is blocking functionality, happens on a separate thread. block returns a future, so we can call .await? on it to wait for it to be complete before returning a response.

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
