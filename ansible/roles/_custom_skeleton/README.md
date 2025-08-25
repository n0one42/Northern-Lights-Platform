# Custom Ansible Role Skeleton for Northern-Lights-Platform

This is an enterprise-grade Ansible role skeleton template designed specifically for Docker-based applications with advanced security patterns, database support, and service mesh integration.

## 🎯 Purpose

This skeleton serves as a Jinja2 template for `ansible-galaxy init` to generate new roles with:
- Docker Compose v2 integration
- Socket proxy pattern for Docker API security
- PostgreSQL and Redis support
- CrowdSec security integration
- Traefik reverse proxy ready
- User namespace remapping support
- Comprehensive secret management

## 🚀 How to Use This Skeleton

### Creating a New Role

From the `ansible/` directory, run:

```bash
# Create a new role using this skeleton
ansible-galaxy init --role-skeleton=roles/_custom_skeleton roles/dc_myapp

# The role name should follow the pattern: dc_<service_name>
# Examples:
#   dc_traefik
#   dc_grafana
#   dc_authentik
```

### Understanding the Template Syntax

This skeleton uses **double Jinja2 templating**:
- `{{ role_name }}` - Processed during `ansible-galaxy init`
- `{{ '{{ variable }}' }}` - Becomes `{{ variable }}` in the generated role

Example:
```yaml
# In skeleton (main.yml.j2):
{{ role_name.split('/')[-1] }}_image: "{{ '{{ docker_registry }}' }}/myapp:latest"

# Becomes in generated role (main.yml):
dc_myapp_image: "{{ docker_registry }}/myapp:latest"
```

## 📁 Generated Structure

```
roles/dc_myapp/
├── README.md           # Role documentation
├── defaults/
│   └── main.yml       # Default variables (fully documented)
├── files/             # Static files
├── handlers/
│   └── main.yml       # Container restart handlers
├── meta/
│   └── main.yml       # Role metadata and dependencies
├── tasks/
│   └── main.yml       # Main task orchestration
├── templates/         # Jinja2 templates
├── tests/
│   ├── inventory      # Test inventory
│   └── test.yml       # Test playbook
└── vars/
    └── main.yml       # Internal variables
```

## 🔧 Features Included

### 1. **Network Isolation**
- Separate internal networks for app and socket proxy
- Integration with external Traefik network
- Configurable subnet allocation based on VM ID

### 2. **Security Patterns**
- Socket proxy to avoid direct Docker socket exposure
- User namespace remapping (`userns_mode: remap`)
- Non-root container execution
- Secrets management via Docker secrets
- No-new-privileges security option

### 3. **Database Support**
- PostgreSQL with Bitnami images
- Automatic password generation if not provided
- Separate database network isolation
- Configurable database parameters

### 4. **Redis Cache**
- Redis with Bitnami images
- Password-protected setup
- Network isolation

### 5. **Service Dependencies**
- CrowdSec for intrusion detection
- Filesystem manager for directory structure
- Conditional service inclusion

## 🔨 Customizing the Generated Role

After generating a role, customize it by:

### 1. **Update Image Configuration**
Edit `defaults/main.yml`:
```yaml
dc_myapp_image: "myapp:1.0.0"
dc_myapp_web_port: 8080
```

### 2. **Configure Networks**
Set network ranges in your inventory:
```yaml
dc_myapp_net:
  subnet: "10.105.0.0/24"
  myapp_ip: "10.105.0.10"
```

### 3. **Enable/Disable Features**
```yaml
dc_myapp_use_socketproxy: false  # Disable if not needed
dc_myapp_use_database: true      # Enable PostgreSQL
dc_myapp_crowdsec_enabled: true  # Enable CrowdSec
```

### 4. **Add Service-Specific Configuration**
Modify the Docker Compose definition in `tasks/main.yml` to add:
- Environment variables
- Volume mounts
- Additional labels
- Health checks

## 📋 Variable Naming Convention

All variables follow the pattern: `<role_name>_<variable>`

Examples:
- `dc_traefik_image`
- `dc_grafana_db_user_pw`
- `dc_authentik_net`

## 🔐 Secret Management

Secrets are handled in three ways:

1. **Auto-generated**: If not provided, passwords are generated
2. **File-based**: Stored in `/opt/docker/secrets/`
3. **Docker secrets**: Mounted as files in containers

Example:
```yaml
filesystem_manager_secret_files_to_create:
  - file_name: "dc_myapp_db_user_pw"
    content: "{{ dc_myapp_db_user_pw | default('') }}"
    type: postgres_password  # Auto-generates if empty
```

## ⚙️ Socket Proxy Configuration

The socket proxy provides controlled access to Docker API:

```yaml
# Safe permissions (default)
dc_myapp_socketproxy_events: 1
dc_myapp_socketproxy_ping: 1
dc_myapp_socketproxy_version: 1
dc_myapp_socketproxy_containers: 1

# Dangerous permissions (disabled by default)
dc_myapp_socketproxy_auth: 0      # Authentication API
dc_myapp_socketproxy_post: 0      # POST operations
dc_myapp_socketproxy_secrets: 0   # Access to secrets
```

## 🧪 Testing

Each generated role includes a test structure:

```bash
# Run tests
cd roles/dc_myapp
ansible-playbook -i tests/inventory tests/test.yml
```

## 📝 ADR Compliance

This skeleton follows ADR-001 requirements:
- File path headers in all generated files
- Structural block comments for organization
- Explicit variable definitions
- FQCN for all modules

## 🤝 Best Practices

1. **Always use FQCN**: `community.docker.docker_compose_v2`
2. **Name all tasks**: Following the `{stem} | description` pattern
3. **Document variables**: In `defaults/main.yml`
4. **Use handlers**: For container restarts
5. **Follow security patterns**: Socket proxy, non-root, userns

## 💡 Examples

### Creating a Grafana Role

```bash
ansible-galaxy init --role-skeleton=roles/_custom_skeleton roles/dc_grafana
```

Then customize:
```yaml
# defaults/main.yml
dc_grafana_image: "grafana/grafana:10.0.0"
dc_grafana_web_port: 3000
dc_grafana_use_database: true
dc_grafana_use_redis: true
```

### Creating a Simple Web App

```bash
ansible-galaxy init --role-skeleton=roles/_custom_skeleton roles/dc_webapp
```

Disable unneeded features:
```yaml
# defaults/main.yml
dc_webapp_use_socketproxy: false
dc_webapp_use_database: false
dc_webapp_use_redis: false
```

## 📚 Further Reading

- [Docker Compose v2 Module](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_v2_module.html)
- [Socket Proxy Documentation](https://github.com/Tecnativa/docker-socket-proxy)
- [User Namespace Remapping](https://docs.docker.com/engine/security/userns-remap/)

## 🛠️ Maintenance

To update this skeleton:
1. Edit files in `roles/_custom_skeleton/`
2. Test by generating a sample role
3. Verify all Jinja2 templates render correctly
4. Update this README if adding new features

## ⚠️ Important Notes

- All `.j2` files are Jinja2 templates processed during role creation
- The `{{ '{{ var }}' }}` syntax is intentional for double templating
- File path headers are added per ADR-001 requirements
- This skeleton is optimized for Docker-based services

## License

BSD

## Author Information

Northern-Lights-Platform Team