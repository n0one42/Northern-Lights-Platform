# Checking ansible-lint vs yamllint Compatibility Guide

## Overview

When ansible-lint runs, it internally uses yamllint for YAML validation. However, if your `.yamllint` configuration
doesn't match ansible-lint's requirements, you'll get warnings or errors.

## How to Check for Compatibility Issues

### 1. Run ansible-lint in Verbose Mode

```bash
# This will show warnings about yamllint incompatibilities
ansible-lint -v playbooks/test.yml
```

Look for warnings like:

```
WARNING  Found incompatible custom yamllint configuration (.yamllint)...
```

### 2. Check ansible-lint's Required yamllint Settings

ansible-lint **requires** these specific yamllint settings:

| Setting                              | Required Value | Reason                                          |
| ------------------------------------ | -------------- | ----------------------------------------------- |
| `comments-indentation`               | `false`        | ansible-lint handles comment indentation itself |
| `braces.min-spaces-inside`           | `0`            | Matches ansible-lint's formatting expectations  |
| `octal-values.forbid-explicit-octal` | `true`         | Security: prevents ambiguous octal notation     |

### 3. How to Fix Incompatibilities

#### Step 1: Check Current yamllint Config

```bash
# View your current settings
cat .yamllint | grep -E "comments-indentation|braces|octal"
```

#### Step 2: Update .yamllint File

```yaml
rules:
  # Required for ansible-lint compatibility
  comments-indentation: false # Must be false

  braces:
    min-spaces-inside: 0 # Must be 0
    max-spaces-inside: 1

  octal-values:
    forbid-implicit-octal: true
    forbid-explicit-octal: true # Must be true
```

#### Step 3: Test the Fix

```bash
# Should pass without warnings
ansible-lint playbooks/test.yml
```

## Finding the Requirements

### Method 1: Read the Warning Message

ansible-lint tells you exactly what's wrong:

```
WARNING  Found incompatible custom yamllint configuration (.yamllint), please either remove the file or edit it to comply with:
  - comments-indentation must be false
  - braces.min-spaces-inside must be 0
  - octal-values.forbid-explicit-octal must be true
```

### Method 2: Check ansible-lint Documentation

```bash
# Check the official docs
open https://ansible.readthedocs.io/projects/lint/rules/yaml/
```

### Method 3: Use Default Configuration

```bash
# Generate ansible-lint's default yamllint config
ansible-lint --write=.yamllint-ansible-lint
# Then compare with your .yamllint
diff .yamllint .yamllint-ansible-lint
```

## Quick Diagnostic Script

Create this script to check compatibility:

```bash
#!/bin/bash
# check-lint-compat.sh

echo "Checking ansible-lint vs yamllint compatibility..."

# Check if files exist
if [ ! -f .yamllint ]; then
    echo "❌ No .yamllint file found"
    exit 1
fi

# Check required settings
echo "Checking required settings..."

# Check comments-indentation
if grep -q "comments-indentation: false" .yamllint; then
    echo "✅ comments-indentation: false"
else
    echo "❌ comments-indentation must be false"
fi

# Check braces.min-spaces-inside
if grep -q "min-spaces-inside: 0" .yamllint; then
    echo "✅ braces.min-spaces-inside: 0"
else
    echo "❌ braces.min-spaces-inside must be 0"
fi

# Check octal-values.forbid-explicit-octal
if grep -q "forbid-explicit-octal: true" .yamllint; then
    echo "✅ octal-values.forbid-explicit-octal: true"
else
    echo "❌ octal-values.forbid-explicit-octal must be true"
fi

# Run ansible-lint to confirm
echo -e "\nRunning ansible-lint check..."
if ansible-lint --version >/dev/null 2>&1; then
    ansible-lint 2>&1 | grep -i "yamllint" || echo "✅ No yamllint warnings found"
else
    echo "⚠️  ansible-lint not installed"
fi
```

## Common Pitfalls

1. **Different Formatting Preferences**: yamllint and ansible-lint may have different opinions on formatting. Always
   defer to ansible-lint for Ansible files.

2. **Version Differences**: Newer versions of ansible-lint may have different requirements. Always check after
   upgrading:

   ```bash
   ansible-lint --version
   ```

3. **Multiple Config Files**: Make sure you don't have multiple yamllint configs:

   ```bash
   find . -name ".yamllint*" -o -name "yamllint"
   ```

## Best Practices

1. **Use ansible-lint's Defaults**: For Ansible projects, prioritize ansible-lint's requirements over general yamllint
   preferences.

2. **Document Exceptions**: If you need to override something, document why:

   ```yaml
   # .yamllint
   # Note: These settings are required for ansible-lint compatibility
   # DO NOT CHANGE without testing ansible-lint
   rules:
     comments-indentation: false # Required by ansible-lint
   ```

3. **Test in CI/CD**: Always run both tools in your pipeline:

   ```yaml
   # .github/workflows/lint.yml
   - name: Lint with yamllint
     run: yamllint .

   - name: Lint with ansible-lint
     run: ansible-lint
   ```

## Summary

The key to compatibility is understanding that ansible-lint has specific yamllint requirements. When you see warnings:

1. Read the warning message carefully - it tells you exactly what to fix
2. Update your `.yamllint` file with the required values
3. Test with `ansible-lint` to confirm the fix

Remember: ansible-lint uses yamllint internally, so your yamllint config must be compatible with ansible-lint's
expectations.
