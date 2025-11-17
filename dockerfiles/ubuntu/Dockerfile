# =============================================================================
# Dockerfile for building Zabbix Docker Module
# =============================================================================
# This Dockerfile builds the zabbix_module_docker.so loadable module for
# Zabbix agent on Ubuntu/Debian-based distributions.
#
# Original project: https://github.com/monitoringartist/zabbix-docker-monitoring
# Fork maintained by: run-as-daemon.ru
#
# Usage:
#   Build: docker build --build-arg ZABBIX_VERSION=release/6.0 -t zabbix-docker-module .
#   Extract: docker run --rm -v /tmp:/output zabbix-docker-module cp /root/zabbix/src/modules/zabbix_module_docker/zabbix_module_docker.so /output/
# =============================================================================

# Use Ubuntu 22.04 LTS (Jammy Jellyfish)
# For other versions: ubuntu:20.04, ubuntu:18.04
# Avoid 'latest' tag for reproducible builds
FROM ubuntu:22.04

# Metadata
LABEL org.opencontainers.image.title="Zabbix Docker Module Builder (Ubuntu)"
LABEL org.opencontainers.image.description="Build environment for Zabbix Docker monitoring module"
LABEL org.opencontainers.image.authors="Jan Garaj <info@monitoringartist.com>, run-as-daemon.ru"
LABEL org.opencontainers.image.source="https://github.com/ranas-mukminov/zabbix-docker-monitoring"
LABEL org.opencontainers.image.licenses="GPL-2.0"

# Prevent interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=UTC

WORKDIR /root

# Install build dependencies
# - git: for cloning repositories
# - Development tools: gcc, make, autoconf, automake, pkg-config for compilation
# - libpcre3-dev: PCRE library for regular expressions (required by Zabbix)
# - libjansson-dev: JSON library (required by Docker module for API parsing)
RUN apt-get -qq update 1>/dev/null && \
    ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone && \
    apt-get -qq --assume-yes install \
        git \
        automake \
        autoconf \
        gcc \
        make \
        pkg-config \
        libpcre3-dev \
        libjansson-dev \
        1>/dev/null && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Clone the zabbix-docker-monitoring repository
# This contains the module source code
RUN git clone -q https://github.com/monitoringartist/zabbix-docker-monitoring

# Define Zabbix version to build against
# Format: <version> (e.g., 6.0.0) or release/<version> (e.g., release/6.0)
# Supported versions: 4.0, 5.0, 5.4, 6.0
# Default: release/6.0 for Ubuntu 22.04 compatibility
ARG ZABBIX_VERSION=release/6.0

# Build the module:
# 1. Clone Zabbix source at specified version
# 2. Bootstrap and configure Zabbix agent build
# 3. Copy module source into Zabbix source tree
# 4. Compile the module using Zabbix headers
RUN git clone -b ${ZABBIX_VERSION} --depth 1 https://github.com/zabbix/zabbix.git ~/zabbix && \
    cd ~/zabbix/ && \
    ./bootstrap.sh 1>/dev/null && \
    ./configure --enable-agent 1>/dev/null && \
    cp -R ~/zabbix-docker-monitoring/src/modules/zabbix_module_docker/ ~/zabbix/src/modules/ && \
    cd ~/zabbix/src/modules/zabbix_module_docker && \
    make

# The compiled module is located at:
# /root/zabbix/src/modules/zabbix_module_docker/zabbix_module_docker.so
#
# Dockerized compilation examples:
#
# Build from GitHub (master branch):
#   docker build --rm -t local/zabbix-docker-module-ubuntu \
#     https://github.com/ranas-mukminov/zabbix-docker-monitoring.git#master:dockerfiles/ubuntu/
#
# Build from local directory:
#   cd dockerfiles/ubuntu
#   docker build --rm -t local/zabbix-docker-module-ubuntu .
#
# Build for specific Zabbix version:
#   docker build --rm --build-arg ZABBIX_VERSION=release/5.4 \
#     -t local/zabbix-docker-module-ubuntu .
#
# Build for older Ubuntu (e.g., 20.04):
#   docker build --rm --build-arg ZABBIX_VERSION=release/5.4 \
#     -f Dockerfile --build-arg BASE_IMAGE=ubuntu:20.04 \
#     -t local/zabbix-docker-module-ubuntu20 .
#
# Extract compiled module:
#   docker run --rm -v /tmp:/tmp local/zabbix-docker-module-ubuntu \
#     cp /root/zabbix/src/modules/zabbix_module_docker/zabbix_module_docker.so \
#     /tmp/zabbix_module_docker.so
#
# Cleanup:
#   docker rmi -f local/zabbix-docker-module-ubuntu
#
# Then copy /tmp/zabbix_module_docker.so to your Zabbix agent modules directory.
