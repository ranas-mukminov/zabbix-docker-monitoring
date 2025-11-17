# Zabbix Docker monitoring module build image
# https://github.com/ranas-mukminov/zabbix-docker-monitoring
#
# This Dockerfile builds the zabbix_module_docker.so for RHEL-compatible systems.
# Target: Rocky Linux 9 (RHEL 9 compatible) with Zabbix 7.0 LTS or 7.4
#
# Note: CentOS 8 and earlier are EOL. This uses Rocky Linux 9 as a modern,
#       RHEL-compatible alternative. For AlmaLinux, change FROM to almalinux:9
#
# Build arguments:
#   ZABBIX_VERSION - Zabbix version tag or branch (default: release/7.0)
#
# Updated: November 2025 (run-as-daemon.ru fork)

# Use Rocky Linux 9 - RHEL 9 compatible (CentOS successor)
# Alternative: almalinux:9
FROM rockylinux:9-minimal

LABEL maintainer="Ranas Mukminov <https://run-as-daemon.ru>"
LABEL description="Build environment for Zabbix Docker monitoring module on Rocky Linux/RHEL"
LABEL upstream="https://github.com/monitoringartist/zabbix-docker-monitoring"

WORKDIR /root

# Install build dependencies using dnf (yum successor in RHEL 9)
# - git: clone repositories
# - automake, autoconf, gcc, make: build tools
# - pcre-devel: PCRE library for regex support
# - jansson-devel: JSON parsing library
RUN microdnf -y install dnf && \
    dnf -y -q install \
        git \
        automake \
        autoconf \
        gcc \
        make \
        pcre-devel \
        jansson-devel \
        ca-certificates \
    && \
    dnf clean all && \
    rm -rf /var/cache/dnf

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
# Build for Zabbix 7.0 LTS on Rocky Linux 9 (default):
#   docker build --rm -t zabbix-docker-module:rocky9-7.0 .
#
# Build for Zabbix 7.4 on Rocky Linux 9:
#   docker build --rm --build-arg ZABBIX_VERSION=release/7.4 -t zabbix-docker-module:rocky9-7.4 .
#
# Build for AlmaLinux 9 (change FROM line to almalinux:9):
#   docker build --rm -t zabbix-docker-module:alma9-7.0 .
#
# Extract compiled module:
#   docker run --rm -v /tmp:/output zabbix-docker-module:rocky9-7.0 \
#       cp /root/zabbix/src/modules/zabbix_module_docker/zabbix_module_docker.so /output/
#
# Then use /tmp/zabbix_module_docker.so with your Zabbix agent
