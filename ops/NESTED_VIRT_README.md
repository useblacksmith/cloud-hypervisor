# Nested Virtualization Setup for Cloud Hypervisor

This guide helps you enable and verify nested virtualization on your x86_64 hosts running cloud-hypervisor, allowing Windows VMs to run Docker containers and WSL2.

## Quick Start

### 1. Check Current Status

First, check which hosts have nested virtualization enabled:

```bash
cd ops
ansible-playbook -i inventory.ini check_nested_virt.yml
```

This will show you for each x86_64 host:
- CPU vendor (Intel/AMD)
- Whether nested virtualization is enabled
- Cloud Hypervisor installation status
- Number of running VMs

**ARM hosts are automatically skipped** (only x86_64 hosts are checked).

### 2. Enable Nested Virtualization

To enable nested virtualization on all x86_64 hosts:

```bash
cd ops
ansible-playbook -i inventory.ini enable_nested_virt.yml
```

This playbook will:
- Create persistent configuration in `/etc/modprobe.d/kvm-nested.conf`
- Attempt to reload KVM modules immediately (if no VMs are running)
- Show which hosts need a reboot

### 3. Reboot Hosts (if required)

If VMs were running or module reload failed, you'll need to reboot:

```bash
cd ops
# Reboot specific hosts
ansible staging -i inventory.ini -m reboot -b --limit "host1,host2"

# Or reboot all hosts (be careful!)
ansible staging -i inventory.ini -m reboot -b
```

### 4. Verify After Reboot

After rebooting, verify nested virtualization is working:

```bash
cd ops
ansible-playbook -i inventory.ini check_nested_virt.yml
```

You should see `✓ NESTED VIRTUALIZATION: ENABLED` for all x86_64 hosts.

## What This Enables

Once nested virtualization is enabled:

1. **Cloud Hypervisor automatically exposes VMX/SVM to guest VMs** - no additional configuration needed
2. **Windows VMs can enable Hyper-V** and run virtualized workloads
3. **WSL2 will work** in Windows VMs (requires Hyper-V)
4. **Docker Desktop can run** in Windows VMs using WSL2 backend
5. **Linux containers in Docker** will work for your customers' test suites

## How It Works

### Intel CPUs
- Sets `options kvm_intel nested=1` in `/etc/modprobe.d/kvm-nested.conf`
- Exposes VMX (Virtual Machine Extensions) to guests

### AMD CPUs
- Sets `options kvm_amd nested=1` in `/etc/modprobe.d/kvm-nested.conf`
- Exposes SVM (Secure Virtual Machine) to guests

### In Cloud Hypervisor
- Cloud Hypervisor uses KVM's `get_supported_cpuid()` function
- If KVM has nested virtualization enabled, VMX/SVM features are automatically included
- No changes needed to your cloud-hypervisor configuration
- Windows VMs will automatically see the virtualization extensions

## Manual Verification on a Host

To manually check on a single host:

```bash
# SSH to the host
ssh ubuntu@100.123.214.111

# Check if enabled (Intel)
cat /sys/module/kvm_intel/parameters/nested
# Should show: Y or 1

# Check if enabled (AMD)
cat /sys/module/kvm_amd/parameters/nested
# Should show: Y or 1

# Verify in a running Windows VM
# From Windows PowerShell in the VM:
systeminfo | findstr /C:"Hyper-V Requirements"
# Should show "Yes" for all requirements
```

## Performance Considerations

- **L2 VMs are slower than L1 VMs** - this is expected with nested virtualization
- For test suites, the performance is usually acceptable
- Production workloads may see 10-30% performance degradation depending on workload
- I/O-bound workloads (like Docker builds) are less affected than CPU-bound ones

## Troubleshooting

### Nested virtualization shows as disabled after reboot
```bash
# Check if config file exists
cat /etc/modprobe.d/kvm-nested.conf

# Reload modules manually
sudo rmmod kvm_intel  # or kvm_amd
sudo rmmod kvm
sudo modprobe kvm
sudo modprobe kvm_intel nested=1  # or kvm_amd
```

### Windows VM doesn't show virtualization support
```bash
# In Windows VM, run PowerShell as Administrator:
Get-VMHost  # Should not error if Hyper-V is available
```

### Docker Desktop fails to start in Windows VM
1. Ensure WSL2 is installed in Windows
2. Enable "Use WSL2 based engine" in Docker Desktop settings
3. Check Windows Features: Hyper-V and WSL2 should be enabled

## Files

- `inventory.ini` - Ansible inventory with host IPs
- `check_nested_virt.yml` - Playbook to check status
- `enable_nested_virt.yml` - Playbook to enable nested virtualization
- `/etc/modprobe.d/kvm-nested.conf` - Created on each host (persistent config)

## Your Inventory

**x86_64 Hosts (will be configured):**
- US: 6 hosts (Intel/AMD)
- EU: 2 hosts (Intel/AMD)

**ARM64 Hosts (automatically skipped):**
- US: 2 ARM hosts
- EU: 1 ARM host
