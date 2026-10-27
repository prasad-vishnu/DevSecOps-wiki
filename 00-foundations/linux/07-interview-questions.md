# Linux: Interview Questions

| | |
|---|---|
| **Cluster** | 00 Foundations |
| **Role** | Prerequisite |
| **Prerequisites** | The rest of this folder |
| **Target version** | Linux kernel 6.x, systemd 255+, cgroups v2 |
| **Capstone phase** | Phase 1 (Months 1-4) |

> Thirty questions from junior to principal, each with a model answer. Interviewers for platform, SRE and DevSecOps roles ask these to find out whether you understand what the container runtime and the kubelet are doing on your behalf. Answer out loud, then compare.

## Junior

### Q1. What happens, step by step, when you type `ls -l /tmp` in a shell and press Enter? (Junior)

**Answer.** The shell parses the line, expands globs and variables, finds `ls` in `PATH` (`/usr/bin/ls`), then `fork()`s (actually `clone()`) a child. In the child it sets up redirections (none here) and calls `execve("/usr/bin/ls", ["ls","-l","/tmp"], envp)`, which replaces the child's memory with the `ls` program while keeping the PID and open file descriptors (0, 1, 2 point at the terminal). `ls` calls `openat`/`getdents64` on `/tmp`, `statx` on each entry, writes to fd 1, and `exit_group(0)`. The parent shell was blocked in `waitpid()`; it collects the exit status (reaping the child so it is not a zombie), stores it in `$?`, and prints the next prompt.

### Q2. Explain the permission string `-rwsr-xr-x` on `/usr/bin/passwd`. Why is it dangerous and why is it necessary? (Junior)

**Answer.** Regular file, owner `rwx` with the setuid bit (`s` in the owner execute position), group `r-x`, other `r-x`. When any user runs it, the process's effective uid becomes the file owner (root), so `passwd` can write `/etc/shadow`, which is root-only. It is dangerous because any bug in a setuid program is a local privilege escalation; that is why hardened images ship with no setuid binaries, mount filesystems `nosuid`, and use capabilities or `no_new_privs` instead.

### Q3. What is the difference between a hard link and a symbolic link? (Junior)

**Answer.** A hard link is a second directory entry pointing at the same inode; both names are equal, the file persists until the link count hits zero and no process holds it open, and hard links cannot cross filesystems or point at directories. A symlink is a separate small file whose content is a path; it can cross filesystems and dangle if the target is removed. `ln -sfn` atomically repoints a symlink, which is how release directories (`/opt/app -> /opt/app-1.4.3`) are switched.

### Q4. A colleague says "the server has only 200 MB free memory, we need a bigger instance." How do you check whether they are right? (Junior)

**Answer.** `free -m` and look at `available`, not `free`. Linux uses idle RAM for page cache (`buff/cache`), which is reclaimable instantly. If `available` is large, there is no problem. Confirm with `vmstat 1` (`si`/`so` should be zero), `/proc/pressure/memory` (`some avg10` near zero) and `dmesg` for OOM kills. Memory is a problem when `available` is low, swap is active, PSI is non-zero, or `memory.events` shows `oom_kill`.

### Q5. What do exit codes 137, 143 and 139 mean? (Junior)

**Answer.** When a process dies from a signal, the shell reports `128 + signal number`. 137 = 128 + 9 (SIGKILL): the OOM killer, `kill -9`, or the kubelet after the grace period. 143 = 128 + 15 (SIGTERM): a polite stop that the process did not handle (or handled and exited with that code). 139 = 128 + 11 (SIGSEGV): a crash from invalid memory access.

### Q6. What is the difference between `apt-get update` and `apt-get upgrade`? Where do `--no-install-recommends` and `rm -rf /var/lib/apt/lists/*` fit in a Dockerfile? (Junior)

**Answer.** `update` refreshes the package index from the configured repositories; `upgrade` installs newer versions of installed packages. In a Dockerfile you run `apt-get update && apt-get install -y --no-install-recommends <pkgs> && rm -rf /var/lib/apt/lists/*` in one `RUN` so the index, install and cleanup are in one layer; `--no-install-recommends` avoids pulling optional dependencies that inflate the image and CVE count, and deleting the lists removes about 40 MB of index data that is useless at runtime. Never delete `/var/lib/dpkg`, which scanners need.

### Q7. How do you find which process is listening on port 8080? (Junior)

**Answer.** `sudo ss -tlnp 'sport = :8080'` (or `ss -tlnp | grep 8080`); the `users:(("python3",pid=6120,fd=3))` column names the process. `lsof -i :8080` is the alternative. If nothing shows but a container is listening, the socket is in another network namespace: `sudo nsenter -t <container-pid> -n ss -tlnp`.

### Q8. What is `/proc` and name three things you can learn from `/proc/<pid>/`. (Junior)

**Answer.** `/proc` is procfs, a virtual filesystem exposing kernel and process state as files, generated on read. From `/proc/<pid>/` you can read `status` (state, uid, threads, capability sets, seccomp mode), `fd/` (every open file, socket, pipe), `cgroup` (the resource group), `ns/` (namespace membership), `cmdline`, `environ`, `exe`, `cwd`, `limits`, `stack`, `oom_score`, and `maps`/`smaps_rollup` (memory layout). Almost every monitoring tool reads these files.

## Mid-level

### Q9. Explain how `docker stop` works at the kernel level and why some containers take exactly 10 seconds to stop. (Mid-level)

**Answer.** `docker stop` sends `SIGTERM` (or the image's `STOPSIGNAL`) to PID 1 of the container, waits the timeout (default 10 s), then sends `SIGKILL`. If PID 1 is the application and handles SIGTERM, it exits quickly. If PID 1 is a shell script that launched the app without `exec`, the shell receives SIGTERM; non-interactive shells running a foreground child do not forward it, so nothing happens until the SIGKILL at 10 s, which kills the shell and, because the PID namespace's init died, everything inside. Kubernetes has the same pattern with `terminationGracePeriodSeconds` (default 30 s). Fixes: `exec "$@"` in the entrypoint, or `tini`/`--init` as PID 1, and make the application handle SIGTERM.

### Q10. What is a zombie process, why does it matter in containers, and how do you prevent it? (Mid-level)

**Answer.** A zombie is a process that has exited but whose parent has not called `wait()` to collect its exit status; it holds a PID and a process table entry but no memory. Normally PID 1 (systemd) reaps orphaned children. In a container, PID 1 is whatever the entrypoint is; if the app spawns children (shell-outs, worker processes) and does not reap orphans, zombies accumulate until `pids.max` is hit and `fork` fails with `EAGAIN`. Prevention: a minimal init as PID 1 (`tini`, `dumb-init`, Docker `--init`), or in Kubernetes `shareProcessNamespace: true` so the pause container reaps, or make the app call `waitpid` / set `SIGCHLD` to `SIG_IGN`.

### Q11. Describe the seven or eight Linux namespaces and which ones a Kubernetes Pod shares between its containers. (Mid-level)

**Answer.** Mount (filesystem view), PID (process IDs), network (interfaces, routes, ports), UTS (hostname), IPC (SysV/POSIX IPC), user (uid/gid mapping), cgroup (hierarchy root view), and time (clock offsets). In a Pod, all containers share the **network**, **IPC** and **UTS** namespaces (that is why they talk over `localhost` and share a hostname), created by the pause/sandbox container. Each container has its own **mount** namespace (its own image rootfs) and, by default, its own **PID** namespace; `shareProcessNamespace: true` shares PID. User namespaces are opt-in with `hostUsers: false`.

### Q12. How does a Kubernetes CPU limit of `500m` become a kernel setting, and how do you detect that it is hurting a service? (Mid-level)

**Answer.** The kubelet (via the container runtime) writes `cpu.max = "50000 100000"` into the container's cgroup v2 directory: a quota of 50,000 µs per 100,000 µs period, enforced by CFS bandwidth control. When the container's threads consume the quota before the period ends, they are descheduled until the next period, which shows up as `nr_throttled` and `throttled_usec` rising in `cpu.stat`, and as `cpu.pressure` PSI. Symptoms are tail latency spikes with CPU usage well below the node's capacity. Detection: `container_cpu_cfs_throttled_periods_total / container_cpu_cfs_periods_total` in Prometheus above a few percent. Fixes: raise or remove the limit and rely on requests (`cpu.weight`), or reduce thread count so the app does not burst across all cores.

### Q13. A process shows `D` state and `kill -9` does nothing. What is happening and what can you do? (Mid-level)

**Answer.** `D` is uninterruptible sleep: the task is inside the kernel waiting for I/O (disk, NFS, a stuck driver) and signals are not delivered until it returns. `cat /proc/<pid>/stack` or `wchan` shows the kernel function it waits in; `dmesg` may show `task blocked for more than 120 seconds`. The cause is almost always storage: check `iostat -xz`, NFS server health, cloud volume status. Options: fix the I/O path so the call returns, use `soft` mounts for NFS, or reboot/replace the node. There is no user-space way to kill a `D` process.

### Q14. What is the difference between `memory.max` and `memory.high`, and which one does Kubernetes set? (Mid-level)

**Answer.** `memory.max` is a hard limit: when the cgroup's usage (anonymous memory plus page cache minus what reclaim can free) hits it, the OOM killer kills a process in the group (all of them if `memory.oom.group=1`). `memory.high` is a throttling threshold: above it the kernel aggressively reclaims and slows allocations in that cgroup but does not kill. Kubernetes sets `memory.max` from `limits.memory`; with the `MemoryQoS` feature gate it also sets `memory.min` from requests and `memory.high` from a formula. systemd lets you set both (`MemoryHigh=` at about 85% of `MemoryMax=`) to get a soft landing before the kill.

### Q15. How do you make a non-root process bind port 443 without giving it root, and what single kernel bit stops a setuid trick from working? (Mid-level)

**Answer.** Grant `CAP_NET_BIND_SERVICE`: on a binary with `setcap cap_net_bind_service=+ep /path`, in systemd with `AmbientCapabilities=CAP_NET_BIND_SERVICE`, in Docker with `--cap-add NET_BIND_SERVICE`, or in Kubernetes `securityContext.capabilities.add: ["NET_BIND_SERVICE"]`. Or avoid the problem by listening on 8443 and letting the Service or load balancer map 443. The `no_new_privs` bit (`NoNewPrivileges=yes`, `allowPrivilegeEscalation: false`) makes `execve` ignore setuid bits and file capabilities, so a process cannot gain privileges it does not already have.

### Q16. Explain what `strace -c -p <pid>` tells you and why you would not run `strace` on a production process for long. (Mid-level)

**Answer.** It attaches with `ptrace`, intercepts every system call, and at exit prints a table of syscall counts, errors and time per call, which identifies syscall storms (`fsync`, `stat`, `futex` contention) and failing calls (`ENOENT` on config lookups, `EACCES`). `ptrace` stops the process at every syscall entry and exit, so a syscall-heavy service can slow 10-100x and miss its deadlines. For production use `perf trace` (lower overhead), or eBPF tools (`bpftrace`, bcc `syscount`, `opensnoop`) that aggregate in the kernel and cost a few percent.

### Q17. Compare systemd `Restart=on-failure` with a Kubernetes `restartPolicy`. What protects against crash loops in each? (Mid-level)

**Answer.** Both restart a failed process; systemd restarts the unit's main process after `RestartSec`, Kubernetes restarts the container (same Pod, same node). systemd rate-limits with `StartLimitIntervalSec`/`StartLimitBurst` and then marks the unit failed until `reset-failed`. Kubernetes applies exponential backoff (10 s, 20 s, 40 s... capped at 5 minutes) and reports `CrashLoopBackOff` but never gives up. In both cases the logs (`journalctl -u` / `kubectl logs --previous`) and exit code tell you why.

### Q18. What is `vm.max_map_count` and why do Elasticsearch and some JVMs demand it be raised? (Mid-level)

**Answer.** It is the maximum number of memory-mapped regions a process can have (default 65,530). Elasticsearch/Lucene memory-maps many index segment files; large JVMs create many mappings for thread stacks, GC regions and JIT code caches. Exceeding the limit produces `ENOMEM` on `mmap` even with free memory. Raising it to 262,144 is a node-level sysctl (`vm.*` is not namespaced), so it goes in the node image or an init container with privileges, which is why it appears in Kubernetes operator prerequisites.

### Q19. How does `journalctl` know which unit a log line belongs to, and how is that different from a log file? (Mid-level)

**Answer.** systemd connects the service's stdout/stderr to a stream socket owned by journald and records the sending process's credentials (`_PID`, `_UID`, `_SYSTEMD_UNIT`, `_SYSTEMD_CGROUP`) from the socket, so the metadata is trustworthy and queryable (`journalctl -u api`, `_COMM=sshd`, `-p err`, `-b -1`). A log file has only what the application wrote, no provenance, no boot indexing, and needs external rotation. The journal is binary and indexed; `-o json` makes it machine-readable for shipping to Loki.

## Senior

### Q20. Walk through creating a container with no runtime: which syscalls and files are involved, in order? (Senior)

**Answer.** (1) `clone()` with `CLONE_NEWNS|NEWPID|NEWNET|NEWUTS|NEWIPC` (optionally `NEWUSER`, `NEWCGROUP`), so the child is PID 1 in a fresh set of namespaces. (2) Before or right after, write the child's PID into a new cgroup directory's `cgroup.procs` with `cpu.max`, `memory.max`, `pids.max` set. (3) Mount an overlayfs of image layers, `mount --make-rprivate /`, bind-mount the rootfs onto itself, `pivot_root` into it, mount `/proc`, `/sys` (ro), a tmpfs `/dev` with the minimal device nodes, then lazily unmount and remove the old root. (4) Create a veth pair, move one end into the child's net namespace, assign addresses, add a default route; on the host add it to a bridge and a NAT rule. (5) Set hostname. (6) Apply a seccomp filter via `prctl`/`seccomp()`, set `PR_SET_NO_NEW_PRIVS`, drop bounding and ambient capabilities with `capset`/`prctl(PR_CAPBSET_DROP)`, apply the LSM label. (7) `setgid`/`setuid` to the non-root user, set `umask`, `chdir` to the workdir, set the environment. (8) `execve` the entrypoint. runc performs exactly this from the OCI `config.json`; `docker run` flags and Pod `securityContext` fields are parameters to these steps.

### Q21. A Java service in a Pod with `limits.memory: 2Gi` gets OOMKilled although the heap is capped at 1 GB. Explain the likely causes and the fix. (Senior)

**Answer.** The cgroup limit counts all memory the container touches, not only the Java heap: metaspace, thread stacks (each platform thread 512 KB-1 MB), JIT code cache, direct `ByteBuffer`s (Netty, gRPC), GC structures, glibc malloc arenas (fragmentation with many threads), plus page cache for files the JVM reads, and any sidecar process in the same container. Diagnose with `jcmd <pid> VM.native_memory summary` (enable `-XX:NativeMemoryTracking=summary`), `memory.stat` (`anon` vs `file`), `pmap -x`. Fixes: size the heap as a fraction (`-XX:MaxRAMPercentage=60-75`) leaving headroom, cap direct memory (`-XX:MaxDirectMemorySize`), limit thread count or use virtual threads, set `MALLOC_ARENA_MAX=2`, and alert on `container_memory_working_set_bytes` approaching the limit before the kill.

### Q22. How does the OOM killer choose a victim, and how does Kubernetes influence it? (Senior)

**Answer.** When a cgroup (or the system) cannot satisfy an allocation after reclaim, the kernel computes `oom_score` for each candidate: roughly the proportion of memory used (RSS + swap + page tables) adjusted by `oom_score_adj` (-1000 never kill, +1000 always prefer). The highest score dies with SIGKILL; with `memory.oom.group=1` the whole cgroup dies. Kubernetes sets `oom_score_adj` per QoS class: Guaranteed Pods -997, BestEffort 1000, Burstable `1000 - (1000 * requests.memory / node allocatable)` so Pods using more than they requested die first under node pressure. Inside a container limit, the kill is confined to that cgroup. The kubelet separately evicts Pods (gracefully) when node `memory.available` falls below thresholds, which is preferable to the kernel OOM killer.

### Q23. Design the sysctl and limits baseline for an EKS node running a high-connection FastAPI service behind an NLB. Justify each. (Senior)

**Answer.** `net.ipv4.ip_forward=1` and `net.bridge.bridge-nf-call-iptables=1` for Pod networking and kube-proxy. `net.core.somaxconn=32768` and `net.ipv4.tcp_max_syn_backlog=16384` so bursts do not overflow accept queues (verify with `nstat TcpExtListenOverflows`). `net.ipv4.ip_local_port_range=1024 65535` and `tcp_tw_reuse=1` because kube-proxy/NAT and the app's outbound calls churn ephemeral ports. `net.netfilter.nf_conntrack_max=1048576` since every NATed Pod connection is a conntrack entry; "table full" drops are silent. `tcp_keepalive_time=600` below the NLB's 350 s idle timeout? No: the NLB resets idle flows at 350 s, so keepalives should be shorter than that on the application side (uvicorn `--timeout-keep-alive`), and `tcp_keepalive_time` around 300 to detect dead peers. `fs.inotify.max_user_watches=1048576`, `max_user_instances=8192` for kubelet, log agents and Argo. `vm.max_map_count=262144`, `vm.swappiness=0`, swap off. `kernel.panic=10`, `panic_on_oops=1` so a wedged node self-replaces. `LimitNOFILE=1048576` for containerd (default in its unit) and per container via the runtime default. Hardening: `kptr_restrict=2`, `dmesg_restrict=1`, `unprivileged_bpf_disabled=1`, `ptrace_scope=1`. All of it in the launch template user data or baked into the AMI, checked into Git, verified by a node conformance test.

### Q24. Explain cgroups v1 versus v2 and why Kubernetes moved to v2. (Senior)

**Answer.** v1 had a separate hierarchy per controller (`/sys/fs/cgroup/memory`, `/cpu`...), so a process could be in different groups for different resources, delegation to unprivileged users was unsafe, and accounting was inconsistent (memory did not account kernel memory uniformly). v2 has one unified tree with controllers enabled per subtree, a "no internal processes" rule, consistent interface files (`cpu.max`, `memory.max`), pressure stall information per cgroup, `memory.oom.group`, safe delegation (rootless containers), and better memory accounting including swap control (`memory.swap.max`). Kubernetes needed v2 for rootless/user-namespaced Pods, PSI-based metrics, MemoryQoS (`memory.high`/`memory.min`) and swap support; v1 is deprecated from Kubernetes 1.31 and absent in newer node images (AL2023, Bottlerocket, Ubuntu 22.04+).

### Q25. How would you prove that a given file on disk is the one a running process actually loaded, and that it has not been modified since? (Senior)

**Answer.** `readlink /proc/<pid>/exe` gives the executable path (with `(deleted)` if replaced since start). `/proc/<pid>/maps` lists mapped files with device and inode numbers; compare with `stat -c '%d %i' /path`. For a replaced file, the inode differs. Hash the live mapped image via `/proc/<pid>/map_files/` (root) or `sha256sum /proc/<pid>/exe`, compare to the package (`dpkg --verify`, `rpm -V`) or the image's SBOM/signature. For config files, `lsof -p` shows open ones; auditd `-w /path -p wa` records who changed what. At fleet scale, file integrity monitoring (Wazuh/AIDE) and `fs-verity`/dm-verity (Bottlerocket's read-only root) answer this continuously.

### Q26. What is eBPF, why is it safe to load into the kernel, and how do Falco and Cilium use it differently? (Senior)

**Answer.** eBPF is an in-kernel virtual machine: user space loads bytecode via the `bpf()` syscall, the verifier statically proves the program terminates (bounded loops), accesses only valid memory, and uses only allowed helpers; a JIT compiles it to native code; it attaches to hooks (kprobes, tracepoints, uprobes, XDP, tc, cgroup socket hooks, LSM) and communicates through maps and ring buffers. Falco attaches to syscall tracepoints and sched events to stream a security-relevant event feed (process exec, file open, network connect) to a user-space rules engine, with read-only observation. Cilium attaches at tc/XDP and cgroup socket hooks to *act* on packets: load balancing (replacing kube-proxy's iptables with hash maps), NetworkPolicy enforcement with identity labels, NAT, and L7 visibility via Hubble; it changes forwarding decisions rather than only observing. Requirements: kernel 5.8+ (`CAP_BPF`), BTF for CO-RE portability.

### Q27. A node reports `DiskPressure` but `df -h` shows 40% free on `/`. What else could be wrong? (Senior)

**Answer.** The kubelet evaluates `nodefs` (usually `/var/lib/kubelet`) and `imagefs` (`/var/lib/containerd`) separately, and both space and **inodes** (`nodefs.inodesFree`, `imagefs.inodesFree`). Check `df -h /var/lib/containerd /var/lib/kubelet` if they are separate volumes, then `df -i` on each; container image layers consume huge inode counts. Also check eviction thresholds (`--eviction-hard` defaults: `nodefs.available<10%`, `imagefs.available<15%`, `nodefs.inodesFree<5%`) versus what is reported, deleted-but-open files (`lsof +L1`), and ephemeral-storage overuse by Pods writing to `emptyDir` or container writable layers (`du -sh /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/*`). Fix the inode exhaustion or garbage-collect images (`crictl rmi --prune`).

## Principal

### Q28. You own base images and node images for 300 services. Design the lifecycle so that a kernel or glibc CVE is remediated fleet-wide within 72 hours without incidents. (Principal)

**Answer.** Separate the two supply chains but run them the same way. **Container bases:** a small set of golden bases (distroless/Wolfi for runtime, Debian-slim for build), rebuilt daily in CI from pinned upstream digests, scanned (Trivy/Grype against SBOM from Syft), signed (Cosign keyless), published with floating tags *and* digests; service pipelines pin by digest and a bot (Renovate/Dependabot) opens PRs bumping the digest; Kyverno admission requires signatures and rejects bases older than N days; Dependency-Track flags affected services from SBOMs when the CVE drops so you know blast radius in minutes. **Node images:** Bottlerocket or weekly-built AMIs via Image Builder/Packer, CIS-hardened, with the sysctl/unit baseline from Git; roll via launch template version with Karpenter drift or ASG instance refresh at 10-20% at a time, PDBs respected, canary node pool first, automatic rollback on `KubeletDown`/`NodeNotReady` alerts. **Controls:** SLO for patch latency per severity, dashboards fed by Dependency-Track and node image versions, exceptions as ADRs with expiry. **Safety:** staging cluster soak for 24 h, progressive delivery for service rebuilds (Argo Rollouts analysis), and the GitOps repo as the single source so the 72-hour clock is an audit query, not a spreadsheet.

### Q29. The on-call engineer sees p99 latency doubling every day at 14:00 for 20 minutes on one node pool; CPU and memory look fine on dashboards. Describe how you would find the root cause with Linux tools, and the systemic fix. (Principal)

**Answer.** Dashboards average over minutes and Pods; the signal is in saturation metrics and short windows. First confirm the resource: `cpu.pressure`/`io.pressure`/`memory.pressure` on the node and per cgroup during the window (PSI shows stalls even at 40% utilisation), `cpu.stat nr_throttled` for the affected Pods (CFS throttling with bursty threads), `iostat -xz` for `await` spikes (EBS burst credits, a daily batch job, log rotation `gzip`), `nstat` for `ListenOverflows`, `ss -ti` for retransmits, `runqlat` for scheduler latency, `biolatency` for disk. Correlate with `journalctl --since 13:55 --until 14:20` and `systemctl list-timers` plus Kubernetes CronJobs for anything scheduled at 14:00 (a backup, a Trivy scan DaemonSet, a log shipper flush, `apt-daily`). Suspects in order of frequency: a CronJob saturating disk or CPU on the same node without limits, a neighbour Pod with no requests (BestEffort) stealing CPU, THP compaction, conntrack table churn, or a cloud burstable volume. Capture with `perf record` during the window if needed. Systemic fix: enforce requests/limits by LimitRange and Kyverno so noisy neighbours are impossible, isolate batch workloads to a separate node pool with taints, move node-level periodic work off business hours or rate-limit it (`IOWeight=`, `CPUWeight=` in systemd), alert on PSI and throttling ratio rather than utilisation, and write the runbook so the next engineer runs the same ten commands.

### Q30. Argue for and against SSH access to production nodes in a regulated environment, and describe the design you would adopt. (Principal)

**Answer.** For: unmatched debugging depth (`perf`, `strace`, kernel logs) when telemetry is insufficient, and familiarity. Against: a standing network path and credential surface, weak attribution with shared keys, no session recording by default, drift from manual fixes, and auditors asking who did what. Design: no inbound 22 by default; nodes are immutable (Bottlerocket) so there is little to fix in place. Debug via `kubectl debug node/` with RBAC, OIDC identity and API audit logs for Kubernetes nodes, and cloud session brokers (SSM Session Manager, with sessions logged to S3 and KMS-encrypted) for other instances; both require MFA and are scoped by IAM policies with time-bound grants (just-in-time access through an approval workflow). Keep a break-glass SSH path: an SSH CA (Vault SSH engine) issuing 1-hour user certificates bound to a ticket ID, `AuthorizedPrincipalsFile` on hosts, `LogLevel VERBOSE`, session recording via `script`/Teleport or `auditd execve` rules shipped to the SIEM, and alerts on any use. Document it as an ADR with the compensating controls, and test the break-glass path quarterly so it works when it is actually needed.

---

Scoring yourself: Q1-Q8 should be instant; Q9-Q19 you should answer with the exact file or command names; Q20-Q27 need a structured explanation with trade-offs; Q28-Q30 are design conversations with no single right answer, judged on the breadth of concerns you raise and the clarity of the plan.
