FROM debian:trixie-slim AS build

ARG ASTERISK_VERSION=22.10.1
ARG PATCH_VERSION=22.10.0
ARG PATCH_SHA256=20f7c1c84e915e873c0eefae17d68302919cb6273940b6326f5838c0b5ac22ed

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update \
    && apt-get install --yes --no-install-recommends \
      ca-certificates \
      curl \
      build-essential \
      file \
      libedit-dev \
      libjansson-dev \
      libncurses-dev \
      libsqlite3-dev \
      libssl-dev \
      libsrtp2-dev \
      liburiparser-dev \
      libxml2-dev \
      patch \
      pkg-config \
      uuid-dev \
      wget \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /usr/src

RUN curl --fail --location --output asterisk.tar.gz \
      "https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-${ASTERISK_VERSION}.tar.gz" \
    && curl --fail --location --output cisco-usecallmanager.patch \
      "https://raw.githubusercontent.com/usecallmanagernz/patches/master/asterisk/cisco-usecallmanager-${PATCH_VERSION}.patch" \
    && echo "${PATCH_SHA256}  cisco-usecallmanager.patch" | sha256sum --check \
    && tar --extract --gzip --file asterisk.tar.gz \
    && cd "asterisk-${ASTERISK_VERSION}" \
    && patch --strip=1 < ../cisco-usecallmanager.patch \
    && ./configure --with-jansson-bundled=no \
    && make menuselect.makeopts \
    && ./menuselect/menuselect --disable BUILD_NATIVE menuselect.makeopts \
    && make -j"$(nproc)" all \
    && make install DESTDIR=/opt/asterisk-root \
    && make samples DESTDIR=/opt/asterisk-root \
    && rm -rf /opt/asterisk-root/var/run

FROM debian:trixie-slim

RUN apt-get update \
    && apt-get install --yes --no-install-recommends \
      ca-certificates \
      libedit2 \
      libjansson4 \
      libncurses6 \
      libsqlite3-0 \
      libssl3t64 \
      libsrtp2-1 \
      liburiparser1 \
      libuuid1 \
      libxml2 \
      tini \
    && rm -rf /var/lib/apt/lists/* \
    && groupadd --system asterisk \
    && useradd --system --gid asterisk --home-dir /var/lib/asterisk asterisk

COPY --from=build /opt/asterisk-root/ /

RUN mkdir -p /var/lib/asterisk /var/log/asterisk /run/asterisk /var/spool/asterisk \
    && chown -R asterisk:asterisk /var/lib/asterisk /var/log/asterisk /run/asterisk /var/spool/asterisk

EXPOSE 5060/tcp 5060/udp 10000-10100/udp

ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["/usr/sbin/asterisk", "-f", "-vvv", "-U", "asterisk", "-G", "asterisk"]
