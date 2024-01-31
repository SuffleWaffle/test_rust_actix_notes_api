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

# ---------------------------------------------------
# 1 - Build Stage
#
# Use official rust image to for application build
# ---------------------------------------------------
FROM rust:1.66.1 as build

# Setup working directory
WORKDIR /usr/src/codefee-works-api
COPY . .
COPY .env.docker .env

# Install dependency (Required by diesel)
RUN apt-get update && apt-get install libpq5 -y

# Build application
RUN cargo install --path .

# ---------------------------------------------------
# 2 - Deploy Stage
#
# Use a distroless image for minimal container size
# - Copy `libpq` dependencies into the image (Required by diesel)
# - Copy application files into the image
# ---------------------------------------------------
FROM gcr.io/distroless/cc-debian11

# Set the architecture argument (arm64, i.e. aarch64 as default)
# For amd64, i.e. x86_64, you can append a flag when invoking the build `... --build-arg "ARCH=x86_64"`
ARG ARCH=aarch64

# libpq related (required by diesel)
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libpq.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libgssapi_krb5.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libldap_r-2.4.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libkrb5.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libk5crypto.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libkrb5support.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/liblber-2.4.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libsasl2.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libgnutls.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libp11-kit.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libidn2.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libunistring.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libtasn1.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libnettle.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libhogweed.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libgmp.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /usr/lib/${ARCH}-linux-gnu/libffi.so* /usr/lib/${ARCH}-linux-gnu/
COPY --from=build /lib/${ARCH}-linux-gnu/libcom_err.so* /lib/${ARCH}-linux-gnu/
COPY --from=build /lib/${ARCH}-linux-gnu/libkeyutils.so* /lib/${ARCH}-linux-gnu/

# Application files
COPY --from=build /usr/local/cargo/bin/codefee-works-api /usr/local/bin/codefee-works-api
COPY --from=build /usr/src/codefee-works-api/.env /.env

CMD ["codefee-works-api"]


# _____________________________________________________________________________________________________________________

# Base image extends rust:nightly which extends debian:buster-slim
FROM ballistacompute/rust-cached-deps:0.2.3 as build

# Compile Ballista
RUN rm -rf /tmp/ballista/src/
COPY rust/Cargo.* /tmp/ballista/
COPY rust/build.rs /tmp/ballista/
COPY rust/src/ /tmp/ballista/src/
COPY proto/ballista.proto /tmp/ballista/proto/

# workaround for Arrow 0.17.0 build issue
COPY rust/format/Flight.proto /format

RUN cargo build --release

# Copy the binary into a new container for a smaller docker image
FROM debian:buster-slim

COPY --from=build /tmp/ballista/target/release/executor /
USER root

ENV RUST_LOG=info
ENV RUST_BACKTRACE=full

CMD ["/executor"]

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