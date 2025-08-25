# Guide: Operational Procedures for Docker Services

This guide provides step-by-step instructions for essential, high-level operational tasks within the
`Northern-Lights-Platform`. It is the authoritative source for procedures such as service migration, data inspection,
and emergency configuration changes.

These procedures are designed to work in harmony with the architecture defined in
`BLUEPRINT_docker_security_and_patterns.md`. A core assumption is that **Ansible is the single source of truth for all
configuration**.

---

## 1. Procedure: Service Migration (Host A to Host B)

This procedure details how to move a stateful service (e.g., Grafana) from an old host to a new host without data loss.
The process involves backing up the service's Docker Named Volume and restoring it on the new, pre-provisioned host.

### Prerequisites

- The **New Host (Host B)** has been fully provisioned by running the master Ansible playbook against it. This is
  critical, as it ensures Docker, `userns-remap`, and all required networks are correctly configured before any data is
  restored.
- You have SSH access to both hosts.

### Phase 1: Graceful Backup on the Old Host (Host A)

1. **Identify and Stop the Application Container(s):** Ansible assigns a predictable project name to all resources. Use
   this name to find and stop the relevant containers.

   ```bash
   # Find the container name using the project label set by Ansible
   # The project name is defined in your Ansible role (e.g., 'grafana')
   docker ps --filter "label=com.docker.compose.project=grafana"

   # Stop the container(s) found in the previous command
   docker stop grafana
   ```

2. **Identify the Volume's Host Path:** Use `docker volume inspect` to find the exact directory where Docker stores the
   volume's data. The volume name is typically `<project_name>_<volume_name_in_compose>`.

   ```bash
   docker volume inspect grafana_app_data
   ```

   The output is JSON. Locate the `"Mountpoint"` key. _Example Mountpoint:
   `/opt/docker/data/volumes/grafana_app_data/_data`_

3. **Create a Compressed Archive of the Volume:** Using `tar` is the recommended method as it creates a single, portable
   file and preserves all file permissions and ownership, which is essential for `userns-remap`.

   ```bash
   # -C changes directory to the Mountpoint, preventing the full path from being stored in the archive.
   sudo tar -czf /tmp/grafana_data_backup.tar.gz -C /opt/docker/data/volumes/grafana_app_data/_data .
   ```

4. **Transfer the Backup to the New Host:** Use a secure method like `scp` to move the archive to Host B.

   ```bash
   scp /tmp/grafana_data_backup.tar.gz wasd@<new_host_ip>:/tmp/
   ```

### Phase 2: Secure Restore on the New Host (Host B)

1. **Create the Empty Named Volume:** Before you can restore data, the named volume must exist on Host B with the
   correct ownership. The safest way to create it is to have Docker do it by running a temporary utility container.

   ```bash
   # This command creates the volume, runs for a second, and cleans up.
   docker run --rm -v grafana_app_data:/data alpine true
   ```

2. **Identify the New Volume's Host Path:** Repeat step 2 on Host B to get the new mountpoint.

   ```bash
   docker volume inspect grafana_app_data
   ```

3. **Restore the Backup Data:** Extract the archive into the new, empty volume directory.

   ```bash
   # Extract the archive, preserving all permissions.
   sudo tar -xzf /tmp/grafana_data_backup.tar.gz -C /opt/docker/data/volumes/grafana_app_data/_data
   ```

4. **Start the Application via Ansible:** Run the Ansible playbook for the service against the new host. Ansible will
   see the container is not running and will start it. Docker will attach the pre-existing, data-filled volume
   automatically.

   ```bash
   ansible-playbook your_playbook.yml --tags grafana --limit <new_host_name>
   ```

   The migration is now complete and the service is under full Ansible management.

---

## 2. Procedure: Accessing and Inspecting Volume Data

There are two safe methods for inspecting the contents of a named volume.

### Method 1: The "Utility Container" (Recommended & Safest)

This is the standard, "Docker-native" method. It respects all security boundaries and works seamlessly with
`userns-remap`.

1. **Run a Temporary Container:** Launch a lightweight container and mount the volume you wish to inspect into it.

   ```bash
   # Replace 'grafana_app_data' with the name of the volume to inspect.
   docker run --rm -it -v grafana_app_data:/inspected-data alpine sh
   ```

2. **Browse the Volume Data:** You now have a shell inside the temporary container. The volume's contents are available
   at `/inspected-data`.

   ```sh
   # You are now inside the alpine container
   cd /inspected-data
   ls -la
   # Use tools like 'cat', 'vi', or 'less' to inspect files.
   exit
   ```

   The `--rm` flag automatically deletes the utility container when you exit.

### Method 2: Direct Host Access (Read-Only Inspection)

This method is for quick, read-only checks by an administrator.

1. **Find the Volume's Host Path:** Use `docker volume inspect <volume_name>`.
2. **Use `sudo` to List Files:** You must use `sudo` to access Docker's data directory. The `-n` flag for `ls` is useful
   to see the numeric UIDs.

   ```bash
   sudo ls -lan /opt/docker/data/volumes/grafana_app_data/_data
   ```

   > **Note:** You will observe that the files are owned by a high-numbered UID (e.g., `101337`). This is the
   > `userns-remap` feature working correctly. Direct editing is strongly discouraged as it can corrupt file ownership.

---

## 3. Procedure: Emergency Configuration & Debugging

This procedure is for temporary, emergency situations where a live change to a container's configuration is required for
debugging.

> **CRITICAL:** This is a **temporary, imperative action** that violates the declarative nature of the architecture. Any
> changes made using this method **will be destroyed and reverted** the next time the service's Ansible playbook is run.
> This is a self-healing feature of the system. This procedure should **only be used for debugging**.

### Method: `docker cp` (Copy, Edit, Replace)

This is the most reliable method for editing files inside a minimal, non-root container. It avoids the need for editors
or special permissions inside the container itself.

1. **Copy the File from the Container to the Host:** Use `docker cp` to pull the configuration file out of the container
   and onto your host machine.

   ```bash
   # Syntax: docker cp <container_name>:<path_in_container> <local_path>
   docker cp grafana:/etc/grafana/grafana.ini /tmp/grafana.ini.edit
   ```

2. **Edit the File on the Host:** Use your preferred text editor on the host to make the necessary emergency changes.

   ```bash
   nano /tmp/grafana.ini.edit
   ```

3. **Copy the Modified File Back into the Container:** Use `docker cp` again to replace the original file inside the
   container with your modified version.

   ```bash
   # Syntax: docker cp <local_path> <container_name>:<path_in_container>
   docker cp /tmp/grafana.ini.edit grafana:/etc/grafana/grafana.ini
   ```

4. **Restart the Container Process:** For the change to be loaded, the container must be restarted.

   ```bash
   docker restart grafana
   ```

Once debugging is complete, ensure you update the configuration in your Ansible variables and redeploy the service via
the playbook to return to a clean, declarative state.
