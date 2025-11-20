# Zabbix Docker monitoring module (run-as-daemon fork of monitoringartist/zabbix-docker-monitoring)

C loadable module + templates for monitoring Docker and container environments with Zabbix.

[English] | [Русский](README.ru.md)

---

## Overview

This project provides a high-performance **Zabbix Docker module** (C loadable module for Zabbix agent) combined with ready-to-use Zabbix templates and Docker integration for comprehensive container monitoring. It enables native collection of Docker container metrics—CPU, memory, block I/O, network, and status information—directly into Zabbix without the overhead of external scripts.

**This is a fork of [monitoringartist/zabbix-docker-monitoring](https://github.com/monitoringartist/zabbix-docker-monitoring)**, maintained and adapted by **Ranas Mukminov** (run-as-daemon.ru) for production use in enterprise Zabbix environments.

This project was originally built for older Zabbix releases but is now intended for use with currently supported Zabbix branches (see [Supported Zabbix Versions](#supported-zabbix--os-updated-november-2025) below).

Key benefits:
- **Native performance**: Implemented as a compiled C module, delivering ~10x faster metric collection compared to UserParameter scripts
- **Container platform support**: Works with Docker, Kubernetes, Mesos/Marathon/Chronos, Docker Swarm, LXC/LXD, and Systemd-based container environments
- **Production-ready**: Battle-tested templates and pre-built binaries for major Linux distributions and Zabbix versions
- **Easy deployment**: Available as part of the Dockbix agent XXL Docker image for rapid setup

## Features

- **Comprehensive container metrics**
  - CPU usage (system, user, total, throttling)
  - Memory consumption (RSS, cache, swap, limits)
  - Block I/O statistics (read/write operations, bytes, service time)
  - Network I/O (bytes, packets, errors, drops)
  - Container uptime and state tracking

- **Low-Level Discovery (LLD)**
  - Automatic discovery of running containers
  - Published container port discovery
  - Dynamic inventory updates

- **Container inventory and metadata**
  - Container names, IDs, and human-readable identifiers
  - IP addresses and network configuration
  - Environment variables and labels
  - Image information

- **Aggregate status metrics**
  - Count of containers by status (running, exited, crashed, paused)
  - Count of images by status (all, dangling)
  - Count of volumes by status (all, dangling)

- **Direct Docker API integration**
  - Access to `docker info` system information
  - `docker inspect` for detailed container metadata
  - `docker stats` for real-time resource usage

- **High performance**
  - Native C implementation eliminates script execution overhead
  - Optimized cgroup filesystem reads
  - Minimal impact on monitored systems

- **Ready-to-use Zabbix templates**
  - Standard template with passive checks
  - Active checks variant for distributed monitoring
  - Specialized Mesos/Marathon/Chronos template

- **Ecosystem integration**
  - Compatible with Dockbix agent XXL Docker image
  - Grafana dashboard support via Zabbix datasource
  - SELinux policy module included

## Architecture

### Zabbix Docker Module

The core of this project is `zabbix_module_docker.so`, a loadable module for the Zabbix agent. The module:
- Plugs directly into the Zabbix agent process at runtime
- Exposes Docker metrics through custom item keys (prefixed with `docker.*`)
- Reads container metrics from the cgroup filesystem for maximum performance
- Communicates with Docker via the Unix socket API for metadata and discovery
- Requires minimal configuration—just load the module and use the templates

### Templates

Pre-configured Zabbix templates provide:
- Item prototypes using LLD for automatic container monitoring
- Graphs for visualizing CPU, memory, I/O, and network metrics
- Triggers for alerting on container state changes and resource thresholds
- Macros for easy customization

### Docker Integration

The module can be deployed in two ways:
1. **Native installation**: Install the module on hosts running standard Zabbix agent
2. **Containerized**: Use the Dockbix agent XXL Docker image with the module pre-installed

```
┌─────────────┐
│ Docker Host │
│             │
│  Container  │ ──┐
│  Container  │   │ cgroups
│  Container  │   │ metrics
└─────────────┘   │
                  ▼
         ┌───────────────┐       ┌──────────────┐       ┌─────────┐
         │ Zabbix Agent  │──────▶│ Zabbix Server│──────▶│ Grafana │
         │  + docker.so  │       │              │       │         │
         └───────────────┘       └──────────────┘       └─────────┘
```

## Quick Start

### Use with Dockbix Agent XXL

The fastest way to start monitoring Docker containers is using the Dockbix agent XXL Docker image:

```bash
docker run \
  --name=dockbix-agent-xxl \
  --net=host \
  --privileged \
  -v /:/rootfs \
  -v /var/run:/var/run \
  --restart unless-stopped \
  -e "ZA_Server=<ZABBIX_SERVER_IP>" \
  -e "ZA_ServerActive=<ZABBIX_SERVER_IP>" \
  -d monitoringartist/dockbix-agent-xxl-limited:latest
```

Replace `<ZABBIX_SERVER_IP>` with your Zabbix server address.

Then in Zabbix:
1. Import the [Zabbix-Template-App-Docker.xml](https://raw.githubusercontent.com/ranas-mukminov/zabbix-docker-monitoring/master/template/Zabbix-Template-App-Docker.xml) template
2. Create or configure a host representing your Docker node
3. Link the "Template App Docker" to the host
4. Wait for discovery and metric collection to begin

For more details, see [Dockbix agent XXL documentation](https://github.com/monitoringartist/dockbix-agent-xxl).

### Manual Installation (Native Zabbix Agent)

For existing Zabbix agent installations:

1. **Download the module** for your OS and Zabbix version from the [prebuilt binaries table](#compatibility)
   
   Example for CentOS 7 with Zabbix 6.0:
   ```bash
   wget https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/6.0/zabbix_module_docker.so -O /usr/lib/zabbix/modules/zabbix_module_docker.so
   ```

2. **Configure Zabbix agent** to load the module:
   
   Edit `/etc/zabbix/zabbix_agentd.conf`:
   ```ini
   LoadModulePath=/usr/lib/zabbix/modules
   LoadModule=zabbix_module_docker.so
   ```

3. **Set Docker permissions** (see [Additional Docker Permissions](#additional-docker-permissions)):
   ```bash
   usermod -aG docker zabbix
   ```

4. **Restart Zabbix agent**:
   ```bash
   systemctl restart zabbix-agent
   ```

5. **Import the template** in Zabbix UI and link it to your Docker host

6. **Verify** the module is loaded:
   ```bash
   zabbix_agentd -p | grep docker.modver
   ```

If you need to compile the module for an unsupported platform, see the [Compilation](#compilation) section.

## Templates

Three Zabbix templates are available in the `template/` directory:

### Zabbix-Template-App-Docker.xml (Standard)

**Recommended** for most use cases. Features:
- Passive checks (Zabbix server polls the agent)
- Automatic container discovery every 60 seconds
- CPU, memory, block I/O, network metrics
- Container state monitoring and triggers
- Pre-configured graphs

**Use when:** You have standard Zabbix agent in passive mode.

### Zabbix-Template-App-Docker-active.xml (Active Checks)

Similar to the standard template but uses active checks where the agent pushes data to the server.

**Use when:** You have distributed or NAT'd environments where agents need to initiate connections.

### Zabbix-Template-App-Docker-Mesos-Marathon-Chronos.xml

Specialized template for **Mesos cluster** monitoring with Marathon and Chronos schedulers. Includes:
- Enhanced container discovery using Mesos task IDs
- Application-specific macros
- Mesos/Marathon naming conventions

**Use when:** Monitoring Docker containers orchestrated by Mesos/Marathon/Chronos.

### Template Import

You can import templates manually via Zabbix UI, or use the automated Docker image:

```bash
docker run --rm \
  -e XXL_apiurl=http://your-zabbix-server/zabbix \
  -e XXL_apiuser=Admin \
  -e XXL_apipass=zabbix \
  monitoringartist/zabbix-templates
```

## Available Metrics

The module provides comprehensive Docker metrics through various item key families:

### Discovery Keys

| Key | Description |
|-----|-------------|
| `docker.discovery[<par1>,<par2>,<par3>]` | **Container discovery** (LLD). Discovers all running containers. Optional parameters allow filtering and custom naming using docker.inspect data. Returns macros: `{#FCONTAINERID}` (full 64-char ID), `{#SCONTAINERID}` (short 12-char ID), `{#HCONTAINERID}` (human name), `{#SYSTEM.HOSTNAME}` (system hostname). Example: `docker.discovery[Config,Env,MESOS_TASK_ID=]` for Mesos containers. |
| `docker.port.discovery[cid,<protocol>]` | **Port discovery** (LLD). Discovers published container ports. Protocol: `tcp`, `udp`, or `all` (default). |

### Resource Metrics

| Key | Description |
|-----|-------------|
| `docker.cpu[cid,cmetric]` | **CPU metrics** from cpuacct.stat and cpu.stat. Metrics: `system`, `user`, `total`, `nr_throttled`, `throttled_time`. Example: `docker.cpu[cid,user]`. Note: Use Delta (speed per second) in Zabbix preprocessing for utilization percentage. |
| `docker.mem[cid,mmetric]` | **Memory metrics** from memory.stat. Metrics: `cache`, `rss`, `mapped_file`, `swap`, `pgfault`, `pgmajfault`, `inactive_anon`, `active_anon`, `total_rss`, `hierarchical_memory_limit`, and more. Example: `docker.mem[cid,rss]`. Requires memory cgroup enabled (`cgroup_enable=memory` kernel parameter). |
| `docker.dev[cid,bfile,bmetric]` | **Block I/O metrics** from blkio cgroup files. Examples: `docker.dev[cid,blkio.io_service_bytes,Total]`, `docker.dev[cid,blkio.io_serviced,'8:0 Sync']`. Some metrics require `CONFIG_DEBUG_BLK_CGROUP=y` kernel config. |
| `docker.xnet[cid,interface,nmetric]` | **Network metrics** (experimental, requires root). Interface: network interface name or `all` for sum. Metrics: `MTU`, `RX-OK`, `RX-ERR`, `RX-DRP`, `TX-OK`, `TX-ERR`, etc. Example: `docker.xnet[cid,eth0,TX-OK]`. Requires `AllowRoot=1` and netstat installed. |

### Container Metadata

| Key | Description |
|-----|-------------|
| `docker.inspect[cid,par1,<par2>,<par3>]` | **Docker inspect** data. Returns values from Docker inspect JSON (e.g., [API v1.21](https://docs.docker.com/engine/api/v1.21/#inspect-a-container)). Parameters: 1st/2nd/3rd level JSON property names or array selectors. Examples: `docker.inspect[cid,Config,Image]`, `docker.inspect[cid,NetworkSettings,IPAddress]`, `docker.inspect[cid,Name]`, `docker.inspect[cid,Config,Env,MESOS_TASK_ID=]`. Returns plain text/numeric values only. Requires additional Docker permissions. |
| `docker.stats[cid,par1,<par2>,<par3>]` | **Docker stats** real-time resource usage. Requires Docker 1.5+. Returns values from stats JSON (e.g., [API v1.21](https://docs.docker.com/engine/api/v1.21/#get-container-stats-based-on-resource-usage)). Examples: `docker.stats[cid,memory_stats,usage]`, `docker.stats[cid,network,rx_bytes]`, `docker.stats[cid,cpu_stats,cpu_usage,total_usage]`. Most accurate but slowest method (0.3-0.7s per call). Requires additional Docker permissions. |
| `docker.info[info]` | **Docker system information**. Returns values from docker info JSON (e.g., [API v1.21](https://docs.docker.com/engine/api/v1.21/#display-system-wide-information)). Examples: `docker.info[Containers]`, `docker.info[Images]`, `docker.info[NCPU]`. Requires additional Docker permissions. |

### Status and Count Metrics

| Key | Description |
|-----|-------------|
| `docker.cstatus[status]` | **Container count by status**. Status values: `All` (all containers), `Up` (running, includes paused), `Exited` (stopped), `Crashed` (exit code ≠ 0), `Paused`. Requires additional Docker permissions. |
| `docker.istatus[status]` | **Image count by status**. Status values: `All` (all images), `Dangling` (untagged images). Requires additional Docker permissions. |
| `docker.vstatus[status]` | **Volume count by status**. Status values: `All` (all volumes), `Dangling` (unused volumes). Requires Docker API v1.21+ and additional Docker permissions. |
| `docker.up[cid]` | **Container running state**. Returns `1` if container is running, `0` otherwise. Fast check using cgroup filesystem. |
| `docker.modver` | **Module version**. Returns the version of the loaded zabbix_module_docker module. |

### Container Identifier (cid)

The `cid` parameter accepts:
- **Full container ID** (64 characters): `{#FCONTAINERID}` macro from discovery
- **Short container ID** (12 characters): `{#SCONTAINERID}` macro—must be prefixed with `/`, e.g., `/2599a1d88f75`
- **Human name**: `{#HCONTAINERID}` macro—must be prefixed with `/`, e.g., `/zabbix-server`

Note: Human names and short IDs require additional Docker permissions.

## Grafana Dashboard

A custom **Grafana dashboard** for Docker monitoring is available when using this Zabbix module. The dashboard provides:
- Real-time container resource utilization
- Historical trend analysis
- Multi-container comparison views
- Integration with Zabbix datasource plugin

![Grafana Docker Dashboard](https://raw.githubusercontent.com/monitoringartist/grafana-zabbix-dashboards/master/overview-docker/overview-docker.png)

Access the dashboard in the [Grafana Zabbix Dashboards repository](https://github.com/monitoringartist/grafana-zabbix-dashboards).

**Note:** Grafana integration is optional. The module works fully within Zabbix alone; Grafana provides enhanced visualization capabilities.

## Supported Zabbix / OS (Updated: November 2025)

### Recommended Zabbix Versions

This module is recommended for use with currently supported Zabbix branches:

- **Zabbix 7.4** (latest stable)
- **Zabbix 7.0 LTS** (long-term support)
- **Zabbix 6.0 LTS** (long-term support)
- Zabbix 7.2 (standard release, optional)

**Legacy versions** (4.0, 5.0, 5.4) are out of official support and not recommended for production use. For current version lifecycle information, refer to:
- https://www.zabbix.com/download
- https://endoflife.date/zabbix

Pre-built binaries are available for Zabbix agent versions: **4.0, 5.0, 5.4, 6.0** (see table below). For Zabbix 7.0 and 7.4, you can compile the module from source (see [Compilation](#compilation) section).

### Recommended Linux Distributions

Typical Linux targets tested with this module:
- **Debian 12** (Bookworm)
- **Ubuntu 22.04 LTS** (Jammy) / **Ubuntu 24.04 LTS** (Noble)
- **Rocky Linux 9** / **AlmaLinux 9** / **RHEL 9**
- **Docker Engine 23.x / 24.x**
- **Docker Compose v2**

For exact supported OS and database combinations, check official Zabbix documentation:
- https://www.zabbix.com/download
- https://www.zabbix.com/requirements

### Pre-built Binary Downloads

| Distribution | Zabbix 6.0 | Zabbix 5.4 | Zabbix 5.0 | Zabbix 4.0 |
|--------------|:----------:|:----------:|:----------:|:----------:|
| Amazon Linux 2 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux2/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux2/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux2/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux2/4.0/zabbix_module_docker.so) |
| Amazon Linux 1 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux1/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux1/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux1/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/amazonlinux1/4.0/zabbix_module_docker.so) |
| CentOS 7 / RHEL 7 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/centos7/4.0/zabbix_module_docker.so) |
| Debian 11 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian11/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian11/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian11/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian11/4.0/zabbix_module_docker.so) |
| Debian 10 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian10/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian10/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian10/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian10/4.0/zabbix_module_docker.so) |
| Debian 9 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian9/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian9/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian9/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/debian9/4.0/zabbix_module_docker.so) |
| Fedora 35 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora35/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora35/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora35/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora35/4.0/zabbix_module_docker.so) |
| Fedora 34 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora34/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora34/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora34/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora34/4.0/zabbix_module_docker.so) |
| Fedora 33 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora33/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora33/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora33/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/fedora33/4.0/zabbix_module_docker.so) |
| openSUSE 15 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse15/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse15/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse15/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse15/4.0/zabbix_module_docker.so) |
| openSUSE 42 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse42/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse42/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse42/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/opensuse42/4.0/zabbix_module_docker.so) |
| Ubuntu 20.04 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu20/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu20/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu20/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu20/4.0/zabbix_module_docker.so) |
| Ubuntu 18.04 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu18/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu18/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu18/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu18/4.0/zabbix_module_docker.so) |
| Ubuntu 16.04 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu16/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu16/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu16/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu16/4.0/zabbix_module_docker.so) |
| Ubuntu 14.04 | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu14/6.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu14/5.4/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu14/5.0/zabbix_module_docker.so) | [Download](https://github.com/monitoringartist/zabbix-docker-monitoring/raw/gh-pages/ubuntu14/4.0/zabbix_module_docker.so) |

**Note:** This fork's CI now builds and tests against Zabbix **7.4**, **7.2**, **7.0 LTS**, and **6.0 LTS**. Pre-built binaries for these modern versions will be available soon. For older versions (4.0-5.4) or distributions not listed, you can compile the module from source. See [Compilation](#compilation).

## Additional Docker Permissions

The module requires access to Docker's Unix socket to use certain features (discovery, inspect, info, stats). Choose one of these options:

### Option 1: Add Zabbix User to Docker Group (Recommended)

```bash
usermod -aG docker zabbix
# Restart Zabbix agent for group changes to take effect
systemctl restart zabbix-agent
```

### Option 2: Run Zabbix Agent as Root

Edit `/etc/zabbix/zabbix_agentd.conf`:
```ini
AllowRoot=1
```

**Note:** This is required when using Docker from RHEL/CentOS repositories.

### SELinux Configuration

If SELinux is in enforcing mode (`getenforce`), install the provided SELinux policy module:

```bash
wget https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/selinux/zabbix-docker.te
checkmodule -M -m -o zabbix-docker.mod zabbix-docker.te
semodule_package -o zabbix-docker.pp -m zabbix-docker.mod
semodule -i zabbix-docker.pp
```

This policy persists across reboots and allows Zabbix agent to access Docker.

## Container Log Monitoring

Standard [Zabbix log monitoring](https://www.zabbix.com/documentation/current/en/manual/config/items/itemtypes/log_items) can be used to monitor container logs. Docker logs stdout/stderr to:

```
/var/lib/docker/containers/<FULL_CONTAINER_ID>/<FULL_CONTAINER_ID>-json.log
```

Use this Zabbix log item key to parse JSON-formatted Docker logs:

```
log[/var/lib/docker/containers/{#FCONTAINERID}/{#FCONTAINERID}-json.log,"\"log\":\"(.*)\",\"stream",,,skip,\1]
```

**Tip:** To make applications log to stdout/stderr, symlink log files:
```bash
ln -sf /dev/stdout /var/log/nginx/access.log
ln -sf /dev/stderr /var/log/nginx/error.log
```

Use LLD macro `{#FCONTAINERID}` for automatic discovery-based log monitoring.

## Troubleshooting

### Module Fails to Load

**Symptom:** Zabbix agent log shows module loading errors

**Solutions:**
- Verify module path in `zabbix_agentd.conf` matches the file location
- Check module file permissions: `chmod 644 /path/to/zabbix_module_docker.so`
- Ensure correct binary for your OS/Zabbix version
- Check Zabbix agent log: `tail -f /var/log/zabbix/zabbix_agentd.log`
- Verify dependencies: `ldd /path/to/zabbix_module_docker.so`

### Docker Keys Return "Unsupported"

**Symptom:** Items using `docker.*` keys show "unsupported" state

**Solutions:**
- Verify module is loaded: `zabbix_agentd -p | grep docker.modver`
- Check Docker permissions (see [Additional Docker Permissions](#additional-docker-permissions))
- Test Docker socket access: `sudo -u zabbix docker ps` (should work)
- Verify Docker is running: `systemctl status docker`
- Check cgroup mount: `mount | grep cgroup`

### Missing Cgroup or Docker Permissions

**Symptom:** Some metrics return empty or zero values

**Solutions:**
- Enable memory cgroup: Add `cgroup_enable=memory` to kernel boot parameters (in `/etc/default/grub`), then `update-grub` and reboot
- Verify cgroup filesystem: `ls -la /sys/fs/cgroup/`
- Add zabbix user to docker group: `usermod -aG docker zabbix`
- For SELinux systems, install the SELinux policy module (see above)

### docker.stats Performance Issues

**Symptom:** Agent timeout or high CPU usage when using `docker.stats` keys

**Explanation:** `docker.stats` streams real-time data from Docker API (0.3-0.7s per call), making it slower than cgroup-based metrics.

**Solutions:**
- Use cgroup-based keys (`docker.cpu`, `docker.mem`, `docker.dev`) when possible—they're ~10x faster
- Increase Zabbix agent `Timeout` setting in `zabbix_agentd.conf`: `Timeout=10`
- Reduce polling frequency for `docker.stats` items
- Consider using passive checks with longer intervals

### Debugging

Enable debug logging in `/etc/zabbix/zabbix_agentd.conf`:
```ini
DebugLevel=4
```

Restart Zabbix agent and check logs:
```bash
systemctl restart zabbix-agent
tail -f /var/log/zabbix/zabbix_agentd.log
```

Module debug messages will include `[Docker]` prefix.

## Relation to Official "Docker by Zabbix agent 2" Template

Modern Zabbix releases (6.0+) ship with the official **"Docker by Zabbix agent 2"** template, which provides Docker monitoring capabilities using:
- **Zabbix agent 2** (Go-based agent) with a built-in Docker plugin
- No external scripts or modules required
- Direct Docker API integration

**Official integration page:** https://www.zabbix.com/integrations/docker

### How This Module Compares

This repository provides an **alternative Docker monitoring approach** via an external C module for the classic Zabbix agent (C agent). Consider using this module when:

1. **You need additional or custom metrics** not available in the official template
2. **You already use this module** in existing production environments and want to continue with it
3. **You are migrating from legacy setups** and need compatibility with older Zabbix agent infrastructure
4. **You prefer cgroup-based metrics** which offer ~10x faster collection compared to API-based methods
5. **You work with specialized container platforms** like Mesos/Marathon/Chronos with specific templates

For new Zabbix deployments (especially with Zabbix 7.0+), evaluate whether the official "Docker by Zabbix agent 2" template meets your needs first. Both approaches are valid and can coexist in different parts of your infrastructure.

## Production Recommendations

When deploying this module in production environments, follow these best practices:

### Discovery and Item Frequency
- Set appropriate discovery intervals (default: 60 seconds) based on container churn rate
- For high-churn environments (frequent container creation/deletion), increase the interval to reduce load
- Use macros to customize polling intervals per host or template

### Performance Optimization
- Prefer **cgroup-based metrics** (`docker.cpu`, `docker.mem`, `docker.dev`) over `docker.stats` for better performance
- `docker.stats` calls take 0.3-0.7s each; use them sparingly or increase item intervals
- Monitor Zabbix agent CPU and memory usage; adjust item intervals if the agent consumes excessive resources

### Security
- **Restrict Docker socket access**: Only grant the `zabbix` user access to the Docker group (avoid running agent as root when possible)
- **SELinux/AppArmor**: Use the provided SELinux policy module or create appropriate AppArmor profiles
- **Network isolation**: Keep Zabbix server and Docker hosts on separate networks with firewall rules
- **Audit container labels**: Filter sensitive containers from discovery if needed

### Architecture
- **Separate monitoring**: Run Zabbix server and database on dedicated infrastructure, not on monitored Docker hosts
- **Agent placement**: Deploy Zabbix agent on the Docker host (not inside containers) for accurate cgroup access
- **Resource limits**: Set memory/CPU limits for containerized Zabbix agents (Dockbix agent XXL) to prevent resource exhaustion

### Monitoring Scope
- **Template assignment**: Link Docker templates only to hosts running Docker; avoid linking to non-Docker hosts
- **Selective monitoring**: Use discovery filters to monitor only critical containers (avoid monitoring all system containers)
- **Log monitoring**: Implement log monitoring selectively; Docker JSON logs can grow large

### High Availability
- Deploy multiple Zabbix proxies in distributed environments for failover
- Use active checks for agents behind NAT or in dynamic environments
- Monitor Zabbix agent availability with separate "ping" items

### Maintenance
- **Module updates**: Test module updates in staging before production deployment
- **Zabbix upgrades**: When upgrading Zabbix, recompile the module against the new Zabbix version
- **Template updates**: Review template changes before importing; backup existing templates

## FAQ

### How do I check if the module is loaded?

Run this command on the host where Zabbix agent is running:

```bash
zabbix_agentd -p | grep docker.modver
```

If the module is loaded, you'll see output like:
```
docker.modver                                 [s|0.5]
```

Alternatively, check the Zabbix agent log (`/var/log/zabbix/zabbix_agentd.log`) for module loading messages.

### Why are no containers discovered?

Common causes and solutions:

1. **Docker permissions**: The `zabbix` user must have access to the Docker socket
   ```bash
   # Add zabbix to docker group
   sudo usermod -aG docker zabbix
   sudo systemctl restart zabbix-agent

   # Verify access
   sudo -u zabbix docker ps
   ```

2. **Module not loaded**: Verify the module is loaded (see question above)

3. **No running containers**: Discovery only finds running containers; start at least one container

4. **Discovery rule disabled**: Check that the discovery rule "Docker containers" is enabled in the linked template

5. **Discovery interval**: Wait for the next discovery cycle (default: 60 seconds)

### Can I use this together with the official "Docker by Zabbix agent 2" template?

Yes, but not on the same host with the same agent. You have two options:

1. **Different hosts**: Use this module (with Zabbix agent C) on some Docker hosts and the official template (with Zabbix agent 2) on other hosts
2. **Different metrics**: If you run both agents on the same host (possible but uncommon), configure different ports and use different templates

In most cases, choose one approach per host to avoid duplicate metrics and confusion.

### How do I monitor containers in Kubernetes?

This module can monitor Docker containers on Kubernetes nodes:

1. **Node-level deployment**: Run Zabbix agent (with this module) directly on each Kubernetes worker node (not as a pod)
2. **DaemonSet approach** (advanced): Deploy Zabbix agent as a Kubernetes DaemonSet with:
   - `hostNetwork: true`
   - `privileged: true`
   - Volume mount for `/var/run/docker.sock` (if using Docker runtime)
   - Volume mount for `/sys/fs/cgroup`

**Note**: If your Kubernetes cluster uses containerd or CRI-O (instead of Docker), this module won't work directly. Consider using Kubernetes-native monitoring tools or Zabbix's Kubernetes templates instead.

### Which metrics are fastest to collect?

Performance ranking (fastest to slowest):

1. **`docker.up[cid]`** - Fastest (simple cgroup check)
2. **`docker.cpu[cid,metric]`** - Fast (cgroup read)
3. **`docker.mem[cid,metric]`** - Fast (cgroup read)
4. **`docker.dev[cid,file,metric]`** - Fast (cgroup read)
5. **`docker.discovery`** - Medium (Docker API call)
6. **`docker.inspect[cid,...]`** - Medium (Docker API call)
7. **`docker.stats[cid,...]`** - Slow (0.3-0.7s per container, real-time streaming)

For best performance, prioritize cgroup-based metrics over `docker.stats`.

### How do I upgrade to a newer Zabbix version?

When upgrading Zabbix:

1. **Upgrade Zabbix server and database** first (follow official Zabbix upgrade guide)
2. **Upgrade Zabbix agent** on Docker hosts
3. **Recompile the module** for the new Zabbix version:
   - Download the new Zabbix source (matching your new version)
   - Compile the module following the [Compilation](#compilation) section
4. **Replace the old module** with the new `.so` file
5. **Restart Zabbix agent**:
   ```bash
   sudo systemctl restart zabbix-agent
   ```
6. **Verify** the module loads correctly

**Pre-built binaries** may not be available for the latest Zabbix versions immediately; be prepared to compile from source.

### What if I get "Unsupported item key" errors?

Causes and solutions:

1. **Module not loaded**: Verify with `zabbix_agentd -p | grep docker`
2. **Wrong Zabbix version**: Ensure module is compiled for your exact Zabbix version
3. **Typo in item key**: Check item key syntax (e.g., `docker.cpu[cid,user]` not `docker.cpu[cid user]`)
4. **Container ID format**: For short IDs and names, prefix with `/` (e.g., `/zabbix-server`)

### Does this work with Podman?

Partially. Podman provides Docker-compatible socket API, so some features may work:

1. **Enable Podman socket**:
   ```bash
   systemctl enable --now podman.socket
   ```
2. **Create Docker socket symlink**:
   ```bash
   ln -s /run/podman/podman.sock /var/run/docker.sock
   ```

However, **cgroup paths differ** between Docker and Podman, so cgroup-based metrics may not work correctly. This module is primarily designed for Docker; Podman support is experimental.

## Compilation

If pre-built binaries don't work for your system, compile the module from source.

### Prerequisites

Install required packages:

**CentOS/RHEL:**
```bash
yum install -y wget autoconf automake gcc git pcre-devel jansson-devel
```

**Debian/Ubuntu:**
```bash
apt-get install -y wget autoconf automake gcc git make pkg-config libpcre3-dev libjansson-dev
```

**Fedora:**
```bash
dnf install -y wget autoconf automake gcc git make pcre-devel jansson-devel
```

**openSUSE:**
```bash
zypper install -y wget autoconf automake gcc git make pkg-config pcre-devel libjansson-devel
```

### Build Steps

```bash
# Clone Zabbix source (match your Zabbix version)
git clone -b 6.0.0 --depth 1 https://github.com/zabbix/zabbix.git /usr/src/zabbix
cd /usr/src/zabbix

# Configure Zabbix
./bootstrap.sh
./configure --enable-agent

# Prepare module directory
mkdir -p src/modules/zabbix_module_docker
cd src/modules/zabbix_module_docker

# Download module source
wget https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/src/modules/zabbix_module_docker/zabbix_module_docker.c
wget https://raw.githubusercontent.com/monitoringartist/zabbix-docker-monitoring/master/src/modules/zabbix_module_docker/Makefile

# Compile
make
```

Output: `zabbix_module_docker.so` (dynamically linked shared object library)

### Docker-based Compilation

Alternatively, use Docker for compilation. Example Dockerfiles for various distributions are available in the [`dockerfiles/` directory](https://github.com/monitoringartist/zabbix-docker-monitoring/tree/master/dockerfiles).

Example:
```bash
cd dockerfiles/centos
docker build -t zabbix-docker-module-builder .
docker run --rm -v $(pwd):/output zabbix-docker-module-builder
```

## Development

### Repository Structure

```
zabbix-docker-monitoring/
├── src/modules/zabbix_module_docker/  # C module source code
│   ├── zabbix_module_docker.c         # Main module implementation
│   └── Makefile                        # Build configuration
├── template/                           # Zabbix templates (XML)
│   ├── Zabbix-Template-App-Docker.xml
│   ├── Zabbix-Template-App-Docker-active.xml
│   └── Zabbix-Template-App-Docker-Mesos-Marathon-Chronos.xml
├── dockerfiles/                        # Dockerfiles for building binaries
│   ├── amazonlinux/
│   ├── centos/
│   ├── debian/
│   ├── fedora/
│   ├── opensuse/
│   └── ubuntu/
├── doc/                                # Documentation and images
├── selinux/                            # SELinux policy module
└── LICENSE                             # GPL-2.0 license
```

### Building the Module

See [Compilation](#compilation) for detailed build instructions.

### Dependencies

- **Zabbix agent headers** (from Zabbix source)
- **PCRE library** (libpcre) for regular expressions
- **Jansson library** (libjansson) for JSON parsing
- **C compiler** (gcc, clang)
- **Make**

### Contributing

Contributions are welcome! Please:
1. Fork this repository
2. Create a feature branch
3. Test your changes thoroughly
4. Submit a pull request with clear description

For bugs and feature requests, use the [GitHub issue tracker](https://github.com/ranas-mukminov/zabbix-docker-monitoring/issues).

## Support and Services by run-as-daemon.ru

This fork is maintained by **Ranas Mukminov** (run-as-daemon.ru) and actively used in production Zabbix environments for monitoring Docker and Kubernetes infrastructure.

If you need help with production Zabbix/Docker/Grafana monitoring, high availability design, backup strategy, or incident response, you can reach out via: **[run-as-daemon.ru](https://run-as-daemon.ru)**

Services offered:
- Zabbix deployment and configuration consulting
- Docker/Kubernetes monitoring solutions
- Grafana dashboard development
- Performance optimization and troubleshooting
- Custom module development and integration

## License

This project is licensed under the **GNU General Public License v2.0 (GPL-2.0)**.

Original project: [monitoringartist/zabbix-docker-monitoring](https://github.com/monitoringartist/zabbix-docker-monitoring) by Monitoring Artist

Fork maintained by: Ranas Mukminov ([run-as-daemon.ru](https://run-as-daemon.ru))

See [LICENSE](LICENSE) file for full license text.

---

## Related Resources

- [Official Zabbix Documentation](https://www.zabbix.com/documentation)
- [Docker API Reference](https://docs.docker.com/engine/api/)
- [Cgroup Documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/)
- [Dockbix Agent XXL](https://github.com/monitoringartist/dockbix-agent-xxl)
- [Grafana Zabbix Plugin](https://github.com/alexanderzobnin/grafana-zabbix)
- [Zabbix Built-in Docker Monitoring (Agent2)](https://www.zabbix.com/integrations/docker)

---

**Note:** This module complements Zabbix agent (C agent). For Zabbix agent2 (Go-based agent), consider using the [built-in Docker plugin](https://www.zabbix.com/integrations/docker).
