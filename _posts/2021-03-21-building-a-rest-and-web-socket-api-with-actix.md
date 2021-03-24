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

I'd suggest to use a newer version of postgres if you can, I first started on this project a few years ago.

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

db_url := postgres://postgres:abc123@localhost:5434/sc_predictions

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