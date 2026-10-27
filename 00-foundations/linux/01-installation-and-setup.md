# Linux: Installation and Lab Setup

| | |
|---|---|
| **Cluster** | 00 Foundations |
| **Role** | Prerequisite |
| **Prerequisites** | A laptop with 8 GB RAM (16 GB recommended) and 40 GB free disk |
| **Target version** | Ubuntu 24.04 LTS (primary), Fedora 40+ and Alpine 3.20+ (for package-manager deltas) |
| **Capstone phase** | Phase 1 (Months 1-4) |

> "Installation" for Linux means standing up a disposable lab machine you can break and rebuild in minutes, then installing the diagnostic toolbox used by every other file in this folder. After this file you will have an Ubuntu 24.04 VM (on WSL2, Multipass, Lima or a cloud instance), snapshot/rebuild ability, SSH access with a key, and `strace`, `perf`, `bpftrace`, `jq`, `yq` and friends verified working.

## 1. Choose a Lab Environment

| Option | Host OS | Pros | Cons | Use when |
|---|---|---|---|---|
| **WSL2** (Ubuntu 24.04) | Windows 11 | Zero cost, fast, integrates with VS Code | Shared Microsoft kernel; `systemd` needs enabling; some cgroup/eBPF labs limited | Daily driver on Windows |
| **Multipass** | Windows, macOS, Linux | Real Ubuntu VM with real kernel in one command; cloud-init support | Hyper-V/QEMU overhead; networking slightly different on each host | Full-fidelity labs (systemd, cgroups, namespaces) |
| **Lima** | macOS, Linux | Lightweight, Docker-free container host; Apple Silicon friendly | macOS-centric | Apple Silicon users |
| **Cloud VM** (EC2 `t3.small`) | any | Identical to production nodes; practice SSH hardening for real | Costs money; must remember to destroy | SSH hardening lab, sysctl labs |
| **Vagrant + VirtualBox/libvirt** | any | Reproducible `Vagrantfile`; multi-VM | Heavy; VirtualBox clashes with Hyper-V on Windows | Multi-node networking labs |

Recommendation: WSL2 for daily reading and text-processing practice, plus **Multipass** for the namespace/cgroup/systemd/eBPF labs. The commands below set up both.

## 2. Option A: WSL2 with Ubuntu 24.04 (Windows 11)

Run in an elevated PowerShell:

```powershell
wsl --install --distribution Ubuntu-24.04
wsl --set-default-version 2
wsl --update
```

Reboot if asked, open "Ubuntu 24.04" from the Start menu, create a user. Then enable systemd (required for `systemctl` labs):

```bash
sudo tee /etc/wsl.conf > /dev/null <<'EOF'
[boot]
systemd=true

[automount]
options = "metadata,umask=22,fmask=11"

[network]
generateResolvConf = true
EOF
```

Back in PowerShell:

```powershell
wsl --shutdown
wsl --distribution Ubuntu-24.04
```

Verify:

```bash
systemctl is-system-running
```

Expected output (either is fine):

```text
running
```

or `degraded` (some units fail in WSL; that is acceptable for the labs).

WSL2 limitations you will hit and how to work around them:

- `perf` for the WSL kernel is not in Ubuntu repos. Use `bpftrace` instead, or run perf labs in Multipass.
- Kernel parameters set via `sysctl` persist only for the session unless placed in `/etc/sysctl.d/` and systemd is enabled.
- `/mnt/c` is a 9p/DrvFs mount; run labs in `~` (ext4) for realistic `inotify`, permissions and performance behaviour.

## 3. Option B: Multipass VM (Windows, macOS, Linux)

Install from https://multipass.run or with a package manager:

```bash
# macOS
brew install --cask multipass
# Ubuntu host
sudo snap install multipass
```

Windows: download the installer, or `winget install Canonical.Multipass`.

Create a VM with cloud-init so the toolbox is pre-installed and your SSH key is injected. Save this as `lab-cloud-init.yaml`:

```yaml
#cloud-config
hostname: lab01
users:
  - name: lab
    groups: [sudo, adm]
    shell: /bin/bash
    sudo: ALL=(ALL) NOPASSWD:ALL
    ssh_authorized_keys:
      - <PASTE_YOUR_PUBLIC_KEY_HERE>   # contents of ~/.ssh/id_ed25519.pub
package_update: true
package_upgrade: true
packages:
  - build-essential
  - strace
  - ltrace
  - sysstat
  - htop
  - iotop
  - iproute2
  - net-tools
  - tcpdump
  - jq
  - tree
  - lsof
  - psmisc
  - procps
  - bpftrace
  - bpfcc-tools
  - linux-tools-common
  - util-linux
  - cgroup-tools
  - apparmor-utils
  - fail2ban
  - unattended-upgrades
write_files:
  - path: /etc/sysctl.d/90-lab.conf
    content: |
      kernel.perf_event_paranoid = -1
      kernel.kptr_restrict = 0
runcmd:
  - [ sysctl, --system ]
  - [ systemctl, enable, --now, sysstat ]
```

Note: `perf_event_paranoid = -1` and `kptr_restrict = 0` are **insecure shortcuts** acceptable only on a throwaway lab VM so that unprivileged `perf` works. Never apply them to a production node.

Launch:

```bash
multipass launch 24.04 --name lab01 --cpus 2 --memory 4G --disk 30G \
  --cloud-init lab-cloud-init.yaml
multipass info lab01
```

Expected output (abridged):

```text
Name:           lab01
State:          Running
IPv4:           10.x.x.x
Release:        Ubuntu 24.04.x LTS
Image hash:     ...
CPU(s):         2
Load:           0.08 0.11 0.05
Disk usage:     2.1GiB out of 29.0GiB
Memory usage:   310.0MiB out of 3.8GiB
```

Enter the VM:

```bash
multipass shell lab01
# or, via SSH using the key you injected
ssh lab@$(multipass info lab01 --format json | jq -r '.info.lab01.ipv4[0]')
```

Snapshots give you the "break it and rebuild" workflow:

```bash
multipass stop lab01
multipass snapshot lab01 --name clean
multipass start lab01
# ... break things ...
multipass stop lab01 && multipass restore lab01.clean && multipass start lab01
```

## 4. Option C: Lima (macOS)

```bash
brew install lima
limactl create --name=lab01 template://ubuntu-lts --cpus=2 --memory=4 --disk=30
limactl start lab01
limactl shell lab01
```

Inside, run the apt commands from section 6.

## 5. Option D: Cloud VM for the SSH Hardening Lab

Use the AWS CLI only after cluster 01; for now the console is fine. Choose Ubuntu 24.04, `t3.small`, a security group allowing TCP 22 from **your IP only** (`<YOUR_PUBLIC_IP>/32`), and your SSH key pair. Tag it `lab=foundations` and set a calendar reminder to terminate it. Never leave 0.0.0.0/0 on port 22.

## 6. Install the Toolbox

### Ubuntu / Debian (apt)

```bash
sudo apt-get update
sudo apt-get install -y \
  build-essential strace ltrace sysstat htop iotop atop \
  iproute2 net-tools tcpdump nmap-ncat mtr-tiny dnsutils \
  jq tree lsof psmisc procps util-linux \
  bpftrace bpfcc-tools linux-tools-common "linux-tools-$(uname -r)" \
  cgroup-tools apparmor-utils auditd \
  fail2ban unattended-upgrades
```

`linux-tools-$(uname -r)` provides `perf` matched to your kernel. On WSL2 it will fail; skip it there.

Install `yq` (mikefarah, Go binary; the apt `yq` is a different Python tool):

```bash
YQ_VERSION=v4.44.3
sudo curl -fsSL -o /usr/local/bin/yq \
  "https://github.com/mikefarah/yq/releases/download/${YQ_VERSION}/yq_linux_amd64"
sudo chmod 0755 /usr/local/bin/yq
yq --version
```

On ARM64 (Apple Silicon Lima/Multipass) use `yq_linux_arm64`. Downloading a binary with `curl` and marking it executable is acceptable here because you pinned a version; in CI prefer verifying the published checksum (`checksums` file in the release) before installing.

### Fedora / RHEL 9 / Amazon Linux 2023 (dnf)

```bash
sudo dnf install -y \
  @development-tools strace ltrace sysstat htop iotop \
  iproute net-tools tcpdump nmap-ncat mtr bind-utils \
  jq tree lsof psmisc procps-ng util-linux \
  bpftrace bcc-tools perf \
  libcgroup-tools policycoreutils-python-utils setools-console audit \
  fail2ban dnf-automatic
```

### Alpine (apk)

Alpine is what many container images use, so knowing its deltas matters:

```bash
sudo apk add --no-cache \
  bash coreutils util-linux procps strace ltrace \
  iproute2 tcpdump bind-tools mtr jq tree lsof \
  bpftrace bcc-tools perf \
  shadow sudo openssh
```

Note that Alpine uses musl libc, BusyBox utilities (`ps`, `grep`, `sed` have fewer flags until you install the GNU versions) and OpenRC rather than systemd. Install `procps` and `coreutils` explicitly to get the GNU behaviour the rest of this folder assumes.

### macOS host (for client tools only)

```bash
brew install coreutils gnu-sed gawk jq yq mtr nmap wget openssh
```

macOS ships BSD `sed`/`awk`; the Homebrew GNU versions are prefixed `gsed`, `gawk`. Prefer running the labs inside the VM.

## 7. First-Run Configuration

### Create a non-root admin user (if the image gave you root only)

```bash
sudo adduser --gecos "" lab
sudo usermod --append --groups sudo,adm lab
sudo mkdir -p /home/lab/.ssh && sudo chmod 0700 /home/lab/.ssh
sudo cp ~/.ssh/authorized_keys /home/lab/.ssh/ && sudo chown -R lab:lab /home/lab/.ssh
```

### Generate an SSH key on your workstation (not on the VM)

```bash
ssh-keygen -t ed25519 -a 64 -C "lab-workstation" -f ~/.ssh/id_ed25519_lab
```

`-a 64` sets the KDF rounds so a stolen private key file is slow to brute-force. Use a passphrase. Add to `~/.ssh/config`:

```text
Host lab01
    HostName <VM_IP>
    User lab
    IdentityFile ~/.ssh/id_ed25519_lab
    IdentitiesOnly yes
    AddKeysToAgent yes
```

### Shell quality of life

```bash
cat >> ~/.bashrc <<'EOF'
export EDITOR=vim
export HISTSIZE=50000 HISTFILESIZE=100000 HISTCONTROL=ignoredups:erasedups
export HISTTIMEFORMAT='%F %T '
shopt -s histappend checkwinsize
alias ll='ls -alF --color=auto'
alias ports='ss -tulpn'
EOF
source ~/.bashrc
```

### Enable persistent journal and sysstat collection

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
sudo systemctl enable --now sysstat
```

Persistent journal lets `journalctl --boot -1` show the previous boot, which matters when debugging a crash. `sysstat` makes `sar` history available (`sar -u -f /var/log/sysstat/saXX`).

## 8. Verification

Run every check; each should succeed before you move on.

```bash
# Kernel and distro
uname -r; cat /etc/os-release | head -2
```

Expected output:

```text
6.8.0-xx-generic
PRETTY_NAME="Ubuntu 24.04.x LTS"
NAME="Ubuntu"
```

```bash
# cgroups v2 is mounted (required for the cgroup labs)
stat -fc %T /sys/fs/cgroup
```

Expected output:

```text
cgroup2fs
```

If you see `tmpfs` you are on the hybrid v1 layout; add `systemd.unified_cgroup_hierarchy=1` to the kernel command line (`/etc/default/grub`, then `update-grub` and reboot).

```bash
# Namespaces usable unprivileged
unshare --user --map-root-user --pid --fork --mount-proc sh -c 'id; ps -o pid,comm'
```

Expected output:

```text
uid=0(root) gid=0(root) groups=0(root),65534(nogroup)
    PID COMMAND
      1 sh
      2 ps
```

```bash
# strace, perf, bpftrace
strace -c -e trace=openat ls / > /dev/null
sudo perf stat -e task-clock true 2>&1 | grep task-clock      # skip on WSL2
sudo bpftrace -e 'BEGIN { printf("bpf ok\n"); exit(); }'
```

Expected output of the last command:

```text
Attaching 1 probe...
bpf ok
```

```bash
# JSON / YAML tooling
echo '{"a":{"b":[1,2,3]}}' | jq '.a.b | add'
printf 'a:\n  b: [1, 2, 3]\n' | yq '.a.b | length'
```

Expected output:

```text
6
3
```

```bash
# systemd and journald
systemctl --version | head -1
journalctl --disk-usage
```

Expected output:

```text
systemd 255 (255.4-1ubuntu8)
Archived and active journals take up 16.0M in the file system.
```

## 9. Reset Procedure

| Environment | Reset command |
|---|---|
| WSL2 | `wsl --unregister Ubuntu-24.04` then `wsl --install --distribution Ubuntu-24.04` |
| Multipass | `multipass restore lab01.clean` or `multipass delete --purge lab01 && multipass launch ...` |
| Lima | `limactl delete --force lab01 && limactl create ...` |
| Cloud VM | Terminate the instance; recreate from the same AMI |

Keep `lab-cloud-init.yaml` in your capstone repo under `tools/lab/`; it is the first piece of infrastructure-as-code you own, and in Phase 2 you will pass the same cloud-init to an EC2 launch template.

## Next

Continue with [02-core-concepts.md](02-core-concepts.md).
