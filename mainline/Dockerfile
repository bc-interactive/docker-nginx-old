FROM debian:buster-slim AS base
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

FROM base AS downloader
# hadolint ignore=DL3009,DL3008
RUN apt-get update \
  && apt-get install --yes --no-install-recommends \
   ca-certificates \
   dirmngr \
   gnupg \
   git \
   wget \
   xz-utils

FROM downloader AS nginx
ARG NGINX_VERSION=1.19.10
RUN wget -nv --compression=gzip -O "nginx-${NGINX_VERSION}.tar.gz" "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz" \
  && wget -nv --compression=gzip -O "nginx-${NGINX_VERSION}.tar.gz.asc" "https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz.asc" \
  && wget -nv --compression=gzip -O "mdounin.key" "http://nginx.org/keys/mdounin.key" \
  && GNUPGHOME="$(mktemp -d)" \
  && export GNUPGHOME \
  && gpg --batch --import "mdounin.key" \
  && gpg --batch --verify "nginx-${NGINX_VERSION}.tar.gz.asc" "nginx-${NGINX_VERSION}.tar.gz" \
  && tar xzf "nginx-${NGINX_VERSION}.tar.gz"

FROM downloader AS ngx_brotli
RUN git clone https://github.com/google/ngx_brotli.git
WORKDIR /ngx_brotli
RUN git submodule update --init

FROM downloader AS upx
RUN wget -nv --compression=gzip -O "upx-3.96-amd64_linux.tar.xz" "https://github.com/upx/upx/releases/download/v3.96/upx-3.96-amd64_linux.tar.xz" \
  && tar xf "upx-3.96-amd64_linux.tar.xz" --strip-components=1 \
  && chmod +x /upx

FROM base AS builder
ARG NGINX_VERSION=1.19.10
# hadolint ignore=DL3008
RUN apt-get update \
  && apt-get install --yes --no-install-recommends \
    binutils \
    ca-certificates \
    wget \
    gnupg \
    dirmngr \
    make \
    gcc \
    libc-dev \
    libpcre3-dev \
    zlib1g-dev \
    libssl-dev
COPY --from=nginx /nginx-${NGINX_VERSION} /nginx-${NGINX_VERSION}
COPY --from=ngx_brotli /ngx_brotli /ngx_brotli
COPY --from=upx /upx /usr/local/bin
WORKDIR /nginx-${NGINX_VERSION}
RUN ./configure \
    --user=nobody \
    --group=nobody \
    --prefix=/usr/local \
    --sbin-path=/usr/local/sbin \
    --conf-path=/etc/nginx/nginx.conf \
    --pid-path=/tmp/nginx.pid \
    --error-log-path=/dev/stderr \
    --http-log-path=/dev/stdout \
    --http-client-body-temp-path=/tmp/client_body_temp \
    --http-proxy-temp-path=/tmp/proxy_temp \
    --http-fastcgi-temp-path=/tmp/fastcgi_temp \
    --without-http_uwsgi_module \
    --without-http_scgi_module \
    --with-threads \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_gunzip_module \
    --with-pcre \
    --add-module=/ngx_brotli \
  && make -j "$(nproc)" \
  && make install \
  # smoke tests \
  && nginx -v \
  && mkdir -p /opt/etc/nginx/conf.d /opt/var/www/html /opt/usr/sbin \
  && cp /usr/local/sbin/nginx /opt/usr/sbin \
  && cp --archive --parents /etc/nginx /opt \
  && cp --archive --parents /etc/passwd /opt \
  && cp --archive --parents /etc/group /opt \
  && cp --archive --parents /etc/shadow /opt \
  # hardening: remove unnecessary accounts \
  && sed --in-place --regexp-extended '/^(root|nobody)/!d' /opt/etc/group \
  && sed --in-place --regexp-extended '/^(root|nobody)/!d' /opt/etc/passwd \
  && sed --in-place --regexp-extended '/^(root|nobody)/!d' /opt/etc/shadow \
  # hardening: remove interactive shell \
  && sed --in-place --regexp-extended 's#^([^:]+):([^:]+):([^:]+):([^:]+):([^:]+):([^:]+):(.+)$#\1:\2:\3:\4:\5:\6:/bin/nologin#' /opt/etc/passwd \
  # hardening: lock all accounts
  #while IFS=: read -r user _; do passwd -l "$user"; done < /etc/passwd
  && mkdir -p /opt/etc/nginx/conf.d /opt/var/www/html /opt/usr/sbin /opt/lib \
  && cp --archive -H -r /lib/x86_64-linux-gnu/libdl.so.* /opt/lib \
  && cp --archive -H -r /lib/x86_64-linux-gnu/libcrypt.so.* /opt/lib \
  && cp --archive -H -r /lib/x86_64-linux-gnu/libpcre.so.3.13.3 /opt/lib/libpcre.so.3 \
  && cp --archive -H -r /usr/lib/x86_64-linux-gnu/libssl.so.* /opt/lib \
  && cp --archive -H -r /usr/lib/x86_64-linux-gnu/libcrypto.so.* /opt/lib \
  && cp --archive -H -r /lib/x86_64-linux-gnu/libz.so.1.2.11 /opt/lib/libz.so.1 \
  && cp --archive -H -r /etc/localtime /opt/etc/ \
  && cp --archive /usr/local/html/* /opt/var/www/html \
  && cp --archive /usr/local/html/* /opt/var/www/html \
  && find /opt -executable -type f -exec strip --strip-all '{}' \; \
  && find /opt -executable -type f -exec upx '{}' \; \
  && chown -R root:root /opt \
  && find /opt -type d -exec chmod 0755 '{}' \; \
  && find /opt -type f -exec chmod 0644 '{}' \; \
  && chmod +x /opt/usr/sbin/nginx
COPY ./nginx.conf /opt/etc/nginx/nginx.conf
COPY ./conf.d /opt/etc/nginx/conf.d
COPY ./mime.types /opt/etc/nginx/mime.types

FROM busybox:1.32.1-glibc
ARG BUILD_DATE
ARG VCS_REF
LABEL org.opencontainers.image.created="${BUILD_DATE}"
LABEL org.opencontainers.image.url="https://github.com/bc-interactive/docker-memcached"
LABEL org.opencontainers.image.source="https://github.com/bc-interactive/docker-memcached"
LABEL org.opencontainers.image.version="${MEMCACHED_VERSION}"
LABEL org.opencontainers.image.revision="${VCS_REF}"
LABEL org.opencontainers.image.vendor="bc-interactive"
LABEL org.opencontainers.image.title="memcached"
LABEL org.opencontainers.image.authors="BC INTERACTIVE <contact@bc-interactive.fr>"
COPY --from=builder /opt /
STOPSIGNAL SIGQUIT
USER nobody
EXPOSE 80 443
ENTRYPOINT ["/usr/sbin/nginx", "-c", "/etc/nginx/nginx.conf"]
