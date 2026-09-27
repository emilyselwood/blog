---
title: "Rust workspaces and docker"
date: 2026-04-16T21:02:00+00:00
author: "Emily Selwood"
draft: false
slug: "rust_workspaces_docker"
tags: ["rust", "docker", "workspaces"]
comments: false     # set false to hide Disqus comments
share: true        # set false to share buttons
ShowPostNavLinks: true
---

Recently I've been working on a multi workspace rust project. It's a client and server with some shared libraries and tools. I've managed to get it to build bits of its self in docker, with dependency caching, and a very minimal container image.

Rust workspaces are a way to have a monorepo with several sub projects. You can define dependency versions globally across all the sub projects to make things consistent, you can keep slowly changing builds cached, and not have to build some bits if you are not working on them.

My project is a multiplayer game. The game or client side is using [bevy](https://bevy.org/) which is a very large dependency and one of the largest libraries that I've used with rust. The server is an [actix-web](https://actix.rs/) and [diesel](https://diesel.rs/) based project. Which are also quite large. I really don't want to be building bevy every time I change the web server, and vise versa. Also bevy is big enough my laptop with 16Gb of ram struggles to build it, some times it takes a couple of goes.

The server needs to run in docker, to make deployments easier. But I don't want to have to wait for the entire project to build every time I deploy it.

Workspaces have a layout like this in rust

```
|- Cargo.toml # top level workspace config.
|- Cargo.lock # global lock file of dependencies
|- data/  # a workspace folder (Shared library of data types to share between backend and front end)
| |- Cargo.toml # a workspace config
| |- src/
|   |- lib.rs 
|   |- ... # rust code here
|- game/ # The game client workspace
| |- Cargo.toml # bevy is defined here
| |- src/
|   |- main.rs
|   |- ... # rest of game here
|- server/
  |- Cargo.toml # server workspace config (defines actix-web and diesel)
  |- Diesel.toml
  |- src/
    |- main.rs
    |- ... rest of server code
```

Cargo defaults to building the entire thing when running `cargo build` or `cargo test`. To run a single application you use `cargo run --bin game` or `cargo run --bin server` There are plenty of guides on line on setting up workspaces, including in the rust book.

Building a basic docker container was pretty easy. First I started off with a two stage build

```Dockerfile
FROM rust:1.94 AS build

WORKDIR /parsecsreach

COPY ./* .
RUN cargo build --bin server --release

FROM debian:bookworm-slim

# install postgres system library
RUN apt-get update && apt-get install libpq-dev -y && rm -rf /var/cache/apt/archives /var/lib/apt/lists/*

# copy the build artifact from the build stage
COPY --from=build /parsecsreach/target/release/server .

# set the startup command to run your binary
CMD ["./server"]
```

A small wrinkle in there to get the postgres library into the bookworm-slim image but that's not too much of a problem. Fairly standard approach to installing a package and then nuking the archives and lists.

The issue with this is that every time you build the container you will rebuild all of the dependencies for the server. Which can take a few minutes on my laptop, which sucks from a iteration time point of view, no hot reload of the server as it needs compiling means we are stuck waiting for that every time we change a line of code. bleh

So what we want is to have the dependencies compiled in one layer while our code compiles in a later layer.

If we just copy the `Cargo.toml` files from each workspace over we will get an error saying that the workspace doesn't contain a main or lib file. For the library code we can just create an empty `lib.rs` in a `src` directory next to the `Cargo.toml` and it'll let us past with no real code.

For the binary workspaces we need a main function too. For that we can include a dummy rust main file somewhere and copy that in. Then we do a `cargo build --bin server --release` and we have all our dependencies compiled and cached, and we can copy our real code over the top and run the build again.

But ...

If we do that and our dummy `lib.rs` and `main.rs` files are newer than the real ones (which if we touch the lib files to create them, is guaranteed) One of cargo or docker are "smart" enough to use that to think it doesn't need to copy over and rebuild our library and the binary for the server.

So we need to trick it into rebuilding the library and the binary.

The simplest I found was to remove the built artifacts for the library. I also tried changing the `Cargo.toml` files to point to a different main, which worked but seems a little more hacky.

```Dockerfile
FROM rust:1.94 AS build

WORKDIR /parsecsreach

# Copy cargo files
COPY ./Cargo.toml ./Cargo.toml
COPY ./Cargo.lock ./Cargo.lock

# copy and create wireframe of the project so we can build only the dependencies.
COPY ./data/Cargo.toml ./data/Cargo.toml
RUN mkdir -p ./data/src && touch ./data/src/lib.rs
COPY ./server/Cargo.toml ./server/Cargo.toml
COPY ./docker/template.main.rs ./server/src/dummy.rs
COPY ./game/Cargo.toml ./game/Cargo.toml
COPY ./docker/template.main.rs ./game/src/main.rs

# this should only build the dependencies
RUN sed -i 's#src/main.rs#src/dummy.rs#' server/Cargo.toml
RUN cargo build --bin server --release
RUN sed -i 's#src/dummy.rs#src/main.rs#' server/Cargo.toml

# Clean out the library stub builds so they get built properly in a second.
# These build files are newer than the ones in the source tree so cargo
# doesn't think it needs rebuilding.
RUN rm ./data/src/lib.rs && \
rm ./data/Cargo.toml && \
rm ./target/release/deps/libdata*

# Copy workspaces that we need
COPY ./data ./data
COPY ./server ./server

RUN cargo build --bin server --release

# our final base
FROM debian:bookworm-slim

# install postgres system library
RUN apt-get update && apt-get install libpq-dev -y && rm -rf /var/cache/apt/archives /var/lib/apt/lists/*

# copy the build artifact from the build stage
COPY --from=build /parsecsreach/target/release/server .

# set the startup command to run your binary
CMD ["./server"]
```

Now the dependencies will only re-compile if we change the `Cargo.toml` files which is reasonable, and we have a nice tiny container image to actually run the thing.
