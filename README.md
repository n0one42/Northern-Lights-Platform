# Northern-Lights-Platform

A declarative, secure-by-default platform for provisioning and managing enterprise-grade infrastructure on Proxmox VE using Ansible, Docker, and Cloud-Init.
Ideal for both powerful homelabs and scalable enterprise environments.

This monorepo contains all the components needed to provision and manage the platform.

## Core Technologies

- **Orchestration & Provisioning:** [Ansible](https://www.ansible.com/)
- **Virtualization:** [Proxmox VE](https://www.proxmox.com/en/)
- **Development Environment:** [Python](https://www.python.org/) with `pyenv` and `venv`.

## Project Structure

This repository is organized by tooling. Each major component has its own directory with specific setup instructions.

- `ansible/`: Contains the Ansible playbooks, roles, and inventory for managing the entire infrastructure. **To get started, begin here.**
- `_docs/`: Contains project-wide documentation, including Architecture Decision Records (ADRs).
- `kestra/`: (Placeholder) for Kestra workflow definitions.
- `terraform/`: (Placeholder) for Terraform configurations.

## Getting Started

To begin setting up the platform, please navigate to the Ansible engine's documentation:

**➡️ [Start with the Ansible Engine Setup](./ansible/README.md)**
