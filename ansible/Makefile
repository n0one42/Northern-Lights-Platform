# Northern-Lights-Platform/ansible/Makefile
.PHONY: lint test setup clean

setup:
	@echo "Installing pre-commit hooks..."
	pre-commit install
	@echo "Installing Ansible collections..."
	ansible-galaxy collection install -r requirements.yml

lint:
	ansible-lint
	yamllint .

test:
	ansible-playbook playbooks/test.yml --check --diff

clean:
	rm -rf .ansible_facts_cache/
	find . -name "*.retry" -delete