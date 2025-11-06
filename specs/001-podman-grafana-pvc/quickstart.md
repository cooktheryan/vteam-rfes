# Quick Start: Podman Grafana with Persistent Storage

**Feature**: 001-podman-grafana-pvc
**Date**: 2025-11-06
**Estimated Time**: 10 minutes

This quick start guide provides the essential commands to deploy Grafana with Podman and persistent storage. For detailed explanations, troubleshooting, and maintenance procedures, refer to the complete documentation.

---

## Prerequisites Check

Run these commands to verify your system is ready:

```bash
# Check Podman installation
podman --version
# Expected: Podman version 3.0 or higher

# Check SELinux status
getenforce
# Expected: Enforcing or Permissive

# Check port availability
ss -tuln | grep :3000
# Expected: No output (port is available)

# Check disk space
df -h ~
# Expected: At least 2GB available

# Check systemd user service support
systemctl --user status
# Expected: No errors
```

**If any checks fail**, refer to the full documentation for prerequisite installation steps.

---

## Deployment (3 Steps)

### Step 1: Create Persistent Volume Directory

```bash
mkdir -p ~/grafana-data
```

### Step 2: Deploy Grafana Container

```bash
podman run -d \
  --name grafana \
  -v ~/grafana-data:/var/lib/grafana:Z \
  -p 3000:3000 \
  grafana/grafana:latest
```

**What this does**:
- `-d`: Run in background (detached)
- `--name grafana`: Name the container "grafana"
- `-v ~/grafana-data:/var/lib/grafana:Z`: Mount persistent volume with SELinux context
- `-p 3000:3000`: Expose Grafana on port 3000
- `grafana/grafana:latest`: Use official Grafana image

### Step 3: Verify Deployment

```bash
# Check container is running
podman ps --filter name=grafana

# Wait ~30 seconds, then check Grafana is accessible
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3000
# Expected: 200 or 302
```

**Access Grafana**: Open browser to `http://localhost:3000`
- Default username: `admin`
- Default password: `admin`
- You will be prompted to change the password on first login

---

## Quick Persistence Test

Verify data persists across container lifecycle:

```bash
# 1. Create a test dashboard in Grafana web UI first

# 2. Stop and remove container
podman stop grafana
podman rm grafana

# 3. Redeploy with same persistent volume
podman run -d \
  --name grafana \
  -v ~/grafana-data:/var/lib/grafana:Z \
  -p 3000:3000 \
  grafana/grafana:latest

# 4. Access Grafana UI and verify your dashboard still exists
# http://localhost:3000
```

**Success**: Your test dashboard should be visible after redeployment.

---

## Auto-Start on Boot (Optional but Recommended)

Configure systemd to automatically start Grafana on system boot:

```bash
# 1. Generate systemd service file
podman generate systemd --name grafana --files --new

# 2. Install service
mkdir -p ~/.config/systemd/user/
mv container-grafana.service ~/.config/systemd/user/

# 3. Reload systemd
systemctl --user daemon-reload

# 4. Enable and start service
systemctl --user enable container-grafana.service
systemctl --user start container-grafana.service

# 5. Enable user lingering (CRITICAL for boot persistence)
loginctl enable-linger $USER

# 6. Verify service is running
systemctl --user status container-grafana
```

**Verify boot persistence**: Reboot system and check service starts automatically:
```bash
sudo reboot
# After reboot:
systemctl --user status container-grafana
# Expected: Active: active (running)
```

---

## Common Management Commands

```bash
# View container logs
podman logs grafana

# View systemd service logs (if configured)
journalctl --user -u container-grafana -f

# Stop Grafana
systemctl --user stop container-grafana
# OR (if not using systemd):
podman stop grafana

# Start Grafana
systemctl --user start container-grafana
# OR:
podman start grafana

# Restart Grafana
systemctl --user restart container-grafana
# OR:
podman restart grafana

# Check Grafana status
systemctl --user status container-grafana
# OR:
podman ps --filter name=grafana
```

---

## Quick Troubleshooting

### Error: "permission denied" when writing to /var/lib/grafana

**Cause**: Missing `:Z` SELinux flag on volume mount

**Fix**:
```bash
# Stop and remove container
podman stop grafana && podman rm grafana

# Redeploy with :Z flag
podman run -d \
  --name grafana \
  -v ~/grafana-data:/var/lib/grafana:Z \
  -p 3000:3000 \
  grafana/grafana:latest
```

### Error: "bind: address already in use"

**Cause**: Port 3000 is already in use

**Fix**: Use a different host port:
```bash
podman run -d \
  --name grafana \
  -v ~/grafana-data:/var/lib/grafana:Z \
  -p 3001:3000 \
  grafana/grafana:latest
# Access via http://localhost:3001
```

### Container not running after reboot

**Cause**: User lingering not enabled

**Fix**:
```bash
loginctl enable-linger $USER

# Verify:
loginctl show-user $USER | grep Linger
# Expected: Linger=yes
```

### Service fails to start

**Cause**: Service file misconfigured or container already running

**Fix**:
```bash
# Stop any running container
podman stop grafana 2>/dev/null || true

# Restart service
systemctl --user restart container-grafana

# Check logs
journalctl --user -u container-grafana -n 50
```

---

## Backup and Restore

### Backup

```bash
# Stop container
systemctl --user stop container-grafana

# Backup persistent volume
tar -czf grafana-backup-$(date +%Y%m%d).tar.gz ~/grafana-data

# Start container
systemctl --user start container-grafana
```

### Restore

```bash
# Stop container
systemctl --user stop container-grafana

# Clear current data
rm -rf ~/grafana-data/*

# Restore from backup
tar -xzf grafana-backup-20251106.tar.gz -C ~/

# Start container
systemctl --user start container-grafana
```

---

## Grafana Version Upgrade

```bash
# Pull new image
podman pull grafana/grafana:latest

# Stop service (service will auto-create new container with new image)
systemctl --user stop container-grafana

# Start service
systemctl --user start container-grafana

# Verify upgrade
podman exec grafana grafana-server -v
```

**Note**: Grafana automatically migrates data schema if needed. Always backup before major version upgrades.

---

## Clean Uninstall

```bash
# Stop and disable service
systemctl --user stop container-grafana
systemctl --user disable container-grafana

# Remove service file
rm ~/.config/systemd/user/container-grafana.service
systemctl --user daemon-reload

# Remove container
podman stop grafana
podman rm grafana

# Remove image (optional)
podman rmi grafana/grafana

# Remove persistent data (WARNING: Deletes all Grafana data)
rm -rf ~/grafana-data
```

---

## What's Persisted?

When using persistent volume at `~/grafana-data`, the following data survives container removal and system reboots:

- ✅ All dashboards
- ✅ All data sources
- ✅ All users and permissions
- ✅ All alert rules
- ✅ All installed plugins
- ✅ All Grafana settings
- ✅ SQLite database (default Grafana storage)

**Not persisted**:
- ❌ Container logs (use `podman logs` or `journalctl`)
- ❌ External data source data (e.g., Prometheus metrics - managed externally)

---

## Resource Usage

Typical resource consumption:
- **Disk**: ~500MB for Grafana image + variable for data (start with 2GB)
- **Memory**: ~100-200MB
- **CPU**: <5% on modern systems
- **Startup Time**: ~30 seconds

---

## Next Steps

After successful deployment:

1. **Change default password**: Grafana prompts on first login (admin/admin)
2. **Configure data sources**: Add Prometheus, InfluxDB, or other monitoring backends
3. **Create dashboards**: Build custom visualizations or import from Grafana community
4. **Set up alerts**: Configure alert rules and notification channels
5. **Review security**: Consider reverse proxy with HTTPS for production

---

## Support

- **Documentation**: See full deployment documentation in this repository
- **Grafana Docs**: https://grafana.com/docs/grafana/latest/
- **Podman Docs**: https://docs.podman.io/
- **Troubleshooting**: Refer to troubleshooting section in full documentation

---

## Summary

You now have:
- ✅ Grafana running in Podman container
- ✅ Persistent storage for all configuration and data
- ✅ Auto-start on boot (if systemd configured)
- ✅ Verified data persistence across container lifecycle

**Total deployment time**: ~5-10 minutes

**Grafana URL**: http://localhost:3000

**Default credentials**: admin/admin (change on first login)
