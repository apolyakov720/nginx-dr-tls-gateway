FROM tls-gost-tools

RUN apt-get update && apt-get install -y --no-install-recommends nginx \
    && mkdir -p /etc/nginx/certs \
    && sed -i '1i nginx = openssl_init' /etc/ssl/openssl-gost.cnf

COPY nginx.conf /etc/nginx/nginx.conf

ENTRYPOINT ["nginx"]
CMD ["-g", "daemon off;"]