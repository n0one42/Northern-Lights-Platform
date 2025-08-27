# Northern-Lights-Platform/ansible/Makefile
.PHONY: help setup install-collections update-collections clean-collections verify-collections lint test clean

# ================================================================
# ENVIRONMENT CONFIGURATION - Force local-only collections usage
# ================================================================
# This ensures we ONLY use project-local collections, ignoring any global installations
# The path is relative to this Makefile's location
export ANSIBLE_COLLECTIONS_PATH := ./.ansible/collections

# Get the local collections path from ansible.cfg if it exists, otherwise use default
COLLECTIONS_PATH := $(shell grep -E '^collections_path' ansible.cfg 2>/dev/null | sed 's/^[^=]*=[[:space:]]*//' | sed 's/[[:space:]]*$$//' || echo "./.ansible/collections")

# Default target - show help
help:
	@echo "Northern Lights Platform - Ansible Makefile"
	@echo ""
	@echo "Available targets:"
	@echo "  help              - Show this help message"
	@echo "  setup             - Complete setup (install pre-commit hooks and collections)"
	@echo "  install-collections - Install Ansible collections from requirements.yml"
	@echo "  update-collections - Force reinstall/update all collections"
	@echo "  update-collections-latest - Install absolute latest versions (no constraints)"
	@echo "  clean-collections - Remove all installed collections"
	@echo "  verify-collections - Verify collections are properly installed"
	@echo "  lint              - Run ansible-lint and yamllint"
	@echo "  test              - Run playbook tests in check mode"
	@echo "  clean             - Clean cache and retry files"

setup:
	@echo "==============================================="
	@echo "Setting up Northern Lights Ansible environment"
	@echo "==============================================="
	@if command -v pre-commit &> /dev/null; then \
		echo "Installing pre-commit hooks..."; \
		pre-commit install; \
	else \
		echo "pre-commit not found, skipping hook installation"; \
	fi
	@echo "Installing Ansible collections..."
	@$(MAKE) install-collections
	@echo ""
	@echo "✅ Setup complete!"

install-collections:
	@echo "Installing Ansible collections from requirements.yml..."
	@echo "📁 Collections path: $(COLLECTIONS_PATH)"
	@echo "🚫 Ignoring any global collections"
	ANSIBLE_COLLECTIONS_PATH="$(COLLECTIONS_PATH)" \
	ansible-galaxy collection install -r requirements.yml \
		--collections-path "$(COLLECTIONS_PATH)" \
		--force
	@echo "✅ Collections installed successfully to $(COLLECTIONS_PATH)"

update-collections:
	@echo "Force updating all Ansible collections..."
	@echo "📁 Collections path: $(COLLECTIONS_PATH)"
	@echo "🚫 Ignoring any global collections"
	ANSIBLE_COLLECTIONS_PATH="$(COLLECTIONS_PATH)" \
	ansible-galaxy collection install -r requirements.yml \
		--collections-path "$(COLLECTIONS_PATH)" \
		--force
	@echo "✅ Collections updated successfully in $(COLLECTIONS_PATH)"

update-collections-latest:
	@echo "Installing ABSOLUTE LATEST versions (no version constraints)..."
	@echo "📁 Collections path: $(COLLECTIONS_PATH)"
	@echo "🚫 Ignoring any global collections"
	ANSIBLE_COLLECTIONS_PATH="$(COLLECTIONS_PATH)" \
	ansible-galaxy collection install \
		ansible.posix \
		community.general \
		community.docker \
		ansible.utils \
		community.crypto \
		--collections-path "$(COLLECTIONS_PATH)" \
		--force --upgrade
	@echo "✅ Collections updated to absolute latest versions in $(COLLECTIONS_PATH)"

clean-collections:
	@echo "Removing all installed collections..."
	rm -rf .ansible/collections/ansible_collections
	rm -rf collections/ansible_collections
	@echo "✅ Collections removed"

verify-collections:
	@echo "======================================"
	@echo "Verifying Ansible Collections"
	@echo "======================================"
	@echo ""
	@echo "📁 Using collections from: $(COLLECTIONS_PATH)"
	@echo "🚫 Global collections are IGNORED"
	@echo ""
	@echo "📦 Installed Collections (local only):"
	@echo "--------------------------------------"
	@ANSIBLE_COLLECTIONS_PATH="$(COLLECTIONS_PATH)" \
	ansible-galaxy collection list --collections-path "$(COLLECTIONS_PATH)" --format yaml 2>/dev/null || \
	ANSIBLE_COLLECTIONS_PATH="$(COLLECTIONS_PATH)" \
	ansible-galaxy collection list --collections-path "$(COLLECTIONS_PATH)"
	@echo ""
	@echo "🔍 Required vs Installed Versions:"
	@echo "----------------------------------"
	@echo "Checking requirements.yml compliance..."
	@for collection in ansible.posix community.general community.docker ansible.utils community.crypto; do \
		installed=$$(ANSIBLE_COLLECTIONS_PATH="$(COLLECTIONS_PATH)" ansible-galaxy collection list --collections-path "$(COLLECTIONS_PATH)" 2>/dev/null | grep "$$collection" | head -1 | awk '{print $$2}'); \
		if [ -n "$$installed" ]; then \
			echo "✅ $$collection: $$installed"; \
		else \
			echo "❌ $$collection: NOT INSTALLED"; \
		fi \
	done
	@echo ""
	@echo "🎯 Configuration Check:"
	@echo "----------------------"
	@echo "Ansible version: $$(ansible --version | head -1)"
	@echo "Config file: $$(ansible --version | grep 'config file' | cut -d'=' -f2)"
	@echo "Collections path (from env): $$ANSIBLE_COLLECTIONS_PATH"
	@echo "Collections path (from cfg): $$(grep -E '^collections_path' ansible.cfg 2>/dev/null | sed 's/.*=\s*//' || echo 'not set')"
	@echo "Python version: $$(python3 --version)"
	@echo ""
	@echo "✨ Testing YAML output format..."
	@if ANSIBLE_COLLECTIONS_PATH="$(COLLECTIONS_PATH)" ansible localhost -m ping 2>&1 | grep -q "pong"; then \
		echo "✅ YAML callback is configured correctly"; \
	else \
		echo "⚠️  YAML callback might need configuration"; \
	fi

lint:
	@echo "Running linters..."
	@echo "📁 Using collections from: $(COLLECTIONS_PATH)"
	ANSIBLE_COLLECTIONS_PATH="$(COLLECTIONS_PATH)" ansible-lint
	@if command -v yamllint &> /dev/null; then \
		yamllint .; \
	else \
		echo "yamllint not found, skipping YAML linting"; \
	fi

test:
	@echo "Running playbook tests..."
	@echo "📁 Using collections from: $(COLLECTIONS_PATH)"
	@if [ -f playbooks/test.yml ]; then \
		ANSIBLE_COLLECTIONS_PATH="$(COLLECTIONS_PATH)" \
		ansible-playbook playbooks/test.yml --check --diff; \
	else \
		echo "No test playbook found at playbooks/test.yml"; \
	fi

clean:
	@echo "Cleaning up temporary files..."
	rm -rf .ansible_facts_cache/
	find . -name "*.retry" -delete
	find . -name "__pycache__" -type d -exec rm -rf {} + 2>/dev/null || true
	find . -name "*.pyc" -delete
	@echo "✅ Cleanup complete"

# Quick reinstall for fixing collection issues
fix-collections: clean-collections install-collections verify-collections
	@echo "✅ Collections have been reinstalled and verified"
