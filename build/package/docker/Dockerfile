FROM        golang:1.14 AS builder

WORKDIR     /workspace

COPY        . .

RUN         CGO_ENABLED=0 GOOS=linux \
            go build \
                -installsuffix cgo \
                -o kube-httpcache \
                -a cmd/kube-httpcache/main.go

FROM        debian:stretch-slim AS vmod

WORKDIR     /

RUN         set -x \
            && \
            apt-get -qq update && apt-get -qq upgrade \
            && \
            apt-get -qq install \
						    build-essential \
								libtool \
								python-docutils \
								libmaxminddb-dev \
								libmaxminddb0 \
								git \
								automake \
								make \
								autoconf \
								pkg-config \
								gnupg \
								curl \
								apt-transport-https \
								libgeoip-dev \
								debian-archive-keyring
#						&& \
#            curl -Ss -L https://packagecloud.io/varnishcache/varnish60lts/gpgkey | apt-key add - \
#            && \
#            printf "%s\n%s" \
#                "deb https://packagecloud.io/varnishcache/varnish60lts/debian/ stretch main" \
#                "deb-src https://packagecloud.io/varnishcache/varnish60lts/debian/ stretch main" \
#            > "/etc/apt/sources.list.d/varnishcache_varnish60lts.list"
RUN         apt-get -qq install \
                libvarnishapi-dev \
                varnish
RUN         git clone https://github.com/simonvik/libvmod_maxminddb.git
WORKDIR     /libvmod_maxminddb
RUN         ./autogen.sh
RUN				  ./configure  VMOD_DIR=/usr/lib/varnish/vmods/
RUN 				make
RUN				  make install

FROM        debian:stretch-slim AS final

ENV         EXPORTER_VERSION=1.6

LABEL       MAINTAINER="Martin Helmich <m.helmich@mittwald.de>"

WORKDIR     /

COPY        --from=vmod /workspace/kube-httpcache .

RUN         set -x \
            && \
            apt-get -qq update && apt-get -qq upgrade \
            && \
            apt-get -qq install \
                debian-archive-keyring \
                curl \
                gnupg \
                apt-transport-https \
								libgeoip-dev \
								software-properties-common \
								git \
								make \
								automake \
								autoconf \
								libtool \
								build-essential \
								libvarnishapi1 \
								libvarnishapi-dev \
								python-docutils \
            && \
            curl -Ss -L https://packagecloud.io/varnishcache/varnish60lts/gpgkey | apt-key add - \
            && \
            printf "%s\n%s" \
                "deb https://packagecloud.io/varnishcache/varnish60lts/debian/ stretch main" \
                "deb-src https://packagecloud.io/varnishcache/varnish60lts/debian/ stretch main" \
            > "/etc/apt/sources.list.d/varnishcache_varnish60lts.list" \
            && \
            apt-get -qq update && apt-get -qq install varnish varnish-dev
#            && \
#            apt-get -qq purge curl gnupg apt-transport-https && \
#            apt-get -qq autoremove && apt-get -qq autoclean && \
#            rm -rf /var/cache/*

#RUN         git clone --recursive https://github.com/maxmind/libmaxminddb
#
#WORKDIR     /libmaxminddb
#
#RUN         ./bootstrap && \
#            ./configure && \
#						make && \
#						make check -j 4 && \
#						make install && \
#						cp /usr/local/lib/libmaxminddb.so /usr/lib/varnish/vmods/libvmod_maxminddb.so
#
WORKDIR     /root
RUN         add-apt-repository ppa:maxmind/ppa && \
            apt-get install -qq libmaxminddb0 libmaxminddb-dev mmdb-bin #build-essential libtool libvarnishapi-dev python-docutils

WORKDIR     /

RUN         git clone https://github.com/scoremedia/libvmod_maxminddb.git
WORKDIR     /libvmod_maxminddb
RUN         ./autogen.sh && \
						./configure  VMOD_DIR=/usr/lib/varnish/vmods/ && \
						make && \
						make install
WORKDIR     /

RUN         mkdir /exporter && \
            chown varnish /exporter

ADD         --chown=varnish https://github.com/jonnenauha/prometheus_varnish_exporter/releases/download/${EXPORTER_VERSION}/prometheus_varnish_exporter-${EXPORTER_VERSION}.linux-amd64.tar.gz /tmp

RUN         cd /exporter && \
            tar -xzf /tmp/prometheus_varnish_exporter-${EXPORTER_VERSION}.linux-amd64.tar.gz && \
            ln -sf /exporter/prometheus_varnish_exporter-${EXPORTER_VERSION}.linux-amd64/prometheus_varnish_exporter prometheus_varnish_exporter

COPY        --from=builder /workspace/kube-httpcache .

ENTRYPOINT  [ "/kube-httpcache" ]
