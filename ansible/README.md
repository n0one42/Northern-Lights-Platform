# Ansible Engine for the Northern-Lights-Platform

This directory contains the complete Ansible project for provisioning and configuring the Northern-Lights-Platform infrastructure.

## 🚀 Quick Start

### Prerequisites

**macOS Users:**
```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Ansible and tools
brew install ansible ansible-lint

# Verify installation
ansible --version
ansible-lint --version
```

**Linux Users:**
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install ansible ansible-lint

# RHEL/CentOS/Fedora
sudo dnf install ansible ansible-lint

# Arch Linux
sudo pacman -S ansible ansible-lint
```

**Windows Users:**
Use WSL2 (Windows Subsystem for Linux) and follow the Linux instructions above.

### Setup

1. **Clone the repository:**
   ```bash
   git clone https://github.com/your-org/Northern-Lights-Platform.git
   cd Northern-Lights-Platform/ansible
   ```

2. **Open in VS Code:**
   ```bash
   # From the repository root
   code Northern-Lights-Platform.code-workspace
   ```

3. **Verify your setup:**
   ```bash
   # Check Ansible version (should be 8.0.0 or higher)
   ansible --version
   
   # Check ansible-lint (should be 6.22.0 or higher)
   ansible-lint --version
   
   # Test the configuration
   ansible-inventory --graph
   
   # Run linting
   ansible-lint
   ```

That's it! You're ready to use Ansible.

## 📁 Project Structure

```
ansible/
├── inventories/          # Inventory files and group/host variables
│   └── hosts.yml        # Main inventory file
├── playbooks/           # Ansible playbooks
├── roles/              # Ansible roles
├── collections/        # Ansible collections (if any)
├── group_vars/         # Group-specific variables
├── host_vars/          # Host-specific variables
├── ansible.cfg         # Ansible configuration
├── .ansible-lint       # Linting rules
├── requirements.txt    # Python dependencies (for CI/CD)
└── .ansible-version    # Version requirements documentation
```

## 🎯 Common Commands

### Inventory Management
```bash
# List all hosts
ansible-inventory --graph

# Show host details
ansible-inventory --host <hostname>

# List hosts in a specific group
ansible-inventory --graph <group_name>
```

### Running Playbooks
```bash
# Run a playbook
ansible-playbook playbooks/<playbook_name>.yml

# Run with specific inventory
ansible-playbook -i inventories/hosts.yml playbooks/<playbook_name>.yml

# Dry run (check mode)
ansible-playbook playbooks/<playbook_name>.yml --check

# Run with verbose output
ansible-playbook playbooks/<playbook_name>.yml -v
```

### Linting and Validation
```bash
# Lint all files
ansible-lint

# Lint specific playbook
ansible-lint playbooks/<playbook_name>.yml

# Validate playbook syntax
ansible-playbook playbooks/<playbook_name>.yml --syntax-check

# Check YAML syntax
yamllint .
```

### Formatting

```bash
# Format a file with Prettier (if not excluded)
npx prettier --write <file.yml>

# Check if a file would be formatted
npx prettier --check <file.yml>

# Format all non-excluded files
npx prettier --write "**/*.{yml,yaml,json,md}"
```

## 💅 Formatting Guidelines

### Formatting Strategy

We use **Prettier** as our single code formatter for consistency across all file types. However, some Ansible YAML files with custom column alignment are excluded from automatic formatting to preserve readability.

### Files Excluded from Prettier

The following files are excluded from Prettier formatting (see `.prettierignore`):
- `inventories/group_vars/all/network.yml` - Network configuration with aligned values
- `inventories/group_vars/all/main.yml` - Main configuration with aligned comments
- `inventories/hosts.yml` - Inventory with visual structure
- `inventories/z_*.yml` - User override files

### Using Prettier Ignore Comments

For temporary formatting exclusions within a file:

```yaml
# prettier-ignore
default_net:
  vlan_id:      1        # Aligned comment
  bridge_name:  "vmbr0"  # Aligned comment
  subnet:       "192.168.1.0/24"

# This section will be formatted normally
regular_config:
  key: value
  another_key: another_value
```

### When to Exclude Files

Only exclude files from Prettier when:
1. The file has intentional column alignment for readability
2. The alignment significantly improves comprehension (e.g., network configs, matrices)
3. Multiple team members agree the custom formatting adds value

### Manual Formatting

For excluded files, maintain consistency manually:
- Use 2-space indentation
- Align colons for related configuration blocks
- Keep line length under 120 characters
- Add explanatory comments for complex alignments

## 🔧 VS Code Features

When you open the workspace file, VS Code automatically:
- ✅ Enables Ansible language support with full IntelliSense
- ✅ Provides module documentation on hover
- ✅ Auto-completes module names and parameters
- ✅ Validates against Ansible schemas
- ✅ Runs ansible-lint in real-time
- ✅ Shows Python interpreter and Ansible version in status bar
- ✅ Formats files with Prettier on save (unless excluded)

## 📚 Documentation

- [Ansible Documentation](https://docs.ansible.com/)
- [Ansible Best Practices](https://docs.ansible.com/ansible/latest/user_guide/playbooks_best_practices.html)
- [ansible-lint Rules](https://ansible.readthedocs.io/projects/lint/rules/)
- See [`ANSIBLE_BEST_PRACTICES_ANALYSIS.md`](ANSIBLE_BEST_PRACTICES_ANALYSIS.md:1) for our enterprise configuration decisions

## 🐛 Troubleshooting

### Issue: "ansible: command not found"
**Solution:** Install Ansible using the package manager for your OS (see Prerequisites)

### Issue: VS Code not recognizing Ansible files
**Solution:** Open the workspace file (`Northern-Lights-Platform.code-workspace`), not just the folder

### Issue: Linting errors about FQCN
**Solution:** Use fully qualified collection names (e.g., `ansible.builtin.command` instead of `command`)

### Issue: Schema validation errors
**Solution:** Ensure your YAML files follow the correct structure for their type (playbook, inventory, etc.)

## 🤝 Contributing

1. Always run `ansible-lint` before committing
2. Use meaningful task names
3. Follow the existing directory structure
4. Document any new roles or playbooks
5. Use FQCN for better clarity and future compatibility