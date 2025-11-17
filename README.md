# Zabbix Docker Monitoring (fork by run-as-daemon.ru)

Zabbix module, templates and Docker image for monitoring Docker, Kubernetes, Mesos, Swarm and other container platforms.

[English] | [Русский](README.ru.md)

---

## Overview

This project provides a high-performance **Zabbix Docker module** (C loadable module for Zabbix agent) combined with ready-to-use Zabbix templates and Docker integration for comprehensive container monitoring. It enables native collection of Docker container metrics—CPU, memory, block I/O, network, and status information—directly into Zabbix without the overhead of external scripts.

**This is a fork of [monitoringartist/zabbix-docker-monitoring](https://github.com/monitoringartist/zabbix-docker-monitoring)**, maintained and adapted by **Ranas Mukminov** for production use in enterprise Zabbix environments.

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

## Relation to Official Zabbix Docker Template

### Community Module vs Official Template

This project provides a **community-developed, high-performance C module** for Docker monitoring in Zabbix. Starting with **Zabbix 6.0**, there is also an **official "Docker by Zabbix agent 2"** template available:

- **Official Template**: https://www.zabbix.com/integrations/docker
- **Agent Type**: Zabbix agent 2 (Go-based agent)
- **Protocol**: Uses Docker API over HTTP/Unix socket

### When to Use This Module

Consider using this community module instead of (or alongside) the official template when:

1. **Performance Requirements**: This C module reads metrics directly from cgroups, making it ~10x faster than API-based collection for CPU/memory/I/O metrics
2. **Legacy Zabbix Versions**: You're running Zabbix 4.0, 5.0, or 5.4 (official template requires 6.0+)
3. **Zabbix Agent 1**: Your infrastructure uses the traditional C-based Zabbix agent rather than agent 2
4. **Existing Deployments**: You have established monitoring based on this module and templates
5. **Resource-Constrained Environments**: Lower overhead for high-density container environments
6. **Custom Metrics**: Need access to advanced cgroup metrics not exposed through standard Docker API

### When to Use the Official Template

The official Zabbix 6.0+ Docker template is better suited when:

- You're running Zabbix 6.0 or newer
- You prefer officially supported solutions with regular updates
- You're already using Zabbix agent 2 infrastructure-wide
- You want simplified setup with native agent 2 plugins

### Using Both

Both solutions can coexist:
- Use this module for **high-frequency cgroup metrics** (CPU, memory, I/O) via fast cgroup reads
- Use official template for **Docker-specific metadata** and system information via API
- Configure different update intervals based on metric criticality

For most production environments with Zabbix 6.0+, we recommend evaluating both approaches and choosing based on your specific performance requirements and existing infrastructure.

## Compatibility

### Supported Zabbix Versions

Pre-built binaries are available for Zabbix agent versions: **4.0, 5.0, 5.4, 6.0**

### Supported Linux Distributions

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

**Note:** For newer Zabbix versions or distributions not listed, you can compile the module from source. See [Compilation](#compilation).

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

## Production Recommendations

### Deployment Best Practices

When deploying this module in production environments, follow these recommendations:

#### 1. Resource Planning

- **Discovery Frequency**: Balance container discovery intervals against API load. Default 60s is suitable for most environments; consider 120-300s for large-scale deployments (100+ containers per host)
- **Item Update Intervals**: Use different intervals for different metric types:
  - Critical metrics (container up/down): 30-60s
  - Resource metrics (CPU/memory): 60-120s
  - Less critical metrics (network I/O): 120-300s
- **Zabbix Agent Timeout**: Set `Timeout=10` in `zabbix_agentd.conf` for environments with many containers

#### 2. High Availability

- **Agent Redundancy**: Deploy Zabbix agent on each Docker host independently; avoid centralized monitoring points
- **Module Updates**: Test module updates in staging before production deployment
- **Monitoring Isolation**: Keep Zabbix server/proxy separate from monitored Docker hosts to avoid cascading failures

#### 3. Performance Optimization

- **Prefer Cgroup Metrics**: Use `docker.cpu`, `docker.mem`, `docker.dev` instead of `docker.stats` whenever possible (10x faster)
- **Limit Discovery Scope**: Use discovery filters to exclude irrelevant containers (e.g., short-lived build containers)
- **Template Tuning**: Disable unused item prototypes in templates to reduce polling overhead
- **Passive vs Active**: Consider active checks for distributed environments to reduce server load

#### 4. Security Considerations

- **Docker Socket Access**: Use Docker group membership (`usermod -aG docker zabbix`) rather than running agent as root
- **SELinux**: Always use the provided SELinux policy module in enforcing mode; never disable SELinux
- **Network Isolation**: Ensure Zabbix agent firewall rules (port 10050) allow only your Zabbix servers/proxies
- **Container Isolation**: Be aware that the module reads cgroup data from the host; compromised agent could leak container metrics

#### 5. Monitoring Strategy

**For Docker Swarm/Standalone**:
- Deploy agent on each Docker node
- Monitor node-level container aggregates
- Use container discovery for per-container metrics

**For Kubernetes**:
- Deploy as DaemonSet on worker nodes
- Alternative: use sidecar pattern for namespace-specific monitoring
- Combine with Kubernetes-native monitoring (Prometheus) for cluster-level metrics
- Use this module for node-level container resource tracking

**For Mesos/Marathon**:
- Use the Mesos-specific template variant
- Enable task ID-based discovery filtering
- Monitor at both framework and task levels

#### 6. Alerting Configuration

- **Container State Changes**: Alert on unexpected transitions (running → exited)
- **Resource Thresholds**: Set memory/CPU triggers based on container limits, not host capacity
- **Discovery Failures**: Alert if container discovery returns empty results on hosts with expected containers
- **Module Health**: Monitor `docker.modver` to detect module load failures

#### 7. Capacity Planning

For sizing Zabbix infrastructure when monitoring Docker:
- **Items per container**: ~15-30 items depending on template configuration
- **Server Load**: Each container adds ~0.1% to Zabbix server CPU on average
- **Database Growth**: Expect ~1-5 MB/day/container with default retention (historical data)
- **Network Bandwidth**: Minimal (<1 Kbps per container for passive checks)

#### 8. Operational Guidelines

- **Change Management**: Document custom template modifications separately from module updates
- **Backup Strategy**: Include Zabbix templates and module configuration in backup procedures
- **Incident Response**: Keep module logs (`/var/log/zabbix/`) accessible for debugging production issues
- **Upgrade Path**: Test new Zabbix versions in lab environment before upgrading production agents

#### 9. Integration with CI/CD

- **Container Labeling**: Use Docker labels for discovery filtering (requires custom template modifications)
- **Deployment Validation**: Add Zabbix checks to deployment pipelines (verify container discovered and monitored)
- **Automated Testing**: Include module functionality in infrastructure-as-code tests

#### 10. Documentation

Maintain internal documentation covering:
- Custom template modifications and rationale
- Non-standard item key parameters
- Troubleshooting runbooks specific to your environment
- Escalation procedures for monitoring alerts

## Frequently Asked Questions (FAQ)

### General Questions

**Q: How do I check if the module is loaded correctly?**

A: Run the following command:
```bash
zabbix_agentd -p | grep docker.modver
```
You should see output like: `docker.modver [t|v0.7.0]`. If empty, check agent logs for loading errors.

**Q: Can I use this module with Zabbix agent 2?**

A: No, this module is designed for Zabbix agent 1 (C-based). For Zabbix agent 2, use the [official Docker plugin](https://www.zabbix.com/integrations/docker).

**Q: Why do I see no containers discovered?**

A: Common causes:
1. Module not loaded (check `zabbix_agentd -p | grep docker`)
2. No Docker permissions (run `sudo -u zabbix docker ps` to test)
3. No containers running (check `docker ps`)
4. Discovery rule not triggered yet (wait 60s or manually execute discovery)
5. SELinux blocking access (check `ausearch -m avc -ts recent`)

**Q: Can I monitor containers on remote Docker hosts?**

A: No, the module must run on each Docker host. Deploy Zabbix agent with the module on every host you want to monitor.

**Q: Does this work with Docker Compose?**

A: Yes, Docker Compose containers are regular Docker containers and will be discovered automatically.

**Q: Can I use this alongside the official Zabbix Docker template?**

A: Yes, both can coexist. See the [Relation to Official Zabbix Docker Template](#relation-to-official-zabbix-docker-template) section for details.

### Performance Questions

**Q: Why are some metrics slow to collect?**

A: `docker.stats` keys are slow (0.3-0.7s) because they stream real-time data from Docker API. Use cgroup-based keys (`docker.cpu`, `docker.mem`, `docker.dev`) instead—they're ~10x faster.

**Q: How many containers can I monitor per host?**

A: The module scales well. Production deployments typically handle 50-200 containers per host without issues. Key factors:
- Use cgroup metrics (not `docker.stats`)
- Tune discovery and polling intervals
- Ensure adequate Zabbix agent timeout settings

**Q: Does the module impact container performance?**

A: Minimal. Reading cgroup metrics is a lightweight filesystem operation. Typical overhead: <0.1% CPU per monitored container.

### Configuration Questions

**Q: How do I monitor specific containers only?**

A: Use discovery filters in the template. Example: `docker.discovery[Config,Env,APP_TYPE=]` discovers only containers with `APP_TYPE` environment variable.

**Q: Can I monitor container logs?**

A: Yes, using standard Zabbix log monitoring. See [Container Log Monitoring](#container-log-monitoring) section.

**Q: How do I monitor containers in different namespaces (Kubernetes)?**

A: Deploy the agent as a DaemonSet on Kubernetes nodes. Each agent monitors containers on its node, regardless of namespace.

**Q: How do I change discovery intervals?**

A: Edit the discovery rule in your Zabbix template (Configuration → Templates → Template App Docker → Discovery rules → Container discovery).

### Compatibility Questions

**Q: Does this work with Podman?**

A: Partial support. Metrics that rely on Docker socket API may not work. Cgroup-based metrics (`docker.cpu`, `docker.mem`) should work if Podman uses compatible cgroup paths.

**Q: Is cgroups v2 supported?**

A: The module primarily targets cgroups v1. Cgroups v2 support depends on Docker's cgroup driver configuration and may require code adjustments.

**Q: Can I use this with Docker Desktop (Windows/Mac)?**

A: No, the module requires Linux cgroups and Docker socket access. It's designed for Linux Docker hosts.

**Q: What's the minimum Docker version required?**

A: Docker 1.5+ for most features. Some metrics require newer versions (e.g., `docker.vstatus` needs Docker API v1.21+).

### Troubleshooting Questions

**Q: Why do memory metrics show zero?**

A: Memory cgroup is likely not enabled. Add `cgroup_enable=memory` to kernel boot parameters in `/etc/default/grub`, run `update-grub`, and reboot.

**Q: Why do I get "Unsupported" for docker.inspect/docker.stats keys?**

A: These keys require Docker socket access. Ensure:
1. Zabbix user is in docker group: `usermod -aG docker zabbix`
2. Docker socket is accessible: `ls -l /var/run/docker.sock`
3. SELinux policy is installed (if using SELinux)

**Q: Module worked before, now it doesn't after reboot. Why?**

A: Likely causes:
1. Docker service starts after Zabbix agent (add dependency)
2. Group membership not persisted (re-run `usermod -aG docker zabbix`)
3. SELinux policy not persistent (re-install policy module)

**Q: How do I enable debug logging?**

A: Set `DebugLevel=4` in `/etc/zabbix/zabbix_agentd.conf`, restart agent, and check `/var/log/zabbix/zabbix_agentd.log`. Module messages include `[Docker]` prefix.

### Kubernetes-Specific Questions

**Q: How do I deploy this in Kubernetes?**

A: Deploy as a DaemonSet with:
```yaml
hostNetwork: true
hostPID: true
volumes:
  - name: docker-sock
    hostPath: {path: /var/run/docker.sock}
  - name: cgroup
    hostPath: {path: /sys/fs/cgroup}
```
Mount these volumes in your Zabbix agent container.

**Q: Can this monitor Kubernetes pod metrics?**

A: It monitors container-level metrics. For pod-level metrics, aggregate container metrics or use Kubernetes-native monitoring (kube-state-metrics, Prometheus).

**Q: Does it work with containerd/CRI-O runtimes?**

A: Limited support. The module is optimized for Docker. For non-Docker runtimes, cgroup metrics may work but Docker API features won't.

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

## Fork Maintainer & Support

This fork is maintained by **Ranas Mukminov** and actively used in production Zabbix environments for monitoring Docker and Kubernetes infrastructure.

**Ranas Mukminov** specializes in enterprise monitoring solutions, Zabbix consulting, and DevOps automation. This fork includes additional improvements, updates, and adaptations based on real-world production experience.

For professional monitoring consulting, Zabbix implementation, or custom development: [run-as-daemon.ru](https://run-as-daemon.ru)

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
