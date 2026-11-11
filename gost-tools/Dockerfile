FROM debian:bookworm-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    openssl \
    libengine-gost-openssl \
    ca-certificates

COPY openssl-gost.cnf /etc/ssl/openssl-gost.cnf

ENV OPENSSL_CONF=/etc/ssl/openssl-gost.cnf

WORKDIR /work

ENTRYPOINT ["openssl"]