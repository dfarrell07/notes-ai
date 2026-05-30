# Ansible Workstation Playbook Testing Research

Date: 2026-05-29

Research into testing an Ansible workstation provisioning playbook targeting
Fedora, RHEL CSB, and macOS. Developer has libvirt/qemu/KVM on a Fedora laptop.

## 1. Molecule: Current State (2025-2026)

Molecule remains the standard Ansible testing framework. It is actively maintained
as part of the Ansible by Red Hat project, with frequent releases (latest: v26.4.0,
April 2026). Python 3.10+ required. Supports latest two major Ansible versions.

Key facts:

- Officially supports testing **playbooks**, not just roles. Dedicated documentation
  at docs.ansible.com/projects/molecule/getting-started-playbooks/
- Default verifier changed to **Ansible** (verify.yml with assert tasks) instead of
  Testinfra in Molecule 6
- Full test lifecycle: dependency, create, prepare, converge, **idempotence**, verify,
  cleanup, destroy
- Idempotence check runs the playbook twice and fails if the second run has any
  changed tasks

### Molecule Drivers for VMs (not just containers)

Multiple options exist for using Molecule with libvirt/qemu VMs:

| Driver | Backend | Notes |
|--------|---------|-------|
| molecule-plugins[vagrant] | Vagrant + libvirt | Most mature VM option. Uses Vagrant box ecosystem. Install with `pip install molecule molecule-plugins[vagrant]` |
| molecule-libvirt | libvirt directly | No Vagrant dependency. Uses community.libvirt collection. Configures qemu_user, CPU model, networking directly |
| molecule-qemu | QEMU directly | Designed for Apple M1/ARM. Minimal dependencies. Note: may have compatibility issues with Molecule 0.6.x which dropped third-party driver support |
| molecule-virtup | virt-up + libvirt | Template-based VM creation, uses qemu:///session by default |

**Recommendation**: molecule-plugins[vagrant] with the libvirt provider is the most
reliable and best-documented option for a Fedora laptop with KVM.

Sources:
- https://pypi.org/project/molecule/
- https://docs.ansible.com/projects/molecule/getting-started-playbooks/
- https://github.com/ansible/molecule
- https://computingforgeeks.com/ansible-molecule-testing/
- https://www.endpointdev.com/blog/2025/03/testing-ansible-with-molecule/

## 2. Vagrant + libvirt

Still relevant and well-supported. The vagrant-libvirt plugin (v0.12.2) is actively
maintained. Fedora treats it as the preferred/default Vagrant provider.

### Installation on Fedora

Use Fedora-packaged versions (not HashiCorp upstream) to avoid compilation issues:

```bash
sudo dnf install libvirt vagrant vagrant-libvirt vagrant-sshfs
```

Or install the package groups: `@virtualization` and `@vagrant`.

### Fedora Vagrant Boxes Available

| Version | Box Name | Support |
|---------|----------|---------|
| Fedora 43 | fedora/43-cloud-base or alvistack/fedora-43 | Active through Dec 2026 |
| Fedora 42 | fedora/42-cloud-base or alvistack/fedora-42 | Active through May 2026 |

Official download: https://fedoraproject.org/cloud/download/index.html

Fedora 43 vagrant-libvirt boxes available for both x86_64 and aarch64.

### Automated Workflow

Yes, you can fully automate: spin up Fedora VM, run playbook, verify idempotency,
destroy. Either through Molecule (which wraps Vagrant) or directly with a Vagrantfile
+ shell script.

### Caveats

- Vagrant 2.3.4 was the last version fully compatible with vagrant-libvirt; versions
  2.3.6/2.3.7 had initialization errors. Use Fedora-packaged versions to avoid this.
- By default on Fedora, `qemu:///session` is used (userspace, no root needed) but some
  features requiring root may not work.

Sources:
- https://developer.fedoraproject.org/tools/vagrant/vagrant-libvirt.html
- https://github.com/vagrant-libvirt/vagrant-libvirt
- https://osbuild.org/docs/user-guide/image-descriptions/fedora-43/generic-vagrant-libvirt/

## 3. Podman/Docker Containers for Testing

### What Works in Containers

- Package installation (dnf/apt/brew)
- Configuration file management (dotfiles, shell config, editor config)
- User creation and group management
- Git configuration
- Python/pip, Node/npm setup
- Application configuration

### What Breaks in Containers

- **systemd services** - Requires privileged mode + cgroup mounts. Possible but fragile.
- **Firewall rules** (iptables/nftables/firewalld) - Requires NET_ADMIN capability
- **Kernel modules** (modprobe)
- **NFS mounts** and disk management (LVM, partitioning)
- **Kernel parameters** (sysctl)
- **Network stack** (bridge setup, VLAN, bonding)
- **Desktop environment** packages (GNOME, display managers)
- **Hardware-dependent** configuration (power management, bluetooth, etc.)

### Verdict for Workstation Playbook

Containers are useful for a **fast feedback loop** on dotfiles/packages/config roles.
Not useful for system-level roles (services, firewall, desktop environment). A two-tier
strategy works well:

1. **Fast tier**: Molecule + Podman containers for dotfiles, packages, configs
2. **Full tier**: Molecule + Vagrant/libvirt VMs for system-level roles and full
   integration testing

Sources:
- https://blog.carlosnunez.me/post/testing-ansible-playbooks-using-systemd-in-docker/
- https://www.jeffgeerling.com/blog/2019/how-i-test-ansible-configuration-on-7-different-oses-docker

## 4. Cloud VMs for Testing

Cloud VMs (GCP, AWS) are viable for CI but less practical for iterative development.

### Pros
- Fedora cloud images exist on all major providers
- No local hardware dependency
- Clean environment every run
- Good for CI/CD pipelines

### Cons
- Slower feedback loop than local VMs (VM startup, network latency)
- Cost (even if small)
- Requires cloud credentials management
- Network-dependent

### Verdict

Cloud VMs are best suited for **CI validation** (post-push). For local development,
libvirt VMs on the laptop are faster and free. If you add CI later, cloud VMs are a
natural extension.

## 5. macOS Testing

### The Hard Truth

There are **no legal macOS VMs on Linux**. Apple's EULA requires macOS to run on
Apple hardware. This eliminates local VM testing for macOS roles.

### Options

**GitHub Actions macOS Runners** (best option):
- GitHub provides macOS runners including macOS 26 (generally available since Feb 2026)
- Free for public repos, included minutes for private repos
- Jeff Geerlingguy's mac-dev-playbook uses this approach
- Limitation: some GUI operations and App Store interactions may not work
- Good enough for: Homebrew installs, dotfiles, CLI tools, shell config

**Tart** (if you have Apple Silicon hardware):
- Open source, uses Apple's Virtualization.Framework for near-native performance
- Install via `brew install cirruslabs/cli/tart`
- Supports macOS Sequoia, Tahoe (macOS 26)
- Acquired by OpenAI in April 2026 (project status may evolve)
- Only runs on Apple Silicon Macs (M1/M2/M3/M4)

**Practical Approach**:
- Test macOS roles in GitHub Actions CI with macOS runners
- Accept that macOS testing will be CI-only unless you have a Mac
- Factor the playbook so macOS-specific roles are isolated and can be tested
  independently

Sources:
- https://github.com/geerlingguy/mac-dev-playbook
- https://github.blog/changelog/2026-02-26-macos-26-is-now-generally-available-for-github-hosted-runners/
- https://tart.run/
- https://github.com/cirruslabs/tart

## 6. RHEL Testing

### Option 1: CentOS Stream (Best Proxy)

CentOS Stream is the upstream for RHEL. CentOS Stream 10 is the basis for RHEL 10.
Same package names, same paths, same systemd units. Very high fidelity for testing.

Available as:
- Container images (for Molecule + Podman/Docker)
- Cloud images
- Vagrant boxes
- Full ISOs

### Option 2: Red Hat Developer Subscription (Free Actual RHEL)

**Two programs available:**

1. **Developer Subscription for Individuals** (long-standing): Up to 16 systems,
   no cost. Can be used for development AND small-scale production.

2. **RHEL for Business Developers** (launched July 2025): Up to 25 entitlements,
   no cost, self-service signup. Designed for development and testing within
   organizations.

Both available at https://developers.redhat.com/products/rhel/download

RHEL 10.0 released May 12, 2026. RHEL 9.6 also available.

Self-support only (no Red Hat support tickets), but you get full software,
updates, and security patches.

### Option 3: Rocky Linux / AlmaLinux

Bug-for-bug RHEL rebuilds. Rocky 10.1 and AlmaLinux 10 both available. Good
substitutes if you want to avoid subscription management entirely.

### Recommendation

Use **CentOS Stream** as the primary test target (it is what RHEL is built from).
If you need to validate against actual RHEL, use the **free developer subscription**
for a second test target.

Sources:
- https://developers.redhat.com/articles/faqs-no-cost-red-hat-enterprise-linux
- https://developers.redhat.com/articles/2025/07/09/announcing-self-service-access-red-hat-enterprise-linux-business-developers

## 7. Testinfra vs Ansible Assert

### Molecule Default: Ansible verify.yml

Since Molecule 6, the **default verifier is Ansible itself** (verify.yml with
assert tasks). This is the path of least resistance.

Example verify.yml:
```yaml
- name: Verify
  hosts: all
  tasks:
    - name: Check git is installed
      ansible.builtin.command: git --version
      changed_when: false

    - name: Check .bashrc exists
      ansible.builtin.stat:
        path: ~/.bashrc
      register: bashrc_stat

    - name: Assert .bashrc exists
      ansible.builtin.assert:
        that: bashrc_stat.stat.exists
```

### When to Use Which

| Criteria | Ansible assert (verify.yml) | Testinfra (pytest) |
|----------|----------------------------|--------------------|
| Simple checks (2-5 tasks) | Preferred | Overkill |
| Complex assertion logic | Jinja becomes unwieldy | Full Python |
| Growing test suites | Hard to maintain | Scales with fixtures |
| Same execution context | Yes | No |
| CI output | Good | Excellent (pytest) |

### Recommendation for Workstation Playbook

Start with **Ansible verify.yml**. A workstation playbook has straightforward
verification needs:
- Is package X installed?
- Does config file Y exist with expected content?
- Is service Z running?

These are simple checks that Ansible assert handles cleanly. Move to Testinfra only
if verification logic gets complex.

Sources:
- https://medium.com/opsops/choosing-between-testinfra-and-ansible-for-tests-e52a9329b3ec
- https://docs.ansible.com/projects/molecule/

## 8. Full Automation: The `make test` Stack

### Simplest Stack

**Molecule + Vagrant + libvirt** wrapped in a Makefile.

```makefile
.PHONY: test lint test-container test-vm

lint:
	ansible-lint playbook.yml

test-container:
	molecule test --scenario-name container

test-vm:
	molecule test --scenario-name vm

test: lint test-vm
```

The `molecule test` command already does the full sequence:
1. Create (spin up VM via Vagrant)
2. Converge (run the playbook)
3. Idempotence (run again, assert 0 changed)
4. Verify (run verify.yml assertions)
5. Destroy (tear down VM)

### Project Structure

```
workstation-playbook/
  playbook.yml
  inventory/
  roles/
  molecule/
    container/          # fast tests with Podman
      molecule.yml
      converge.yml
      verify.yml
    vm/                 # full VM tests with Vagrant+libvirt
      molecule.yml
      converge.yml
      verify.yml
  Makefile
```

### molecule.yml for VM Scenario

```yaml
driver:
  name: vagrant
  provider:
    name: libvirt

platforms:
  - name: fedora-test
    box: fedora/43-cloud-base
    memory: 4096
    cpus: 2

provisioner:
  name: ansible

verifier:
  name: ansible

scenario:
  test_sequence:
    - dependency
    - create
    - converge
    - idempotence
    - verify
    - destroy
```

Sources:
- https://www.tauceti.blog/posts/testing-ansible-roles-with-molecule-libvirt-vagrant-qemu-kvm/
- https://trustedsec.com/blog/automation-testing-with-ansible-molecule-and-vagrant
- https://gianglabs.github.io/glabs-docs-blogs/blog/hpc-ansible-molecule-playbook-testing

## 9. Snapshot/Rollback

### libvirt Snapshots with virsh

vagrant-libvirt does **not** natively support snapshots like VirtualBox. Use virsh
directly:

```bash
# Create snapshot
virsh snapshot-create-as <vm-name> --name before-test

# Revert to snapshot
virsh snapshot-revert <vm-name> --snapshotname before-test --running

# Delete snapshot
virsh snapshot-delete <vm-name> --snapshotname before-test
```

### Use Cases

- **Iterative development**: Snapshot a clean Fedora VM, run playbook, inspect
  results, rollback, tweak, repeat. Faster than destroy/recreate cycle.
- **Debugging**: Snapshot before a failing role, revert after investigating.
- **Not needed for CI**: Molecule's create/destroy cycle is sufficient for
  automated testing.

### Practical Workflow

```bash
# One-time setup
vagrant up
virsh snapshot-create-as workstation_fedora-test --name clean

# Development loop
ansible-playbook playbook.yml -i inventory
# inspect results...
virsh snapshot-revert workstation_fedora-test --snapshotname clean --running
# tweak playbook, repeat
```

Restoring from snapshot is up to 10x faster than full VM recreation.

Sources:
- https://ansible-development.readthedocs.io/en/master/provider/libvirt/
- https://linuxhaxor.net/code/kvm_snapshots_libvirt.html

## 10. CI Integration

### GitHub Actions

**Container-based testing**: Works out of the box. Molecule + Podman/Docker on
GitHub-hosted Linux runners.

**VM-based testing**: Nested virtualization (KVM) is **not officially supported** on
GitHub-hosted runners. `/dev/kvm` sometimes exists on free runners but it is
inconsistent and undocumented.

### Options for VM Testing in CI

| Approach | Pros | Cons |
|----------|------|------|
| GitHub-hosted + containers only | Free, reliable, fast | No real VM testing |
| Self-hosted runner (bare metal) | Full KVM access | Must maintain hardware, security concerns for public repos |
| Actuated (third-party) | Managed runners with KVM | Cost |
| Cloud VM provisioned in CI | No nested virt needed | Slower, cloud costs, credential management |

### macOS CI

GitHub provides macOS runners (including macOS 26, GA since Feb 2026). These work
for Ansible playbook testing. No nested virtualization, but the runner itself is the
test target.

### Practical CI Strategy

1. **Every PR**: Molecule + containers (Podman) -- lint, converge, idempotence,
   verify for dotfiles/packages/config roles
2. **Nightly or weekly**: Full VM test on self-hosted runner (or manual local run)
3. **macOS**: GitHub Actions macOS runner for macOS-specific roles

Sources:
- https://github.com/orgs/community/discussions/8305
- https://actuated.com/blog/kvm-in-github-actions
- https://github.blog/changelog/2026-02-26-macos-26-is-now-generally-available-for-github-hosted-runners/

## Summary: Recommended Stack

For a workstation playbook targeting Fedora, RHEL CSB, and macOS:

### Local Development
- **Molecule + Vagrant + libvirt** for full VM testing (Fedora, CentOS Stream)
- **virsh snapshots** for fast iterative development loops
- **Molecule + Podman** for quick feedback on package/config roles

### CI (GitHub Actions)
- **Container-based Molecule tests** on every PR (fast, free)
- **macOS runner** for macOS roles
- **Self-hosted runner** or **manual local run** for full VM tests (optional)

### Test Targets
- **Fedora 43** vagrant-libvirt box (primary)
- **CentOS Stream 10** container or VM (RHEL proxy)
- **macOS 26 GitHub runner** (macOS)
- **RHEL via free developer subscription** (optional second RHEL target)

### Verification
- **Ansible verify.yml** with assert tasks (Molecule default)
- Upgrade to **Testinfra** only if verification needs grow complex

### One Command
```bash
make test  # runs: ansible-lint, molecule test (create, converge, idempotence, verify, destroy)
```
