# Docker Provisioning Role

## Overview

This Ansible role implements a production-ready, security-hardened Docker CE installation following the
Northern-Lights-Platform architecture principles. It enforces the three-pillar security model defined in the platform's
security blueprint.

## Architecture

### Three-Pillar Security Model

1. **Host-Container Isolation** (Pillar 1)

   - Implements `userns-remap` with a dedicated `dockremap` user
   - Maps container UIDs to unprivileged host UIDs (500000-165535)
   - Prevents container breakout attacks from escalating to host compromise

2. **Container-Container Isolation** (Pillar 2)

   - Enforces Docker Named Volumes for all persistent data
   - Prohibits bind mounts for stateful data
   - Ensures complete isolation between container filesystems

3. **Intra-Container Security** (Pillar 3)
   - Standard non-root UID (1337) for all container processes
   - Minimizes attack surface within compromised containers
   - Consistent security posture across all services

### Directory Structure

```tree
/opt/docker/
├── data/       # Mode: 0701 - Docker's private directory
├── services/   # Mode: 0750 - Ansible role data and compose files
├── secrets/    # Mode: 0700 - Host-side secret files
├── logs/       # Mode: 0755 - Parent for log file bind-mounts
├── backups/    # Mode: 0700 - Configuration backups
└── scripts/    # Mode: 0750 - Custom Docker scripts
```

All directories are owned by `root:root` following the security model.

## Requirements

- **Ansible Version**: >= 2.12
- **Operating Systems**: Ubuntu, Debian, CentOS, RHEL, AlmaLinux, Rocky
- **Architecture**: x86_64, amd64, aarch64, arm64
- **Disk Space**: Minimum 10GB free in `/opt`
- **Privileges**: Root or sudo access required

## Role Variables

### Required Variables (define in group_vars)

```yaml
# Core security configuration
dockremap_user: "dockremap"
subordinate_uid_start: 500000
subordinate_range_size: 65536
container_internal_uid: 1337

# Docker directories
docker:
  home: "/opt/docker"
  container_logs: "/opt/docker/logs"
  container_secrets: "/opt/docker/secrets"

# Network configuration
docker_net:
  bip: "10.142.0.1/24" # Bridge IP based on hierarchical networking
  default_address_pools:
    - { base: "10.142.1.0/24", size: 28 }
```

### Optional Variables

```yaml
# Primary user to add to docker group
primary_user_name: "myuser"

# Docker daemon settings
docker_provisioning_log_max_size: "10m"
docker_provisioning_log_max_file: "3"
docker_provisioning_storage_driver: "overlay2"
docker_provisioning_live_restore: true
```

## Dependencies

None. This role is self-contained.

## Example Playbook

```yaml
---
- name: Provision Docker with security hardening
  hosts: docker_hosts
  become: true
  roles:
    - role: docker_provisioning
      tags:
        - docker
```

## Usage Examples

### Basic Installation

```bash
# Run the complete Docker provisioning
ansible-playbook -i inventories/hosts.yml playbooks/docker.yml

# Run only specific phases
ansible-playbook -i inventories/hosts.yml playbooks/docker.yml --tags docker-install
ansible-playbook -i inventories/hosts.yml playbooks/docker.yml --tags docker-config
```

### Using Named Volumes (Required Pattern)

```yaml
# docker-compose.yml
version: "3.8"

services:
  app:
    image: myapp:latest
    user: "1337" # Standard container UID
    volumes:
      - app_data:/data # Named volume for persistent data
      - app_config:/config # Named volume for configuration

volumes:
  app_data: # Docker manages this volume
  app_config: # Docker manages this volume
```

### Sharing Logs (Managed Exception)

```yaml
services:
  log-reader:
    image: analyzer:latest
    user: "1337"
    group_add:
      - "3000" # logreaders group
    volumes:
      - /opt/docker/logs:/logs:ro # Read-only bind mount for logs only
```

## Security Considerations

### UID Mapping

- Container UID 0 (root) → Host UID 500000
- Container UID 1337 → Host UID 101337
- All container UIDs are mapped to unprivileged host UIDs

### Named Volumes Only

This role enforces Named Volumes for all persistent data. Bind mounts are prohibited except for:

- Read-only log collection
- Unix socket access (e.g., Docker socket)
- Development hot-reload (never in production)

### File Permissions

- All Docker directories owned by `root:root`
- Service logs use group `logreaders` (GID: 3000) for shared access
- Secrets directory has mode 0700 (root access only)

## Validation

After installation, the role validates:

- Docker service is running
- User namespace remapping is active
- Required directories exist with correct permissions
- Docker Compose plugin is available
- Network configuration is applied

## Troubleshooting

### Common Issues

1. **Permission Denied Errors**

   - Ensure you're using Named Volumes, not bind mounts
   - Verify container is running with UID 1337

2. **Docker Group Membership**

   - Users need to log out and back in after being added to docker group
   - Verify with: `groups username`

3. **Subordinate UID Conflicts**
   - Check `/etc/subuid` and `/etc/subgid` for conflicts
   - Ensure range 500000-165535 is available

### Logs and Debugging

```bash
# Check Docker daemon logs
journalctl -u docker.service -f

# Verify user namespace remapping
docker info | grep -i userns

# Check subordinate ranges
cat /etc/subuid /etc/subgid | grep dockremap

# Inspect volume permissions
docker run --rm -v myvolume:/data alpine ls -la /data
```

## Migration from Bind Mounts

If migrating from bind mounts to Named Volumes:

```bash
# Create named volume
docker volume create app_data

# Copy data from bind mount to volume
docker run --rm \
  -v /old/host/path:/source:ro \
  -v app_data:/destination \
  alpine cp -a /source/. /destination/

# Update docker-compose.yml to use named volume
# Restart services
```

## License

BSD

## Author Information

Northern-Lights-Platform Team

## References

- [Docker Security Blueprint](../../_docs/README_docker_security.md)
- [Named Volumes vs Bind Mounts Guide](../../_docs/DEFINITIVE_GUIDE_volumes_vs_bind_mounts.md)
- [ADR-001: Foundational Architecture](../../_docs/adrs/ADR-001.md)
