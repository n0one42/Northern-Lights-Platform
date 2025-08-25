# Project Roadmap

This document outlines the future direction and planned features for the Northern-Lights-Platform project.

## Near-Term Goals (v1.0)

The primary focus is to achieve a fully functional and stable version of the architecture described in the
documentation.

- [ ] Complete the implementation of all core Ansible roles required for the Ingress, Metrics, and Application VMs.
- [ ] Ensure all services (Traefik, Grafana, Loki, etc.) are correctly configured and integrated.
- [ ] Finalize and document the default variable structure for the out-of-the-box experience.

## Future Enhancements (Post-v1.0)

These are planned improvements that will be addressed after the core functionality is complete.

- **Automated Testing:** Implement Molecule for comprehensive, automated testing of all Ansible roles to ensure
  reliability and stability during development.
- **CI/CD Pipeline:** Introduce a GitHub Actions workflow to automate linting, validation, and testing on all code
  changes, creating a robust quality gate.
- **Expanded Provider Support:** Investigate abstracting Ansible roles to support other virtualization providers and
  cloud platforms beyond Proxmox.
- **Enhanced Secrets Management:** Integrate Ansible Vault for encrypting sensitive variables, allowing them to be
  safely committed to the repository.
