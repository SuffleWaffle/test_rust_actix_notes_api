FROM rust:slim

RUN apt-get update && \
  apt-get -y upgrade && \
  apt-get -y install libpq-dev

WORKDIR /app
COPY . /app/
COPY .env.docker /app/.env

RUN cargo build --release

EXPOSE 8001

ENTRYPOINT ["/bin/bash", "-c", "cargo run --release"]

# _____________________________________________________________________________________________________________________

# This is what I use in most scenarios, it uses cargo chef,
# a tool used to maximize docker build caching with rust.
# It uses multi-stage builds to make the last layer orders of magnitude smaller.

FROM lukemathwalker/cargo-chef:latest as chef
WORKDIR /app

FROM chef AS planner
COPY ./Cargo.toml ./Cargo.lock ./
COPY ./src ./src
RUN cargo chef prepare

FROM chef AS builder
COPY --from=planner /app/recipe.json .
RUN cargo chef cook --release
COPY . .
RUN cargo build --release
RUN mv ./target/release/<your-crate> ./app

FROM debian:stable-slim AS runtime
WORKDIR /app
COPY --from=builder /app/app /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/app"]

# _____________________________________________________________________________________________________________________

# If you can remove all non-rust dependencies, you can use musl and statically link using alpine,
# which allows your final stage to be scratch, essentially making the entire
# docker image only as large as your final rust binary.

# Some performance issues with musl and tthe usage of Alpine images are desribed here:
# https://andygrove.io/2020/05/why-musl-extremely-slow/

FROM lukemathwalker/cargo-chef:latest-rust-alpine as chef
WORKDIR /app

FROM chef AS planner
COPY ./Cargo.toml ./Cargo.lock ./
COPY ./src ./src
RUN cargo chef prepare

FROM chef AS builder
COPY --from=planner /app/recipe.json .
RUN cargo chef cook --release
COPY . .
RUN cargo build --release
RUN mv ./target/release/<your-crate> ./app

FROM scratch AS runtime
WORKDIR /app
COPY --from=builder /app/app /usr/local/bin/
ENTRYPOINT ["/usr/local/bin/app"]