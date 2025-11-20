# Changes 0.7.1 (November 2025)

**Modernization and Multi-Version Zabbix Support**

- Added support for Zabbix 7.4, 7.2, 7.0 LTS, and 6.0 LTS
- Fixed API compatibility: standardized `zbx_strdup()` calls (2 arguments instead of 3) for modern Zabbix versions
- Updated GitHub Actions CI with build matrix for all supported Zabbix versions (7.4, 7.2, 7.0, 6.0)
- Enhanced Makefile with stricter compiler warnings (`-Werror`, `-Wformat-security`, `-Wstack-protector`)
- Added `make check` target for static code analysis with cppcheck
- Updated release build script to include Zabbix 7.0, 7.2, and 7.4 binaries
- Updated Dockerfiles: Rocky Linux 9 (RHEL 9), Debian 12, Ubuntu 24.04 LTS
- Performance improvements: added Zabbix source caching in CI builds
- Improved test coverage with module load verification

**Breaking Changes:**
- None. Module maintains backward compatibility with existing Zabbix templates and configurations.

**Maintenance Notes:**
- This is a fork maintained by Ranas Mukminov (run-as-daemon.ru)
- See implementation_plan.md for detailed modernization strategy

# Changes 0.7.0
- Zabbix JSON processing functions replaced with Jansson library, ([#152](https://github.com/monitoringartist/zabbix-docker-monitoring/pull/152), thanks to [@i-ky](https://github.com/i-ky))

# Changes 0.6.9
- new item key docker.port.discovery

# Changes 0.6.8
- fixed docker.discovery for newer kernels 4.X ([#80](https://github.com/monitoringartist/zabbix-docker-monitoring/pull/80))

# Changes 0.6.7
- fixed incorrect docker.cstatus/vstatus/istatus metric ([#74](https://github.com/monitoringartist/zabbix-docker-monitoring/issues/74))
- removed support for systemd - code has been moved to https://github.com/cavaliercoder/zabbix-module-systemd
- added timeout for Docker socket query ([#73](https://github.com/monitoringartist/zabbix-docker-monitoring/issues/73))
- new item key docker.modver

# Changes 0.6.6
- fixed incorrect CPU multiplier in the template ([#30](https://github.com/monitoringartist/zabbix-docker-monitoring/issues/30)) - update your container CPU triggers
- added total CPU utilization metric (current sum of user+system ticks)
- total CPU utilization metric is used in the templates instead of user+system CPU utilization

# Changes 0.6.5

- fixed incorrect docker.vstatus metric ([#66](https://github.com/monitoringartist/zabbix-docker-monitoring/issues/66))

# Changes 0.6.4

- added changelog ([#51](https://github.com/monitoringartist/zabbix-docker-monitoring/issues/51))
- compatibility with all zabbix module API versions 2.x/3.x ([#54](https://github.com/monitoringartist/zabbix-docker-monitoring/issues/54))
