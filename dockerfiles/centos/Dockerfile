# =============================================================================
# Dockerfile for building Zabbix Docker Module
# =============================================================================
# This Dockerfile builds the zabbix_module_docker.so loadable module for
# Zabbix agent on CentOS/RHEL-based distributions.
#
# Original project: https://github.com/monitoringartist/zabbix-docker-monitoring
# Fork maintained by: run-as-daemon.ru
#
# Usage:
#   Build: docker build --build-arg ZABBIX_VERSION=release/6.0 -t zabbix-docker-module .
#   Extract: docker run --rm -v /tmp:/output zabbix-docker-module cp /root/zabbix/src/modules/zabbix_module_docker/zabbix_module_docker.so /output/
# =============================================================================

# Use CentOS Stream 9 (CentOS 8 is EOL)
# For older versions, use: centos:7
FROM quay.io/centos/centos:stream9

# Metadata
LABEL org.opencontainers.image.title="Zabbix Docker Module Builder (CentOS)"
LABEL org.opencontainers.image.description="Build environment for Zabbix Docker monitoring module"
LABEL org.opencontainers.image.authors="Jan Garaj <info@monitoringartist.com>, run-as-daemon.ru"
LABEL org.opencontainers.image.source="https://github.com/ranas-mukminov/zabbix-docker-monitoring"
LABEL org.opencontainers.image.licenses="GPL-2.0"

WORKDIR /root

# Install build dependencies
# - git: for cloning repositories
# - Development tools: gcc, make, autoconf, automake for compilation
# - pcre-devel: PCRE library for regular expressions (required by Zabbix)
# - jansson-devel: JSON library (required by Docker module for API parsing)
RUN yum -y -q install git automake autoconf gcc make pcre-devel jansson-devel 1>/dev/null && \
    yum clean all

# Clone the zabbix-docker-monitoring repository
# This contains the module source code
RUN git clone -q https://github.com/monitoringartist/zabbix-docker-monitoring

# Define Zabbix version to build against
# Format: <version> (e.g., 6.0.0) or release/<version> (e.g., release/6.0)
# Supported versions: 4.0, 5.0, 5.4, 6.0
# Default: release/6.0 for CentOS Stream 9 compatibility
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
#   docker build --rm -t local/zabbix-docker-module-centos \
#     https://github.com/ranas-mukminov/zabbix-docker-monitoring.git#master:dockerfiles/centos/
#
# Build from local directory:
#   cd dockerfiles/centos
#   docker build --rm -t local/zabbix-docker-module-centos .
#
# Build for specific Zabbix version:
#   docker build --rm --build-arg ZABBIX_VERSION=release/5.4 \
#     -t local/zabbix-docker-module-centos .
#
# Extract compiled module:
#   docker run --rm -v /tmp:/tmp local/zabbix-docker-module-centos \
#     cp /root/zabbix/src/modules/zabbix_module_docker/zabbix_module_docker.so \
#     /tmp/zabbix_module_docker.so
#
# Cleanup:
#   docker rmi -f local/zabbix-docker-module-centos
#
# Then copy /tmp/zabbix_module_docker.so to your Zabbix agent modules directory.
