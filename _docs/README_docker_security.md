# Blueprint: Docker Security & Implementation Patterns

This document outlines the definitive, multi-layered security architecture and standard implementation patterns for all
container deployments in the `Northern-Lights-Platform`. It is designed to provide defense-in-depth, ensure operational
consistency, and serve as the single source of truth for the project's security posture. Adherence to this model is
mandatory.

## Architectural Philosophy

The entire model is built upon three foundational principles which inform every decision:

1. **Security First:** We employ a defense-in-depth strategy, assuming a zero-trust environment. The primary goal is to
   protect the host from containers and to protect containers from each other.
2. **Declarative State:** The desired state of our system is defined as code (primarily Ansible). We describe _what_ we
   want, and let the tools enforce that state idempotently.
3. **Simplicity & Scalability:** The system must be easy to manage and scale. We achieve this by using a single,
   consistent user model and relying on battle-tested Docker features.

---

## Project-Wide Constants

To ensure consistency, the following values are established as project-wide standards, managed within Ansible variables.

| Variable Name            | Example Value | Purpose                                                                                     |
| :----------------------- | :------------ | :------------------------------------------------------------------------------------------ |
| `dockremap_user`         | `dockremap`   | The dedicated, non-login system user for `userns-remap`.                                    |
| `subordinate_uid_start`  | `100000`      | The starting UID/GID in the subordinate range assigned to the `dockremap` user.             |
| `subordinate_range_size` | `65536`       | The total number of UIDs/GIDs in the subordinate pool.                                      |
| `container_internal_uid` | `1337`        | The standard non-root UID/GID to be used for all application processes _inside_ containers. |

---

## Host System Security & User Policy

### Sudo Configuration Policy

For a single-administrator system where authentication is secured by a hardware key (e.g., YubiKey), the risk profile
for `sudo` operations is altered. The primary threat shifts from weak passwords to post-authentication attacks like
session hijacking or malicious script execution.

- **The Enterprise Best Practice (Recommended):** Retain the default `sudo` password requirement. The prompt serves as a
  critical, final authorization checkpoint before executing a privileged command, providing a layer of defense against
  accidental or malicious actions.
- **The Homelab Convenience Option (Accepted Risk):** For a streamlined workflow, passwordless `sudo` may be implemented
  for the primary administrative user. This is a conscious trade-off that prioritizes convenience over the added
  security of the authorization checkpoint. This risk is deemed acceptable _only_ when strong, hardware-based
  authentication is strictly enforced for all logins.
- **Implementation (if passwordless is chosen):**

  ```bash
  # /etc/sudoers.d/01-admin-user
  wasd ALL=(ALL) NOPASSWD: ALL
  ```

### Filesystem Ownership Policy

To maintain strict security boundaries and ensure operational stability of the Docker daemon, the following ownership
structure is mandatory.

- **/opt/docker:** The root directory for the entire environment **MUST** be owned by `root:root`.
- **Rationale:** The Docker daemon runs as the `root` user and requires authoritative control over its environment.
  Administrative actions on this directory are performed via `sudo` through Ansible, adhering to the Principle of Least
  Privilege. Granting ownership to a non-root user would introduce significant security risks and potential for runtime
  permission errors.

---

## Pillar 1: Host-Container Isolation (The Host's Shield)

This is the most critical layer of defense, preventing a container compromise from escalating to a host compromise.

- **Mechanism:** Docker's User Namespaces (`userns-remap`).
- **How It Works:** `userns-remap` maps users inside the container to the unprivileged subordinate range on the host.
  For example, `root` (UID 0) inside a container becomes UID `100000` on the host. If an attacker escapes the container,
  they are effectively neutralized, having no permissions on the host system.
- **Reference Implementation:**

  - A dedicated `dockremap` user and subordinate UID/GID ranges are configured on the host via Ansible.

    ```yaml
    # Ansible Task: tasks/host-setup.yml
    - name: Create the 'dockremap' system user
      ansible.builtin.user:
        name: "{{ dockremap_user }}"
        system: yes
        shell: /bin/false
        create_home: no

    - name: Assign subordinate UID/GID range to the dockremap user
      ansible.builtin.lineinfile:
        path: "/etc/sub{{ item }}"
        line: "{{ dockremap_user }}:{{ subordinate_uid_start }}:{{ subordinate_range_size }}"
        create: yes
        owner: root
        group: root
        mode: "0644"
      loop:
        - uid
        - gid
    ```

  - The Docker daemon is configured to enable the feature. The full `daemon.json` content is managed declaratively by
    Ansible.

    ```json
    // /etc/docker/daemon.json
    {
      // Pillar 1: Enable userns-remap to isolate host from containers.
      "userns-remap": "dockremap",

      // Define the root directory for all Docker-managed files (volumes, images, etc.).
      "data-root": "/opt/docker/data",

      // General best practices
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "10m",
        "max-file": "3"
      }
    }
    ```

---

## Pillar 2: Container-Container Isolation (The "Golden Rule")

This layer prevents a compromised container from accessing the data of its neighbors.

- **Mechanism:** Docker **Named Volumes**.
- **The Golden Rule:** All persistent, stateful application data **MUST** be stored in a Docker Named Volume.
- **Why This Is The Correct Method:** With `userns-remap` active, the Docker daemon automatically handles the complex
  ownership mapping for named volumes, ensuring the container's remapped user can access its data. Furthermore, Docker
  creates a `root`-owned directory structure for its volumes, which acts as a security barrier preventing a process in
  one container from traversing the filesystem to access another container's data.
- **Forbidden Alternative (Bind Mounts):** The use of bind-mounts (`- /some/host/path:/data`) for stateful data is
  **strictly forbidden**. This practice is incompatible with our security model, as the container's remapped user (e.g.,
  `101337`) would be denied permission to write to a host path owned by a different user, leading to runtime failures.
- **Reference Implementation (`docker-compose.yml`):**

  ```yaml
  services:
    app1:
      image: some/image
      user: "{{ container_internal_uid }}"
      volumes:
        # Correct: Maps the named volume 'app1_data' to a path inside the container.
        - app1_data:/path/in/container

  # Crucial: Declares the named volume for Docker to manage.
  volumes:
    app1_data:
  ```

---

## Pillar 3: Intra-Container Security (Least Privilege)

This final layer minimizes an attacker's capabilities _inside_ a compromised container.

- **Mechanism:** A single, consistent, non-root user for all application processes.
- **Concept:** We use the standard `container_internal_uid` (e.g., `1337`) for all containers, set via
  `user: "{{ container_internal_uid }}"`. This user does not need to exist on the host.
- **Rationale:** Running as non-root _inside_ the container prevents an attacker who compromises the process from using
  package managers (`apt install`), modifying application binaries, or changing configuration files within the
  container's own filesystem.
- **Reference Implementation (`docker-compose.yml`):**

  ```yaml
  services:
    app1:
      image: some/image
      # Run the process as the standard non-root user inside the container.
      user: "{{ container_internal_uid }}"
      volumes:
        - app1_data:/path/in/container
  volumes:
    app1_data:
  ```

---

## Standard Implementation Patterns

These are repeatable, secure patterns for building and deploying applications on the platform.

### 1. Host Filesystem Structure

This structure, managed by Ansible, provides clear separation of concerns. All paths are owned by `root:root`.

```tree
/opt/docker/
├── data/       # (Mode: 0701) Docker's private directory. DO NOT TOUCH.
├── services/   # (Mode: 0750) Stores Ansible role data (e.g., compose templates).
├── secrets/    # (Mode: 0700) Stores host-side secret files.
└── logs/       # (Mode: 0755) Parent for log file bind-mounts (a managed exception).
```

### 2. Handling Secrets

This pattern securely provides credentials, even to applications that can't read from Docker secrets natively.

1. **Ansible Creates the Secret File on the Host:**

   ```yaml
   # tasks/secrets.yml
   - name: Generate and save secret if it does not exist
     ansible.builtin.copy:
       content: "{{ lookup('password', '/dev/null length=32') }}"
       dest: /opt/docker/secrets/db_password
       owner: root
       group: root
       mode: "0600"
   ```

2. **Docker Compose Consumes the File as a Docker Secret:**

   ```yaml
   services:
     app:
       image: ...
       user: "{{ container_internal_uid }}"
       secrets:
         - db_password
   secrets:
     db_password:
       file: /opt/docker/secrets/db_password
   ```

3. **(Optional) Entrypoint Bridge for Legacy Apps:** For apps that can only read secrets from environment variables.

   - **`entrypoint.sh`:**

     ```sh
     #!/bin/sh
     set -e
     export APP_DB_PASSWORD=$(cat /run/secrets/db_password)
     exec "$@"
     ```

   - **`Dockerfile`:**

     ```dockerfile
     FROM original/image:latest
     COPY entrypoint.sh /usr/local/bin/
     RUN chmod +x /usr/local/bin/entrypoint.sh
     ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
     CMD ["/run.sh"] # Must be the original CMD from the base image
     ```

### 3. Sharing Read-Only Access (e.g., Log Analysis)

This pattern allows a "reader" container to securely access logs written by other containers via a bind-mount, which is
a managed exception to the "Golden Rule" for non-stateful, read-only data.

1. **Ansible Creates a Shared Group:** A dedicated group (e.g., `logreaders` with GID `3000`) is created on the host.
2. **Ansible Sets Log Directory Permissions:**

   ```yaml
   # tasks/logs.yml
   - name: Create service log directory with shared group ownership
     ansible.builtin.file:
       path: "/opt/docker/logs/{{ item.name }}"
       state: directory
       # The owner is the container's remapped UID. The group is the shared group.
       owner: "{{ subordinate_uid_start + item.uid }}" # e.g., 100000 + 1337
       group: logreaders
       mode: "0750" # Owner: rwx, Group: r-x, Other: ---
     loop:
       - { name: "grafana", uid: "{{ container_internal_uid }}" }
       - { name: "postgres", uid: "{{ container_internal_uid }}" }
   ```

   > **Note:** This pattern relies on the "writer" application creating its log files with group-readable permissions
   > (e.g., `640` or `660`).

3. **"Reader" Container Joins the Group:**

   ```yaml
   services:
     log-analyzer:
       image: some/analyzer
       user: "{{ container_internal_uid }}"
       # The container process is added to the 'logreaders' GID on the host.
       group_add:
         - "3000"
       volumes:
         # Mount the entire log directory as read-only.
         - /opt/docker/logs:/logs:ro
   ```

---

## Summary

This three-pillar model provides a complete, defense-in-depth security posture.

| Pillar                               | Mechanism                  | Security Outcome                                                    |
| :----------------------------------- | :------------------------- | :------------------------------------------------------------------ |
| **1. Host-Container Isolation**      | `userns-remap`             | Protects the **Host** from a compromised **Container**.             |
| **2. Container-Container Isolation** | **Named Volumes**          | Protects **Containers** from **each other** on the host filesystem. |
| **3. Intra-Container Security**      | **Standard Non-Root User** | Protects a **Container** from its **own compromised process**.      |
