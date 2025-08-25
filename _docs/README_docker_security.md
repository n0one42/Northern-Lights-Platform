# Docker Container Security: UID Management Strategy

## Question Overview

I need advice on Docker container security regarding UID management. I'm unsure if it's worth the effort to use unique
UIDs for each container or if a simpler approach would be sufficient.

## Current Setup

I have an environment with multiple Docker containers where:

- Each container uses a non-root, non-host mapped user
- Example: UID 1234 doesn't exist on the host system (no permissions)
- Permissions are manually set through Ansible so containers only access their specific host files
- Example structure:
  - `opt/docker/appdata/grafana/app/` owned by UID 1234
  - `opt/docker/appdata/grafana/db/` owned by UID 1333
  - Docker daemon runs with userns-remap UID 2222

## Security Questions

### Primary Concern

Is it worth using unique UIDs (1234, 1333, etc.) for each container? Since Grafana only mounts
`opt/docker/appdata/grafana/app/` and not `opt/docker/appdata/grafana/db/`, wouldn't it be impossible to access the
unmounted path?

### Compromise Scenario

If the Grafana container (UID 1234) gets compromised:

- Docker daemon runs as non-root user (UID 2222)
- Grafana has no other host permissions
- **Question:** Could Grafana access `opt/docker/appdata/grafana/db/` if compromised, even when unmounted?

### Simplification Question

Would it be easier to let containers run with default users mapped through userns-remap to UID 2222? Although this might
give compromised containers access to everything UID 2222 can access.

## Expert Analysis

### 1. Container Filesystem Isolation

**Short Answer:** No, containers cannot access unmounted paths by default.

**Detailed Explanation:**

Docker isolates filesystems by default. A container only sees:

1. Its own filesystem layers (image + union filesystem overhead)
2. Explicitly attached bind mounts or volumes

If `opt/docker/appdata/grafana/db/` is not mounted into the container, the container has no path referencing it. An
attacker inside the container cannot magically access the host's `/opt` directory unless it's mounted or there's a
Docker misconfiguration.

Even if the container UID matches a host user that can read the folder, the container's namespace doesn't map to the
real host filesystem in a way that allows reading arbitrary directories.

**Bottom Line:** Having the same UID on the host doesn't grant access to arbitrary unmounted host paths.

### 2. Unique Host UIDs: Security Benefits

**Short Answer:** Can improve security, but may be overkill for typical setups.

**Detailed Analysis:**

#### User Namespaces (userns-remap)

You've enabled userns-remap, which maps container users (including root) to non-privileged host users (e.g., UID 2222).
This drastically limits damage from container compromise since the host sees only an unprivileged UID.

#### Multiple Unique UIDs

**Scenarios where helpful:**

- Multiple containers with different unprivileged host UIDs reduce risk of cross-container data access if there's
  partial filesystem exposure
- Provides defense-in-depth if you accidentally mount directories with shared ownership

**Limitations:**

- If containers never mount shared folders and only see explicitly bound directories, they're already isolated
- File ownership on host only matters when multiple containers share host directories with different permission
  requirements

#### Security vs. Operational Complexity

**Benefits:**

- Ensures compromised containers can't access other containers' data, even with accidental mounts
- Provides additional isolation layer

**Drawbacks:**

- Maintenance overhead: Ansible user ID matrix, Dockerfile/Compose mappings
- Consistency requirements across staging/production
- Increased operational complexity

**Bottom Line:** If containers don't share volumes and mount only their own data, unique host UIDs provide minimal
practical benefit for most internal deployments.

### 3. Same UID for App and DB Containers

**Mount-based Access Control:**

- **Not mounted = not accessible:** If `grafana/app/` is the only mounted folder, the container can't read
  `grafana/db/`, regardless of matching UIDs, because the folder isn't in the container's mount namespace
- **Shared volume scenario:** If both directories are mounted in containers with the same UID, a compromised container
  would have read/write access

**Bottom Line:** If the `db/` path is never mounted in the compromised container, it remains invisible.

### 4. Recommended Approach

**Consider These Factors:**

#### Minimal Additional Security

If each container's data is in separate directories without cross-mounting, the default userns mapping (all containers
as user 2222 on host) is generally sufficient. Container compromise escalates to host UID 2222, but that user only owns
the single directory mounted for that container.

#### Multi-Tenant / High-Security Environments

For containers from different teams or lower trust environments, unique host UIDs provide "defense in depth" by reducing
chances that mount mistakes or exploits could allow cross-container volume access.

#### Operational Complexity

Every additional unique UID requires maintenance in Docker, orchestrators (Compose, Swarm, Kubernetes), and
configuration management (Ansible). This accumulates friction when managing large container fleets.

**Recommendation:** For self-managed environments where you control all containers, using default userns remap to an
unprivileged host user provides good security-vs-simplicity balance.

## Additional Hardening Steps

Regardless of UID approach, consider these best practices:

### 1. Minimal Privilege Security Profiles

- **AppArmor/SELinux:** Enable policies restricting container system calls and paths
- **Seccomp profiles:** Block unnecessary syscalls when feasible

### 2. Non-root Container Users

- Even with userns mapping root → UID 2222 on host, run containers with non-root users internally
- Many official images support `USER` directive in Dockerfile/Compose

### 3. Explicit Restrictions

- **Read-only mounts:** For data not requiring writes
- **no-new-privileges:** Use in Docker/Compose settings when possible

### 4. Network Segmentation

- Segment Docker networks so compromised containers can't easily pivot to other internal services

### 5. Capability Limitations

- Drop unnecessary capabilities (SYS_ADMIN, NET_ADMIN, etc.)

## Conclusion

### Key Takeaways

- **Single userns remapped UID** (e.g., 2222) for all containers typically provides sufficient isolation when each
  container only mounts dedicated volume paths
- **Unique UIDs per container** can help in advanced, highly secure, or multi-tenant environments, but adds complexity
  that often isn't justified if container volumes are strictly separate
- **Container isolation** prevents access to unmounted host directories—if `db/` is never mounted in the "app"
  container, it remains inaccessible

### Final Recommendation

If operational simplicity is important, don't over-engineer the per-container UID approach. Default userns remap is
already a significant security improvement over running containers as host root. For serious isolation or compliance
requirements, you can implement more complex approaches, but for most internal environments, the overhead may not be
justified.

## Follow-up Discussion

### Clarification Questions

**Q:** For single-admin environments without multi-tenancy, is container compromise still concerning regarding unmounted
path access?

**A:** No, not under normal Docker isolation rules.

#### Mount Namespace Isolation

By default, containers only see:

- Container's own filesystem/image layers
- Explicitly attached volumes or host bind mounts
- Unmounted host directories are invisible to container processes

#### User ID Limitations

Having matching or different UIDs on the host doesn't grant containers magical ability to see unmounted directories.
Even if a compromised process "knows" about `/opt/docker/appdata/grafana/db/` on the host, that path doesn't exist
inside the container unless explicitly mounted.

#### Exceptions to Consider

- **Privileged containers:** `--privileged` flag can break namespace isolation (avoid unless absolutely necessary)
- **Docker socket mounting:** Mounting `/var/run/docker.sock` gives container control over Docker daemon (huge security
  risk)
- **Kernel/Docker vulnerabilities:** System-level exploits could theoretically break container isolation

### Simplified Approach Recommendation

#### Single Non-Root User Strategy

**Setup:**

- Every container runs as `USER 1234` (inside container)
- Userns-remap maps container UID 1234 → host UID 2222
- All containers appear as same unprivileged host user while maintaining filesystem isolation

**Security Impact:**

- For single-admin environments, difference between one non-root user vs. separate users is usually negligible
- Container compromise scenario remains the same: hacked containers cannot access files outside mounted volumes
- Host UID sharing doesn't compromise isolation if mount points are separate

**Operational Benefits:**

- Much easier management with single user ID in Dockerfiles/Compose/Ansible
- Reduced overhead tracking "who owns what" at host level
- Simplified permission management

### Why Use Non-Root Inside Containers with userns?

**Layered Security Benefits:**

1. **Container Root → Host Non-root:** Even if container root is compromised, it corresponds to unprivileged UID on host
   (e.g., 2222)
2. **Non-root Container User:** Many official images run as root by default; overriding to non-root user (USER 1234) is
   best practice
3. **Combined Protection:** Layers two forms of user separation—inside container (non-root) and on host (remapped to
   unprivileged UID)

This combination reduces risk of container escapes and host system tampering.

### Final Implementation Recommendation

**Given your single-admin scenario:**

1. **Not Multi-Tenant:** No urgent need to isolate containers via distinct host UIDs
2. **No Overlapping Volumes:** Each container exclusively mounts its own directory
3. **Userns-Remap Configured:** Root-level container compromise maps to non-root host user

**Proposed Simple Setup:**

- Use single non-root user for all containers (USER 1234 in Dockerfile/Compose)
- Let userns-remap handle mapping (container user 1234 → host user 2222)
- Manage single set of file permissions for each container's data directory

**Security:** Compromised containers still cannot read/write data outside explicitly mounted volumes due to Docker's
isolation.

**Simplicity:** Track only one user ID inside containers (1234) and let Docker remap to single unprivileged host user
(2222).

### Optional Additional Hardening

- **AppArmor/SELinux:** Tailored profiles for extra path and syscall enforcement
- **Seccomp:** Keep default profile blocking dangerous system calls
- **no-new-privileges:** Set `security_opt: ["no-new-privileges"]` when possible
- **Capability dropping:** Remove unnecessary capabilities (CAP_SYS_ADMIN)
- **Docker socket protection:** Never mount `/var/run/docker.sock` unless absolutely required

## Summary

For single-tenant users where each container has dedicated mounts, there's little practical benefit to unique UIDs per
container. Using a single non-root user (1234) inside containers with userns-remap to unprivileged host user (2222)
provides adequate security with reduced operational overhead.

Compromised containers cannot access other containers' host files unless explicitly mounted, making this setup standard
and sufficient for most personal or single-admin Docker environments.
