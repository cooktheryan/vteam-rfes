# Research: Podman Grafana with Persistent Storage

**Feature**: 001-podman-grafana-pvc
**Date**: 2025-11-06
**Status**: Complete

## Overview

This document consolidates research findings for implementing Grafana deployment with Podman using persistent volumes. The research focuses on rootless Podman best practices, SELinux compatibility, Grafana data persistence requirements, and container lifecycle management.

---

## 1. Podman Persistent Volume Strategy

### Decision
Use **host directory bind mounts** with SELinux `:Z` flag for single-container deployments.

### Rationale
- **Simplicity**: Bind mounts provide direct visibility into data on the host filesystem, making backup and troubleshooting straightforward
- **SELinux Compatibility**: The `:Z` flag (uppercase) ensures proper private unshared labeling for container-exclusive volumes
- **Rootless Support**: Modern Podman handles UID mapping automatically for rootless containers
- **Production Ready**: Provides full control over data location and permissions without requiring named volume management

### Alternatives Considered
- **Named Volumes**: While cleaner for multi-container scenarios, bind mounts offer better transparency for documentation purposes and easier troubleshooting
- **`:z` (lowercase) flag**: Rejected because lowercase z allows volume sharing between containers, which is unnecessary for single-container Grafana deployment and reduces security isolation
- **No SELinux flag**: Would fail on RHEL/CentOS systems with enforcing SELinux due to permission denied errors

### Implementation Guidance
```bash
# Create host directory
mkdir -p ~/grafana-data

# Run with bind mount and SELinux context
podman run -d \
  -v ~/grafana-data:/var/lib/grafana:Z \
  -p 3000:3000 \
  grafana/grafana
```

**Key Points**:
- `:Z` (uppercase) sets SELinux context as `container_file_t` with private unshared label
- Rootless Podman automatically handles UID mapping between host and container
- Modern Podman (with `:U` option) can also handle ownership, but `:Z` is sufficient for basic deployments

---

## 2. Grafana Data Persistence Requirements

### Decision
Mount **`/var/lib/grafana`** as the sole persistent volume for complete Grafana data retention.

### Rationale
- **Single Directory Scope**: All Grafana persistent data (dashboards, datasources, users, settings, SQLite DB) resides in `/var/lib/grafana`
- **Official Container Image Standard**: The grafana/grafana official image is designed with this directory as the persistence point
- **Complete State Preservation**: Mounting this directory ensures 100% configuration and data survival across container lifecycle events
- **Simplified Documentation**: Single mount point reduces complexity and potential for user error

### Alternatives Considered
- **Multiple Mount Points**: Separating config, dashboards, and database would add unnecessary complexity without meaningful benefits for typical deployments
- **Configuration File Only**: Mounting only `/etc/grafana` would preserve config but lose dashboards, users, and all runtime data
- **Partial Persistence**: Selecting specific subdirectories increases fragility and risk of data loss

### Implementation Guidance
```bash
# All persistent data in one directory
-v ~/grafana-data:/var/lib/grafana:Z
```

**What Gets Persisted**:
- SQLite database (default Grafana DB)
- Dashboard definitions (JSON)
- User accounts and permissions
- Data source configurations
- Plugin data
- Session data
- Alert configurations

**Permission Requirements**:
- Container runs as UID 472 (grafana user)
- Rootless Podman handles UID mapping automatically
- Host directory permissions managed by Podman with `:Z` flag

---

## 3. Container Lifecycle Management

### Decision
Use **`podman generate systemd`** with user-level systemd services for auto-start and lifecycle management.

### Rationale
- **Native Integration**: Systemd is the standard init system on RHEL/CentOS - leveraging it provides familiar management interface
- **Auto-Start on Boot**: Systemd ensures containers start automatically after system reboot (critical for production)
- **Dependency Management**: Can define startup ordering and dependencies with other system services
- **Logging Integration**: Automatic integration with journald for centralized logging
- **No Additional Daemons**: Podman's daemon-less architecture means no background processes consuming resources when containers aren't running

### Alternatives Considered
- **Manual Restart Scripts**: Rejected due to lack of failure handling and auto-restart capabilities
- **Podman restart=always Policy**: Works for immediate restarts but doesn't handle system reboots
- **Systemd System Service (rootful)**: User-level services are preferred for security (rootless Podman)
- **External Orchestration**: Tools like Kubernetes/Docker Swarm are overkill for single-node deployments

### Implementation Guidance

**Step 1: Create and verify container**
```bash
podman run -d --name grafana \
  -v ~/grafana-data:/var/lib/grafana:Z \
  -p 3000:3000 \
  grafana/grafana
```

**Step 2: Generate systemd unit file**
```bash
# For rootless Podman (user service)
podman generate systemd --name grafana --files --new
```

**Step 3: Install and enable service**
```bash
mkdir -p ~/.config/systemd/user/
mv container-grafana.service ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable container-grafana.service
systemctl --user start container-grafana.service
```

**Step 4: Enable lingering (critical for user services)**
```bash
loginctl enable-linger $USER
```

**Key Configuration Details**:
- `--new`: Ensures systemd service creates container from scratch (handles updates cleanly)
- `--files`: Writes unit file to current directory for manual installation
- **User lingering**: Required for user services to start at boot without user login
- **Restart policy**: Default is `on-failure` - automatically restarts on crashes

**Service Management**:
```bash
# Check status
systemctl --user status container-grafana

# Stop service
systemctl --user stop container-grafana

# Restart service
systemctl --user restart container-grafana

# View logs
journalctl --user -u container-grafana
```

---

## 4. Rootless Podman Best Practices Summary

### Key Findings
1. **SELinux is non-negotiable** on RHEL/CentOS - always use `:Z` or `:z` flags
2. **UID mapping is automatic** - Podman handles container UID 472 to host UID mapping
3. **User services require lingering** - Must run `loginctl enable-linger` for boot persistence
4. **Systemd integration is the standard** - Preferred over custom restart scripts
5. **Bind mounts are transparent** - Easier for backup, monitoring, and troubleshooting vs named volumes

### Security Considerations
- Rootless Podman runs without root privileges - improved security posture
- Private SELinux labeling (`:Z`) prevents other containers from accessing volume
- User-level systemd services run under user context, not system root

### Production Readiness Checklist
- [ ] Host directory created with appropriate location for backups
- [ ] Container deployed and tested with `:Z` volume flag
- [ ] Systemd service generated and installed
- [ ] User lingering enabled for boot persistence
- [ ] Service verified to start after system reboot
- [ ] Logs accessible via `journalctl --user -u container-grafana`
- [ ] Data persistence verified through container stop/start cycle

---

## 5. Documentation Requirements

Based on research, the deployment documentation must include:

1. **Prerequisites Section**
   - Podman installation verification
   - SELinux status check
   - Disk space validation
   - Port availability check

2. **Deployment Commands**
   - Host directory creation
   - Container deployment with proper volume flags
   - Systemd service generation
   - Service installation and enablement
   - Lingering configuration

3. **Verification Steps**
   - Container status check
   - Grafana web UI access
   - Data persistence validation (create dashboard, restart container, verify persistence)
   - Systemd service status
   - Boot persistence test

4. **Troubleshooting Section**
   - SELinux permission denied errors
   - Port conflicts
   - Service startup failures
   - Data persistence issues
   - UID mapping problems

5. **Maintenance Procedures**
   - Container updates while preserving data
   - Backup procedures
   - Log management
   - Service restart procedures

---

## References

- [Podman volumes and SELinux](https://blog.christophersmart.com/2021/01/31/podman-volumes-and-selinux/)
- [Understanding Podman Volumes: Named Volumes vs Bind Mounts](https://medium.com/@ketanpradhan/understanding-podman-volumes-named-volumes-vs-bind-mounts-and-why-rootless-users-on-rhel-should-277ee89ad498)
- [Using volumes with rootless podman](https://www.tutorialworks.com/podman-rootless-volumes/)
- [Grafana Docker and data persistence](https://community.grafana.com/t/grafana-docker-and-data-persistence/33702)
- [Using Podman and systemd to manage container lifecycle](https://podman.io/blogs/2020/12/09/podman-systemd-demo.html)
- [How to Autostart Podman Containers](https://linuxhandbook.com/autostart-podman-containers/)
