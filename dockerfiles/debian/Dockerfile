# Zabbix Docker monitoring module build image
# https://github.com/ranas-mukminov/zabbix-docker-monitoring
#
# This Dockerfile builds the zabbix_module_docker.so for Debian-based systems.
# Target: Debian 12 (Bookworm) with Zabbix 7.0 LTS or 7.4
#
# Build arguments:
#   ZABBIX_VERSION - Zabbix version tag or branch (default: release/7.0)
#
# Updated: November 2025 (run-as-daemon.ru fork)

# Use Debian 12 (Bookworm) - current stable release
FROM debian:12-slim

LABEL maintainer="Ranas Mukminov <https://run-as-daemon.ru>"
LABEL description="Build environment for Zabbix Docker monitoring module on Debian"
LABEL upstream="https://github.com/monitoringartist/zabbix-docker-monitoring"

WORKDIR /root

# Install build dependencies
# - git: clone repositories
# - automake, autoconf, gcc, make: build tools
# - pkg-config: dependency detection
# - libpcre3-dev: PCRE library for regex support
# - libjansson-dev: JSON parsing library
RUN apt-get -qq update && \
    apt-get -qq --assume-yes --no-install-recommends install \
        git \
        automake \
        autoconf \
        gcc \
        make \
        pkg-config \
        libpcre3-dev \
        libjansson-dev \
        ca-certificates \
    1>/dev/null && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Clone this repository (module source)
RUN git clone -q https://github.com/ranas-mukminov/zabbix-docker-monitoring

# Define Zabbix version to build against
# Supported: release/7.4, release/7.0, release/6.0
# For exact tags: 7.0.0, 7.4.0, etc.
ARG ZABBIX_VERSION=release/7.0

# Build the module
RUN git clone -b ${ZABBIX_VERSION} --depth 1 https://github.com/zabbix/zabbix.git ~/zabbix && \
    cd ~/zabbix/ && \
    ./bootstrap.sh 1>/dev/null && \
    ./configure --enable-agent 1>/dev/null && \
    cp -R ~/zabbix-docker-monitoring/src/modules/zabbix_module_docker/ ~/zabbix/src/modules/ && \
    cd ~/zabbix/src/modules/zabbix_module_docker && \
    make

# Usage examples:
#
# Build for Zabbix 7.0 LTS (default):
#   docker build --rm -t zabbix-docker-module:debian12-7.0 .
#
# Build for Zabbix 7.4:
#   docker build --rm --build-arg ZABBIX_VERSION=release/7.4 -t zabbix-docker-module:debian12-7.4 .
#
# Extract compiled module:
#   docker run --rm -v /tmp:/output zabbix-docker-module:debian12-7.0 \
#       cp /root/zabbix/src/modules/zabbix_module_docker/zabbix_module_docker.so /output/
#
# Then use /tmp/zabbix_module_docker.so with your Zabbix agent
