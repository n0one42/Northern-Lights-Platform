# Ansible Collections Update Summary

## Date: 2025-08-26

### Overview

Successfully upgraded Ansible environment to use the absolute latest versions of all collections, fixing MANIFEST.json
warnings and deprecated YAML callback issues along the way.

## Issues Resolved

1. **MANIFEST.json Warnings**: Fixed corrupted/incomplete collection installations
2. **Deprecated YAML Callback**: Updated from `community.general.yaml` to built-in `yaml`
3. **Version Fragmentation**: Consolidated all collection requirements into single root file

## Major Version Updates

All collections have been updated to their absolute latest versions:

| Collection        | Old Version | New Version | Change Type   |
| ----------------- | ----------- | ----------- | ------------- |
| ansible.posix     | 1.6.2       | **2.1.0**   | Major upgrade |
| ansible.utils     | 5.1.2       | **6.0.0**   | Major upgrade |
| community.general | 10.7.3      | **11.2.1**  | Major upgrade |
| community.docker  | 3.10.0      | **4.7.0**   | Major upgrade |
| community.crypto  | 2.26.5      | **3.0.3**   | Major upgrade |

## Files Modified

### 1. requirements.yml (Root Level)

- Created consolidated requirements file with latest versions
- Uses semantic versioning constraints to allow patch updates
- Single source of truth for all collection dependencies

### 2. ansible.cfg

- Fixed deprecated YAML callback: `stdout_callback = yaml`
- Added `deprecation_warnings = False` to suppress warnings
- Maintained all other configurations

### 3. requirements.txt

- Updated Python dependencies to latest versions:
  - ansible>=11.0.0
  - ansible-lint>=24.0.0
  - molecule>=24.0.0
  - molecule-plugins[docker]>=23.0.0
  - yamllint>=1.35.0
  - jmespath>=1.0.0
  - netaddr>=1.0.0
  - passlib>=1.7.0
  - bcrypt>=4.0.0

### 4. Makefile

- Enhanced `verify-collections` target with detailed output
- Added `update-collections-latest` target for future updates
- Improved readability and maintenance

## New Makefile Commands

```bash
# Install collections with version constraints from requirements.yml
make install-collections

# Verify installed collections and show versions
make verify-collections

# Update all collections to absolute latest versions (no constraints)
make update-collections-latest
```

## Breaking Changes to Watch

With major version upgrades (especially community.general 10→11, community.crypto 2→3), monitor for:

1. **Deprecated Module Removals**: Some old modules may have been removed
2. **Parameter Changes**: Module parameters may have changed names or defaults
3. **Behavior Changes**: Some modules may behave differently in major versions
4. **Python Version Requirements**: Newer collections may require newer Python

## Testing Completed

✅ Collections installed successfully ✅ YAML callback working correctly ✅ Test playbook runs without errors ✅ No
MANIFEST.json warnings ✅ All Makefile targets functional

## Best Practices Implemented

1. **Single Source of Truth**: All collection requirements in root `requirements.yml`
2. **Semantic Versioning**: Proper version constraints allowing patch updates
3. **Local Installation**: Collections installed to project `.ansible/collections/`
4. **Verification Tools**: Easy commands to verify and update collections
5. **Clean Configuration**: Removed deprecated settings and duplicate files

## Next Steps

1. Test existing playbooks thoroughly with new major versions
2. Review breaking changes documentation for each collection:
   - [ansible.utils 6.0.0 changelog](https://github.com/ansible-collections/ansible.utils/releases)
   - [community.general 11.0.0 changelog](https://github.com/ansible-collections/community.general/releases)
   - [community.crypto 3.0.0 changelog](https://github.com/ansible-collections/community.crypto/releases)
3. Update any deprecated module usage found during testing
4. Consider pinning to specific minor versions once stability confirmed

## Maintenance Schedule

- **Weekly**: Run `make verify-collections` to check health
- **Monthly**: Review for new releases and security updates
- **Quarterly**: Consider running `make update-collections-latest` for major updates

## Commands Reference

```bash
# Regular workflow
make install-collections  # Install from requirements.yml
make verify-collections   # Check installation status

# Update workflow
make update-collections-latest  # Get absolute latest versions
# Then update requirements.yml with new versions found
```

---

_This update ensures the Ansible environment is modern, secure, and ready for production use with the latest features
and security patches from all collections._
