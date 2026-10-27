# Linux: Hands-On Labs

| | |
|---|---|
| **Cluster** | 00 Foundations |
| **Role** | Prerequisite |
| **Prerequisites** | Lab VM from [01-installation-and-setup.md](01-installation-and-setup.md); concepts from [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | Ubuntu 24.04, kernel 6.8+, systemd 255, cgroups v2 |
| **Capstone phase** | Phase 1 (Months 1-4) |

> Eight labs that go from reading `/proc` to building a working container with `unshare`, `pivot_root`, cgroups and capabilities, with no Docker installed. Run them on the disposable Multipass VM (several labs intentionally exhaust resources). Take a snapshot first: `multipass snapshot lab01 --name pre-labs`. Each lab lists the exact commands, expected output, a verification step and cleanup.

| Lab | Topic | Level | Time |
|---|---|---|---|
| 1 | Processes, `/proc`, file descriptors and signals | Beginner | 30 min |
| 2 | Deleted-but-open files and inode exhaustion | Beginner | 25 min |
| 3 | A hardened systemd service with resource limits and journald | Intermediate | 45 min |
| 4 | Users, permissions, setuid, capabilities | Intermediate | 40 min |
| 5 | cgroups v2 by hand: CPU throttling and OOM kill | Intermediate | 40 min |
| 6 | Namespaces by hand: pid, mount, uts, net | Intermediate | 45 min |
| 7 | Performance investigation: USE method, strace, perf, bpftrace | Advanced | 60 min |
| 8 | Build a container from scratch (rootfs + namespaces + cgroup + caps) | Advanced | 75 min |

---

## Lab 1: Processes, `/proc`, File Descriptors and Signals

**Objective.** See a process's full state in `/proc`, observe fork/exec, redirect file descriptors, and verify the `128 + N` exit-code rule.

**Prerequisites.** Lab VM shell.

**Time.** 30 minutes.

### Steps

1. Start a background process and inspect it.

   ```bash
   sleep 600 &
   SP=$!
   echo "sleep is PID $SP, parent is $$"
   cat /proc/$SP/status | grep -E '^(Name|State|Pid|PPid|Uid|Threads|VmRSS|CapEff|NoNewPrivs|Seccomp)'
   ```

   Expected output:

   ```text
   Name:   sleep
   State:  S (sleeping)
   Pid:    4821
   PPid:   4790
   Uid:    1000    1000    1000    1000
   VmRSS:     1792 kB
   Threads:        1
   CapEff: 0000000000000000
   NoNewPrivs:     0
   Seccomp:        0
   ```

2. Look at its file descriptors and namespaces.

   ```bash
   ls -l /proc/$SP/fd
   ls -l /proc/$SP/ns | awk '{print $9, $11}'
   readlink /proc/$SP/exe
   cat /proc/$SP/cgroup
   ```

   Expected output (fds):

   ```text
   lrwx------ 1 lab lab 64 ... 0 -> /dev/pts/0
   lrwx------ 1 lab lab 64 ... 1 -> /dev/pts/0
   lrwx------ 1 lab lab 64 ... 2 -> /dev/pts/0
   ```

   The namespaces have the same inode numbers as your shell's (`ls -l /proc/$$/ns`); the cgroup line shows `0::/user.slice/user-1000.slice/session-N.scope`.

3. Observe fork and exec with `strace`.

   ```bash
   strace -f -e trace=clone,clone3,execve,wait4,exit_group bash -c 'ls / > /dev/null' 2>&1 | head -20
   ```

   Expected output (abridged):

   ```text
   execve("/usr/bin/bash", ["bash", "-c", "ls / > /dev/null"], ...) = 0
   clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD, ...) = 4902
   [pid  4902] execve("/usr/bin/ls", ["ls", "/"], ...) = 0
   [pid  4902] exit_group(0)               = ?
   wait4(-1, [{WIFEXITED(s) && WEXITSTATUS(s) == 0}], 0, NULL) = 4902
   exit_group(0)                           = ?
   ```

   bash forks, the child execs `ls`, the parent waits. Now run `strace -e trace=execve bash -c 'exec ls / > /dev/null'` and notice there is no `clone`; `exec` replaced bash in place.

4. Redirect file descriptors manually.

   ```bash
   exec 3> /tmp/lab1.log          # open fd 3 for writing
   echo "hello via fd 3" >&3
   ls -l /proc/$$/fd/3
   exec 3>&-                      # close it
   cat /tmp/lab1.log
   ```

   Expected output:

   ```text
   l-wx------ 1 lab lab 64 ... 3 -> /tmp/lab1.log
   hello via fd 3
   ```

5. Signals and exit codes.

   ```bash
   kill -TERM $SP; wait $SP; echo "exit=$?"
   sleep 600 & SP=$!; kill -KILL $SP; wait $SP; echo "exit=$?"
   sleep 600 & SP=$!; kill -STOP $SP; grep State /proc/$SP/status; kill -CONT $SP; grep State /proc/$SP/status; kill $SP
   ```

   Expected output:

   ```text
   exit=143
   exit=137
   State:  T (stopped)
   State:  S (sleeping)
   ```

6. Make a zombie and reap it.

   ```bash
   bash -c 'sleep 1 & sleep 30' &
   sleep 2; ps -o pid,ppid,stat,comm --ppid $(pgrep -f 'sleep 30' | head -1)
   ```

   Expected output shows `Z` for the finished `sleep 1`:

   ```text
     PID    PPID STAT COMMAND
    5011    5010 Z    sleep <defunct>
    5012    5010 S    sleep
   ```

   When the outer `bash -c` exits after 30 s, systemd (PID 1) reaps the zombie. In a container without an init, nobody would.

### Verify

`echo $?` after a `kill -9`'d process is `137`; you can explain where that number comes from and which Kubernetes field (`lastState.terminated.exitCode`) will show it.

### Cleanup

```bash
pkill -f 'sleep 600'; pkill -f 'sleep 30'; rm -f /tmp/lab1.log
```

### What you learned

A process is a directory in `/proc`; stdin/stdout/stderr are fds 0/1/2 that the shell rearranges before `execve`; `exec` without fork is what container entrypoints need; exit code `128+N` identifies the killing signal.

---

## Lab 2: Deleted-but-Open Files and Inode Exhaustion

**Objective.** Reproduce the two classic "disk full" mysteries and fix them without rebooting.

**Prerequisites.** Lab 1.

**Time.** 25 minutes.

### Steps

1. Create a small loop-mounted filesystem so you cannot hurt the root disk.

   ```bash
   sudo mkdir -p /mnt/lab2
   sudo dd if=/dev/zero of=/var/tmp/lab2.img bs=1M count=64 status=none
   sudo mkfs.ext4 -q -N 2048 /var/tmp/lab2.img         # only 2048 inodes
   sudo mount -o loop /var/tmp/lab2.img /mnt/lab2
   sudo chown "$USER" /mnt/lab2
   df -h /mnt/lab2; df -i /mnt/lab2
   ```

2. Hold a big file open, delete it, and watch `df` disagree with `du`.

   ```bash
   dd if=/dev/zero of=/mnt/lab2/big.log bs=1M count=40 status=none
   tail -f /mnt/lab2/big.log > /dev/null &
   TP=$!
   rm /mnt/lab2/big.log
   df -h /mnt/lab2 | tail -1
   du -sh /mnt/lab2
   ```

   Expected output:

   ```text
   /dev/loop0   55M   41M   10M  81% /mnt/lab2
   24K   /mnt/lab2
   ```

3. Find the culprit and reclaim the space without killing the process.

   ```bash
   sudo lsof +L1 | grep lab2
   ls -l /proc/$TP/fd | grep deleted
   : > /proc/$TP/fd/3            # truncate through the fd (adjust fd number from the ls output)
   df -h /mnt/lab2 | tail -1
   ```

   Expected output:

   ```text
   tail  5210  lab  3r  REG  7,0  41943040  0  12 /mnt/lab2/big.log (deleted)
   /dev/loop0   55M   1M   50M   2% /mnt/lab2
   ```

4. Exhaust inodes with free space remaining.

   ```bash
   for i in $(seq 1 2100); do : > /mnt/lab2/f$i 2>/dev/null || { echo "failed at $i"; break; }; done
   df -h /mnt/lab2 | tail -1; df -i /mnt/lab2 | tail -1
   ```

   Expected output:

   ```text
   failed at 2038
   /dev/loop0   55M  1.2M   50M   3% /mnt/lab2
   /dev/loop0   2048  2048     0  100% /mnt/lab2
   ```

   `touch` fails with "No space left on device" while `df -h` shows 97% free. Find where inodes went: `sudo find /mnt/lab2 -xdev -printf '%h\n' | sort | uniq -c | sort -rn | head`.

### Verify

You can state the two commands that resolve each case (`lsof +L1` then truncate via `/proc/<pid>/fd/N`; `df -i` then find the directory with the most files).

### Cleanup

```bash
kill $TP; sudo umount /mnt/lab2; sudo rm -f /var/tmp/lab2.img; sudo rmdir /mnt/lab2
```

### What you learned

Space belongs to inodes, not names; a file is freed when the last fd closes; filesystems have a fixed inode budget. Both failure modes appear on Kubernetes nodes under `/var/lib/containerd` and `/var/log/pods`.

---

## Lab 3: A Hardened systemd Service with Resource Limits and journald

**Objective.** Run a small Python HTTP service as a sandboxed, resource-limited systemd unit; observe restarts, logs, and a live cgroup change.

**Prerequisites.** Python 3.12 (`sudo apt-get install -y python3`).

**Time.** 45 minutes.

### Steps

1. Create a service account and the application.

   ```bash
   sudo useradd --system --user-group --home-dir /opt/lab-api --shell /usr/sbin/nologin labapi
   sudo install -d -m 0750 -o labapi -g labapi /opt/lab-api /var/lib/lab-api
   sudo tee /opt/lab-api/server.py > /dev/null <<'EOF'
   import http.server, os, signal, sys, time
   class H(http.server.BaseHTTPRequestHandler):
       def do_GET(self):
           if self.path == "/leak":
               global blob; blob = bytearray(300 * 1024 * 1024)   # allocate 300 MB
           if self.path == "/crash":
               os._exit(3)
           body = f"pid={os.getpid()} uid={os.getuid()} path={self.path}\n".encode()
           self.send_response(200); self.send_header("Content-Length", str(len(body))); self.end_headers()
           self.wfile.write(body)
       def log_message(self, fmt, *args):
           print(f"{self.address_string()} {fmt % args}", flush=True)
   def term(*_):
       print("SIGTERM received, shutting down gracefully", flush=True); sys.exit(0)
   signal.signal(signal.SIGTERM, term)
   print("starting on 127.0.0.1:8080", flush=True)
   http.server.ThreadingHTTPServer(("127.0.0.1", 8080), H).serve_forever()
   EOF
   sudo chown labapi:labapi /opt/lab-api/server.py
   ```

2. Write the unit with sandboxing and limits.

   ```bash
   sudo tee /etc/systemd/system/lab-api.service > /dev/null <<'EOF'
   [Unit]
   Description=Lab API
   After=network-online.target
   Wants=network-online.target
   StartLimitIntervalSec=60
   StartLimitBurst=5

   [Service]
   Type=simple
   User=labapi
   Group=labapi
   WorkingDirectory=/opt/lab-api
   ExecStart=/usr/bin/python3 /opt/lab-api/server.py
   Restart=on-failure
   RestartSec=1s
   TimeoutStopSec=10s
   KillSignal=SIGTERM

   MemoryMax=200M
   MemoryHigh=160M
   CPUQuota=50%
   TasksMax=64

   NoNewPrivileges=yes
   ProtectSystem=strict
   ProtectHome=yes
   PrivateTmp=yes
   PrivateDevices=yes
   ProtectKernelTunables=yes
   ProtectKernelModules=yes
   ProtectControlGroups=yes
   RestrictAddressFamilies=AF_INET AF_INET6
   RestrictNamespaces=yes
   LockPersonality=yes
   SystemCallFilter=@system-service
   SystemCallErrorNumber=EPERM
   CapabilityBoundingSet=
   ReadWritePaths=/var/lib/lab-api
   UMask=0077

   [Install]
   WantedBy=multi-user.target
   EOF
   sudo systemd-analyze verify /etc/systemd/system/lab-api.service
   sudo systemctl daemon-reload
   sudo systemctl enable --now lab-api
   systemctl status lab-api --no-pager
   ```

   Expected output (abridged):

   ```text
   ● lab-api.service - Lab API
        Loaded: loaded (/etc/systemd/system/lab-api.service; enabled; preset: enabled)
        Active: active (running) since ...
      Main PID: 6120 (python3)
         Tasks: 1 (limit: 64)
        Memory: 9.8M (high: 160.0M max: 200.0M)
           CPU: 48ms
        CGroup: /system.slice/lab-api.service
                └─6120 /usr/bin/python3 /opt/lab-api/server.py
   ```

3. Check the sandbox score and the cgroup.

   ```bash
   systemd-analyze security lab-api.service | tail -3
   curl -s http://127.0.0.1:8080/hello
   cat /sys/fs/cgroup/system.slice/lab-api.service/memory.max
   cat /sys/fs/cgroup/system.slice/lab-api.service/cpu.max
   sudo cat /proc/$(systemctl show lab-api -p MainPID --value)/status | grep -E 'CapEff|NoNewPrivs|Seccomp'
   ```

   Expected output:

   ```text
   → Overall exposure level for lab-api.service: 1.9 OK 🙂
   pid=6120 uid=998 path=/hello
   209715200
   50000 100000
   CapEff: 0000000000000000
   NoNewPrivs:     1
   Seccomp:        2
   ```

4. Prove the sandbox: writes outside `ReadWritePaths` fail.

   ```bash
   sudo systemd-run --unit=probe --pipe --property=User=labapi --property=ProtectSystem=strict \
     --property=ReadWritePaths=/var/lib/lab-api sh -c 'touch /opt/lab-api/x; echo rc=$?; touch /var/lib/lab-api/x; echo rc=$?'
   ```

   Expected output:

   ```text
   touch: cannot touch '/opt/lab-api/x': Read-only file system
   rc=1
   rc=0
   ```

5. Trigger the memory limit and watch the OOM kill and restart.

   ```bash
   journalctl -u lab-api -f --no-pager &
   JP=$!
   curl -s --max-time 5 http://127.0.0.1:8080/leak || echo "connection dropped (expected)"
   sleep 3
   cat /sys/fs/cgroup/system.slice/lab-api.service/memory.events
   systemctl show lab-api -p NRestarts -p Result
   kill $JP
   ```

   Expected output (journal lines and files):

   ```text
   systemd[1]: lab-api.service: A process of this unit has been killed by the OOM killer.
   systemd[1]: lab-api.service: Main process exited, code=killed, status=9/KILL
   systemd[1]: lab-api.service: Failed with result 'oom-kill'.
   systemd[1]: lab-api.service: Scheduled restart job, restart counter is at 1.
   low 0
   high 12
   max 41
   oom 1
   oom_kill 1
   NRestarts=1
   Result=oom-kill
   ```

   This is exactly what `OOMKilled` with exit code 137 looks like in Kubernetes, down to the `memory.events` file the kubelet reads.

6. Graceful stop versus crash, and a live limit change.

   ```bash
   sudo systemctl restart lab-api
   sudo systemctl stop lab-api; journalctl -u lab-api -n 3 -o cat
   sudo systemctl start lab-api
   curl -s http://127.0.0.1:8080/crash; sleep 2; systemctl show lab-api -p NRestarts
   sudo systemctl set-property lab-api.service MemoryMax=400M
   cat /sys/fs/cgroup/system.slice/lab-api.service/memory.max
   systemctl cat lab-api | grep -A2 '# /etc/systemd/system.control'
   ```

   Expected output:

   ```text
   SIGTERM received, shutting down gracefully
   lab-api.service: Deactivated successfully.
   Stopped lab-api.service - Lab API.
   NRestarts=1
   419430400
   # /etc/systemd/system.control/lab-api.service.d/50-MemoryMax.conf
   [Service]
   MemoryMax=400M
   ```

### Verify

`systemd-analyze security lab-api.service` reports an exposure level under 3.0; `curl /leak` produces `Result=oom-kill` and an automatic restart; `systemctl stop` logs the graceful message.

### Cleanup

```bash
sudo systemctl disable --now lab-api
sudo rm -f /etc/systemd/system/lab-api.service
sudo rm -rf /etc/systemd/system.control/lab-api.service.d /opt/lab-api /var/lib/lab-api
sudo systemctl daemon-reload; sudo userdel labapi
```

### What you learned

systemd exposes every kernel primitive from the concepts file as a declarative key; `MemoryMax` is `memory.max`; the OOM killer behaviour and exit code 137 are identical under Kubernetes; a service that handles SIGTERM stops in milliseconds, one that does not waits out `TimeoutStopSec`.

---

## Lab 4: Users, Permissions, setuid and Capabilities

**Objective.** Bind a privileged port three ways (root, setuid, file capability), then prove that `no_new_privs` defeats the setuid route.

**Prerequisites.** `gcc` (`build-essential`), `libcap2-bin`.

**Time.** 40 minutes.

### Steps

1. Create a tiny TCP listener that binds port 80.

   ```bash
   mkdir -p ~/lab4 && cd ~/lab4
   cat > bind80.c <<'EOF'
   #include <stdio.h>
   #include <string.h>
   #include <unistd.h>
   #include <arpa/inet.h>
   int main(void) {
     int s = socket(AF_INET, SOCK_STREAM, 0);
     struct sockaddr_in a = { .sin_family = AF_INET, .sin_port = htons(80), .sin_addr.s_addr = htonl(INADDR_LOOPBACK) };
     if (bind(s, (struct sockaddr*)&a, sizeof a) < 0) { perror("bind :80"); return 1; }
     printf("bound :80 as uid=%d euid=%d\n", getuid(), geteuid());
     return 0;
   }
   EOF
   gcc -o bind80 bind80.c
   ./bind80
   ```

   Expected output:

   ```text
   bind :80: Permission denied
   ```

2. Route 1: run as root (what you must not do in production).

   ```bash
   sudo ./bind80
   ```

   Expected output: `bound :80 as uid=0 euid=0`.

3. Route 2: setuid root binary.

   ```bash
   sudo chown root:root bind80 && sudo chmod u+s bind80
   ls -l bind80
   ./bind80
   ```

   Expected output:

   ```text
   -rwsr-xr-x 1 root root 16K ... bind80
   bound :80 as uid=1000 euid=0
   ```

   Note `euid=0`: the whole program runs as root. Any bug in it is a root exploit.

4. Route 3: file capability (least privilege).

   ```bash
   sudo chmod u-s bind80
   sudo setcap cap_net_bind_service=+ep bind80
   getcap bind80
   ./bind80
   ```

   Expected output:

   ```text
   bind80 cap_net_bind_service=ep
   bound :80 as uid=1000 euid=1000
   ```

   The process never became root; it holds one capability.

5. `no_new_privs` blocks both escalation routes.

   ```bash
   sudo chmod u+s bind80
   setpriv --no-new-privs ./bind80
   sudo chmod u-s bind80; sudo setcap cap_net_bind_service=+ep bind80
   setpriv --no-new-privs ./bind80
   ```

   Expected output for both:

   ```text
   bind :80: Permission denied
   ```

   This is what `allowPrivilegeEscalation: false` and systemd `NoNewPrivileges=yes` do.

6. Drop a capability from an otherwise-root process.

   ```bash
   sudo capsh --drop=cap_net_bind_service -- -c './bind80'
   sudo capsh --print | grep -E '^Current' | head -1
   ```

   Expected output: `bind :80: Permission denied` even though uid is 0.

7. Permissions puzzle: a directory you can write but not list.

   ```bash
   mkdir -m 0300 dropbox; echo secret > dropbox/note; ls dropbox; cat dropbox/note
   chmod 0700 dropbox; ls dropbox
   ```

   Expected output: `ls: cannot open directory 'dropbox': Permission denied` then `secret`, then after chmod the listing `note`. Directory `x` without `r` allows access by exact name only.

8. setgid directories and umask.

   ```bash
   sudo groupadd devs; sudo usermod -aG devs "$USER"
   sudo install -d -m 2770 -g devs /srv/shared
   sg devs -c 'umask 007; touch /srv/shared/a; ls -l /srv/shared/a'
   ```

   Expected output: `-rw-rw---- 1 lab devs 0 ... /srv/shared/a` — group inherited from the directory, no world bits because of umask 007.

### Verify

`getcap bind80` shows `cap_net_bind_service=ep` and the binary runs unprivileged; under `setpriv --no-new-privs` it fails.

### Cleanup

```bash
cd ~ && rm -rf ~/lab4; sudo rm -rf /srv/shared; sudo groupdel devs
```

### What you learned

Capabilities split root into 41 pieces; file capabilities beat setuid; `no_new_privs` is the single bit that makes "non-root container" meaningful; `securityContext.capabilities.add: [NET_BIND_SERVICE]` is this lab in YAML.

---

## Lab 5: cgroups v2 by Hand: CPU Throttling and OOM Kill

**Objective.** Create a cgroup, put processes in it, observe `cpu.stat` throttling and `memory.events` OOM kills, and read PSI.

**Prerequisites.** `stress-ng` (`sudo apt-get install -y stress-ng`), cgroups v2 confirmed (`stat -fc %T /sys/fs/cgroup` = `cgroup2fs`).

**Time.** 40 minutes.

### Steps

1. Create a cgroup with controllers enabled.

   ```bash
   sudo mkdir -p /sys/fs/cgroup/lab5
   cat /sys/fs/cgroup/cgroup.subtree_control
   echo "+cpu +memory +pids +io" | sudo tee /sys/fs/cgroup/cgroup.subtree_control > /dev/null 2>&1 || true
   ls /sys/fs/cgroup/lab5 | tr '\n' ' '
   ```

   Expected output includes `cpu.max cpu.stat memory.max memory.events pids.max cgroup.procs`.

2. Limit to 0.5 CPU and run two busy loops inside.

   ```bash
   echo "50000 100000" | sudo tee /sys/fs/cgroup/lab5/cpu.max
   sudo bash -c 'echo $$ > /sys/fs/cgroup/lab5/cgroup.procs; exec stress-ng --cpu 2 --timeout 20s' &
   sleep 10
   cat /sys/fs/cgroup/lab5/cpu.stat
   top -b -n1 | grep stress | head -2
   ```

   Expected output:

   ```text
   usage_usec 5021345
   user_usec 5019876
   system_usec 1469
   nr_periods 100
   nr_throttled 100
   throttled_usec 14978655
   ```

   Two CPU-bound workers in 10 s wanted 20 CPU-seconds and got about 5; `nr_throttled` equals `nr_periods`: throttled every period. `top` shows each at about 25%. This is a Kubernetes `limits.cpu: 500m` Pod under load; latency-sensitive services suffer from exactly this.

3. Compare with `cpu.weight` (no hard cap, proportional sharing).

   ```bash
   wait
   echo max | sudo tee /sys/fs/cgroup/lab5/cpu.max
   sudo mkdir -p /sys/fs/cgroup/lab5b
   echo 100 | sudo tee /sys/fs/cgroup/lab5/cpu.weight; echo 300 | sudo tee /sys/fs/cgroup/lab5b/cpu.weight
   sudo bash -c 'echo $$ > /sys/fs/cgroup/lab5/cgroup.procs; exec stress-ng --cpu "$(nproc)" --timeout 15s' &
   sudo bash -c 'echo $$ > /sys/fs/cgroup/lab5b/cgroup.procs; exec stress-ng --cpu "$(nproc)" --timeout 15s' &
   sleep 8; systemd-cgtop -b -n1 --depth=1 | grep -E 'lab5'
   wait
   ```

   Expected output shows `lab5b` getting roughly three times the CPU of `lab5` (for example 150% vs 50% on a 2-CPU VM). Weight only matters under contention; with idle CPU both would get what they ask for. This is `requests.cpu`.

4. Memory limit and OOM kill.

   ```bash
   echo 100M | sudo tee /sys/fs/cgroup/lab5/memory.max
   echo 1 | sudo tee /sys/fs/cgroup/lab5/memory.oom.group
   sudo bash -c 'echo $$ > /sys/fs/cgroup/lab5/cgroup.procs; exec stress-ng --vm 1 --vm-bytes 300M --vm-keep --timeout 20s'
   echo "exit=$?"
   cat /sys/fs/cgroup/lab5/memory.events
   sudo dmesg -T | grep -iE 'oom-kill|Killed process' | tail -2
   ```

   Expected output:

   ```text
   stress-ng: info: [7210] dispatching hogs: 1 vm
   Killed
   exit=137
   low 0
   high 0
   max 1583
   oom 1
   oom_kill 2
   [...] oom-kill:constraint=CONSTRAINT_MEMCG,...,task=stress-ng-vm,pid=7212,uid=0
   [...] Memory cgroup out of memory: Killed process 7212 (stress-ng-vm) total-vm:... anon-rss:98304kB
   ```

   `max 1583` counts how many times the limit was hit before the kill (reclaim attempts). `oom.group=1` killed both processes, the behaviour Kubernetes relies on so a Pod container dies whole.

5. PIDs limit (fork bomb protection).

   ```bash
   echo 20 | sudo tee /sys/fs/cgroup/lab5/pids.max
   echo max | sudo tee /sys/fs/cgroup/lab5/memory.max
   sudo bash -c 'echo $$ > /sys/fs/cgroup/lab5/cgroup.procs; for i in $(seq 1 30); do sleep 30 & done; sleep 1; echo "spawned: $(jobs -r | wc -l)"; cat /sys/fs/cgroup/lab5/pids.events; kill $(jobs -p)'
   ```

   Expected output:

   ```text
   bash: fork: retry: Resource temporarily unavailable
   ...
   spawned: 18
   max 12
   ```

6. Read pressure stall information while a limit is hit.

   ```bash
   echo "20000 100000" | sudo tee /sys/fs/cgroup/lab5/cpu.max
   sudo bash -c 'echo $$ > /sys/fs/cgroup/lab5/cgroup.procs; exec stress-ng --cpu 2 --timeout 10s' &
   sleep 6; cat /sys/fs/cgroup/lab5/cpu.pressure; wait
   ```

   Expected output:

   ```text
   some avg10=78.12 avg60=24.53 avg300=5.71 total=6128893
   full avg10=78.12 avg60=24.53 avg300=5.71 total=6128893
   ```

   `some avg10=78` means that in the last 10 s, tasks in this cgroup were stalled waiting for CPU 78% of the time. node_exporter's `node_pressure_cpu_waiting_seconds_total` and cAdvisor's `container_pressure_*` metrics come from these files.

### Verify

You can predict `nr_throttled` for a given `cpu.max` and workload, and `memory.events` shows `oom_kill` after the memory test.

### Cleanup

```bash
sudo rmdir /sys/fs/cgroup/lab5 /sys/fs/cgroup/lab5b
```

### What you learned

`cpu.max` is a hard quota that throttles; `cpu.weight` is a share that only matters under contention; the memory limit includes cache and ends in SIGKILL; PSI is the saturation metric that tells you a limit is actively hurting.

---

## Lab 6: Namespaces by Hand: pid, mount, uts, net

**Objective.** Create each namespace type with `unshare`, see what changes, join namespaces with `nsenter`, and connect a network namespace to the host with a veth pair.

**Prerequisites.** Root on the VM.

**Time.** 45 minutes.

### Steps

1. UTS and PID namespaces.

   ```bash
   sudo unshare --uts --pid --fork --mount-proc bash
   # inside:
   hostname lab6-container; hostname
   echo $$; ps -ef
   ls -l /proc/self/ns/pid /proc/self/ns/uts
   ```

   Expected output inside:

   ```text
   lab6-container
   1
   UID          PID    PPID  C STIME TTY          TIME CMD
   root           1       0  0 10:40 pts/1    00:00:00 bash
   root           8       1  0 10:40 pts/1    00:00:00 ps -ef
   ```

   Your shell is PID 1 and sees only itself; the hostname changed. In another terminal on the host: `hostname` is unchanged and `ps -ef | grep 'unshare'` shows the real PID of that bash. `--mount-proc` mounted a fresh `/proc` so `ps` reads the new PID namespace; without it, `ps` would show host processes.

2. See it from outside, then enter it.

   ```bash
   # host terminal
   CP=$(pgrep -f 'unshare --uts --pid' | head -1); CHILD=$(pgrep -P "$CP")
   sudo lsns -p "$CHILD" | grep -E 'uts|pid'
   sudo nsenter --target "$CHILD" --uts --pid --mount -- hostname
   sudo nsenter --target "$CHILD" --uts --pid --mount -- ps -ef
   ```

   Expected output: `lab6-container` and the two-process listing. `nsenter` is `docker exec`. Exit the namespaced shell with `exit`.

3. Mount namespace: private mounts.

   ```bash
   sudo unshare --mount bash -c '
     mount -t tmpfs -o size=16M tmpfs /mnt
     echo inside > /mnt/marker
     findmnt /mnt
     ls /mnt'
   ls /mnt; findmnt /mnt || echo "host /mnt unchanged"
   ```

   Expected output: inside the namespace `findmnt` shows a tmpfs on `/mnt` with the marker; on the host `/mnt` is empty and `findmnt` prints nothing. This is `PrivateTmp=` and every container's rootfs.

4. Network namespace with a veth pair to the host.

   ```bash
   sudo ip netns add lab6
   sudo ip netns exec lab6 ip -brief link
   sudo ip link add veth-host type veth peer name veth-ns
   sudo ip link set veth-ns netns lab6
   sudo ip addr add 10.200.0.1/24 dev veth-host && sudo ip link set veth-host up
   sudo ip netns exec lab6 ip addr add 10.200.0.2/24 dev veth-ns
   sudo ip netns exec lab6 ip link set veth-ns up
   sudo ip netns exec lab6 ip link set lo up
   sudo ip netns exec lab6 ip route add default via 10.200.0.1
   sudo ip netns exec lab6 ip -brief address
   ping -c 2 10.200.0.2
   ```

   Expected output:

   ```text
   lo               UNKNOWN        127.0.0.1/8
   veth-ns@if12     UP             10.200.0.2/24
   64 bytes from 10.200.0.2: icmp_seq=1 ttl=64 time=0.064 ms
   ```

   Before the veth, the namespace had only a down `lo`: a container with `--network none`.

5. NAT the namespace to the internet (what Docker's default bridge does).

   ```bash
   sudo sysctl -w net.ipv4.ip_forward=1
   OUT=$(ip route show default | awk '{print $5}')
   sudo iptables -t nat -A POSTROUTING -s 10.200.0.0/24 -o "$OUT" -j MASQUERADE
   sudo ip netns exec lab6 ping -c 2 1.1.1.1
   sudo ip netns exec lab6 sh -c 'echo "nameserver 1.1.1.1" > /etc/resolv.conf' 2>/dev/null || true
   sudo mkdir -p /etc/netns/lab6 && echo "nameserver 1.1.1.1" | sudo tee /etc/netns/lab6/resolv.conf
   sudo ip netns exec lab6 curl -sI https://example.com | head -1
   sudo conntrack -L 2>/dev/null | grep 10.200.0.2 | head -2 || sudo cat /proc/net/nf_conntrack | grep 10.200.0.2 | head -2
   ```

   Expected output: `64 bytes from 1.1.1.1 ...`, `HTTP/2 200`, and a conntrack entry mapping `src=10.200.0.2 ... [src=<HOST_IP>]`. Full networking detail is in [../networking/04-hands-on-labs.md](../networking/04-hands-on-labs.md).

6. A service inside the namespace, visible via `nsenter` and `ss`.

   ```bash
   sudo ip netns exec lab6 python3 -m http.server 8080 --bind 10.200.0.2 > /dev/null 2>&1 &
   sleep 1; curl -s http://10.200.0.2:8080/ | head -3
   sudo ip netns exec lab6 ss -tlnp
   ss -tlnp | grep 8080 || echo "not visible in host namespace"
   ```

   Expected output: the directory listing HTML, the `ss` inside shows `*:8080` with `python3`, and the host `ss` shows nothing on 8080. Ports are per network namespace, which is why two Pods can both listen on 8080.

### Verify

`sudo lsns` lists the extra uts/pid/mnt/net namespaces you created while they exist, with distinct inode numbers from PID 1's.

### Cleanup

```bash
sudo pkill -f 'http.server 8080'
sudo iptables -t nat -D POSTROUTING -s 10.200.0.0/24 -o "$OUT" -j MASQUERADE
sudo ip netns delete lab6            # also deletes veth-ns and thereby veth-host
sudo rm -rf /etc/netns/lab6
```

### What you learned

Each namespace type isolates one global table; `unshare` creates, `nsenter` joins; a network namespace is a complete private stack connected by a veth pair and NAT, which is the Docker bridge model and the per-Pod model.

---

## Lab 7: Performance Investigation with the USE Method, strace, perf and bpftrace

**Objective.** Given a slow "mystery" service, find the bottleneck using the USE method, then confirm it with syscall tracing and profiling.

**Prerequisites.** `stress-ng`, `bpftrace`, `bpfcc-tools`, `linux-tools-$(uname -r)` (skip perf on WSL2), `sysstat`.

**Time.** 60 minutes.

### Steps

1. Create the mystery workload: a script that is slow because of `fsync` storms and a CPU hog hidden in a cgroup.

   ```bash
   mkdir -p ~/lab7 && cd ~/lab7
   cat > slow.py <<'EOF'
   import os, time, hashlib
   fd = os.open("/var/tmp/lab7.dat", os.O_WRONLY | os.O_CREAT, 0o600)
   t0 = time.time(); n = 0
   while time.time() - t0 < 60:
       os.write(fd, hashlib.sha256(str(n).encode()).hexdigest().encode())
       os.fsync(fd); n += 1      # fsync on every 64-byte write
   print(f"writes={n} rate={n/60:.0f}/s")
   EOF
   python3 slow.py &
   SLOW=$!
   sudo systemd-run --unit=hog --quiet -p CPUQuota=80% stress-ng --cpu 1 --timeout 90s
   ```

2. USE method, first pass: utilisation and saturation per resource.

   ```bash
   uptime
   vmstat 1 3
   mpstat -P ALL 1 1 | tail -4
   iostat -xz 1 2 | tail -6
   cat /proc/pressure/cpu /proc/pressure/io
   ```

   Expected observations:

   ```text
   procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
    r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
    2  1      0 2814520  41232 611232    0    0     0  1840 2415 4890 42  9 38 11  0
   Device   r/s  w/s  rkB/s  wkB/s  await aqu-sz %util
   vda      0.0 920.0  0.0  3680.0  1.04  0.95   96.4
   some avg10=14.21 ...            (cpu)
   some avg10=22.87 ...            (io)
   ```

   `b=1` and `wa=11` plus `%util` near 100% with tiny `wkB/s` says the disk is saturated by many small writes. `r=2` on a 2-CPU box with `us=42` says something is also hogging CPU.

3. Who is doing the I/O, who is burning CPU?

   ```bash
   sudo pidstat -d 1 2 | awk '$NF ~ /python|stress/'
   pidstat -u 1 2 | awk '$NF ~ /python|stress/'
   sudo iotop -boPa -n 2 2>/dev/null | head -8
   systemd-cgtop -b -n1 --depth=2 | head -8
   ```

   Expected output: `python3` with high `kB_wr/s` and thousands of writes/s; `stress-ng-cpu` near 80% CPU inside `hog.service`.

4. Confirm with syscall tracing.

   ```bash
   sudo strace -c -p $SLOW -f -e trace=write,fsync & sleep 5; sudo kill -INT %1; wait %1 2>/dev/null
   ```

   Expected output:

   ```text
   % time     seconds  usecs/call     calls    errors syscall
   ------ ----------- ----------- --------- --------- ----------------
    99.12    4.871223        1062      4588           fsync
     0.88    0.043201           9      4588           write
   ```

   `fsync` dominates at about 1 ms per call. Now the same with low overhead using eBPF:

   ```bash
   sudo bpftrace -e 'tracepoint:syscalls:sys_enter_fsync { @start[tid] = nsecs; }
     tracepoint:syscalls:sys_exit_fsync /@start[tid]/ { @us = hist((nsecs - @start[tid]) / 1000); delete(@start[tid]); }
     interval:s:5 { exit(); }'
   ```

   Expected output: a histogram with most samples in the `[512, 1K)` or `[1K, 2K)` microsecond buckets.

5. Profile the CPU hog with perf (Multipass only).

   ```bash
   HOG=$(systemctl show hog -p MainPID --value)
   sudo perf record -F 99 -g -p "$HOG" -- sleep 10 > /dev/null 2>&1
   sudo perf report --stdio --no-children 2>/dev/null | grep -v '^#' | head -15
   ```

   Expected output lists `stress-ng` hot functions (e.g. `stress_cpu_*`) at the top with their percentage. Then `sudo runqlat-bpfcc 5 1` shows run-queue latency rising into the millisecond buckets: proof that CPU saturation is delaying other tasks.

6. Fix and measure.

   ```bash
   sudo systemctl stop hog
   kill $SLOW; wait $SLOW 2>/dev/null
   sed -i 's/os.fsync(fd); n += 1/n += 1\n       if n % 1000 == 0: os.fsync(fd)/' slow.py
   timeout 20 python3 slow.py; iostat -xz 1 2 | tail -3
   ```

   Expected output: the write rate rises from roughly 1,000/s to hundreds of thousands/s, `%util` drops to near zero. Batching `fsync` is the same trade-off databases expose as `commit_delay` or `innodb_flush_log_at_trx_commit`.

### Verify

You can write down, for each of CPU/memory/disk/network, the one command you ran for utilisation and the one for saturation, and the tool that confirmed the root cause.

### Cleanup

```bash
sudo systemctl stop hog 2>/dev/null; sudo systemctl reset-failed hog 2>/dev/null
rm -rf ~/lab7 /var/tmp/lab7.dat; sudo rm -f perf.data
```

### What you learned

USE finds *which* resource; `pidstat`/`iotop`/`cgtop` find *who*; `strace -c`/`bpftrace` find *what syscall*; `perf` finds *which function*. Memorise that ladder; the Phase 6 incident agent follows it.

---

## Lab 8: Build a Container from Scratch

**Objective.** Without Docker, create an Alpine root filesystem, run a shell in it with new pid/mount/uts/ipc/net namespaces, `pivot_root` into it, place it in a cgroup with limits, connect it to the network, and drop capabilities. Then compare with what `runc` does.

**Prerequisites.** Labs 5 and 6; `curl`, `iptables`, `libcap2-bin`.

**Time.** 75 minutes.

### Steps

1. Fetch and unpack an Alpine minirootfs (this is an "image layer").

   ```bash
   sudo mkdir -p /var/lib/lab8/rootfs && cd /var/lib/lab8
   ALPINE_VER=3.20.3
   sudo curl -fsSL -o alpine.tar.gz "https://dl-cdn.alpinelinux.org/alpine/v${ALPINE_VER%.*}/releases/x86_64/alpine-minirootfs-${ALPINE_VER}-x86_64.tar.gz"
   sudo curl -fsSL -o alpine.tar.gz.sha256 "https://dl-cdn.alpinelinux.org/alpine/v${ALPINE_VER%.*}/releases/x86_64/alpine-minirootfs-${ALPINE_VER}-x86_64.tar.gz.sha256"
   sudo sh -c 'cd /var/lib/lab8 && sed "s# .*# alpine.tar.gz#" alpine.tar.gz.sha256 | sha256sum -c -'
   sudo tar -xzf alpine.tar.gz -C rootfs
   ls rootfs
   ```

   Expected output: `alpine.tar.gz: OK` then `bin dev etc home lib media mnt opt proc root run sbin srv sys tmp usr var`. On ARM64 use `aarch64` in the URLs.

2. Add an overlay writable layer (so the image stays pristine, like Docker).

   ```bash
   sudo mkdir -p upper work merged
   sudo mount -t overlay overlay -o lowerdir=/var/lib/lab8/rootfs,upperdir=/var/lib/lab8/upper,workdir=/var/lib/lab8/work /var/lib/lab8/merged
   findmnt /var/lib/lab8/merged
   ```

3. Prepare the cgroup.

   ```bash
   sudo mkdir -p /sys/fs/cgroup/lab8
   echo "50000 100000" | sudo tee /sys/fs/cgroup/lab8/cpu.max
   echo 128M | sudo tee /sys/fs/cgroup/lab8/memory.max
   echo 64 | sudo tee /sys/fs/cgroup/lab8/pids.max
   ```

4. Prepare the network namespace and veth (as in Lab 6).

   ```bash
   sudo ip netns add lab8
   sudo ip link add veth8h type veth peer name veth8c
   sudo ip link set veth8c netns lab8
   sudo ip addr add 10.201.0.1/24 dev veth8h && sudo ip link set veth8h up
   sudo ip netns exec lab8 sh -c 'ip addr add 10.201.0.2/24 dev veth8c; ip link set veth8c up; ip link set lo up; ip route add default via 10.201.0.1'
   OUT=$(ip route show default | awk '{print $5}')
   sudo sysctl -q -w net.ipv4.ip_forward=1
   sudo iptables -t nat -A POSTROUTING -s 10.201.0.0/24 -o "$OUT" -j MASQUERADE
   echo "nameserver 1.1.1.1" | sudo tee merged/etc/resolv.conf > /dev/null
   ```

5. Write the "runtime" script that does what runc does.

   ```bash
   sudo tee /var/lib/lab8/run.sh > /dev/null <<'EOF'
   #!/bin/bash
   set -euo pipefail
   ROOT=/var/lib/lab8/merged
   # 1. join the cgroup before cloning so children inherit it
   echo $$ > /sys/fs/cgroup/lab8/cgroup.procs
   # 2. new namespaces; join the prepared net ns; fork so we become PID 1 inside
   exec unshare --mount --uts --ipc --pid --fork --net=/run/netns/lab8 bash -c '
     set -euo pipefail
     ROOT='"$ROOT"'
     # 3. make mounts private so nothing leaks to the host
     mount --make-rprivate /
     # 4. bind-mount rootfs onto itself so it is a mount point (pivot_root requires it)
     mount --bind "$ROOT" "$ROOT"
     mkdir -p "$ROOT/oldroot"
     # 5. pivot: new root becomes /, old root goes to /oldroot
     pivot_root "$ROOT" "$ROOT/oldroot"
     cd /
     # 6. essential virtual filesystems
     mount -t proc proc /proc
     mount -t sysfs -o ro sys /sys
     mount -t tmpfs -o nosuid,nodev,mode=755,size=16M tmpfs /dev
     mknod -m 666 /dev/null c 1 3; mknod -m 666 /dev/zero c 1 5; mknod -m 666 /dev/urandom c 1 9; mknod -m 666 /dev/tty c 5 0
     mount -t tmpfs -o nosuid,nodev,noexec,size=32M tmpfs /tmp
     # 7. detach the old root
     umount -l /oldroot && rmdir /oldroot
     hostname lab8
     # 8. drop capabilities, set no_new_privs, become uid 10001, exec the entrypoint
     exec capsh --drop=all --secbits=noroot,no_setuid_fixup,noroot_locked --user=nobody -- -c "exec /bin/sh"
   '
   EOF
   sudo chmod 0750 /var/lib/lab8/run.sh
   ```

   Alpine minirootfs lacks `capsh`; it exists on the host and is executed before the exec into `/bin/sh`... but after `pivot_root` the host binaries are gone. So for this step install `libcap` inside the rootfs first:

   ```bash
   sudo chroot /var/lib/lab8/merged /bin/sh -c 'apk add --no-cache libcap-utils >/dev/null && echo libcap ok'
   sudo chroot /var/lib/lab8/merged /bin/sh -c 'adduser -D -u 10001 app && echo user ok'
   sudo sed -i 's/--user=nobody/--user=app/' /var/lib/lab8/run.sh
   ```

   Expected output: `libcap ok`, `user ok`. (The `chroot` here uses the host network for `apk`; that is a build-time step, like a Dockerfile `RUN`.)

6. Run the container.

   ```bash
   sudo /var/lib/lab8/run.sh
   ```

   Inside the container:

   ```sh
   hostname; id; echo "pid=$$"
   ps
   cat /proc/1/status | grep -E 'CapEff|NoNewPrivs'
   cat /sys/fs/cgroup/memory.max 2>/dev/null || echo "cgroup fs not mounted (expected; we did not create a cgroup ns)"
   mount | head -5
   ls /oldroot 2>&1
   wget -qO- https://example.com | head -3
   apk add --no-cache curl 2>&1 | tail -1
   ```

   Expected output:

   ```text
   lab8
   uid=10001(app) gid=10001(app) groups=10001(app)
   pid=1
   PID   USER     TIME  COMMAND
       1 app       0:00 /bin/sh
       8 app       0:00 ps
   CapEff: 0000000000000000
   NoNewPrivs:     1
   overlay on / type overlay (rw,relatime,lowerdir=...,upperdir=...,workdir=...)
   proc on /proc type proc (rw,relatime)
   sysfs on /sys type sysfs (ro,relatime)
   ls: /oldroot: No such file or directory
   <!doctype html>
   <html>
   <head>
   ERROR: Unable to lock database: Permission denied
   ```

   You are PID 1, uid 10001, zero capabilities, `no_new_privs`, on an overlay root, with internet via NAT, and you cannot install packages (as it should be at runtime).

7. Observe from the host while the container runs (second terminal).

   ```bash
   CPID=$(pgrep -f '^/bin/sh$' | tail -1)
   sudo lsns -p "$CPID" | awk '{print $2, $1}'
   cat /proc/$CPID/cgroup
   cat /sys/fs/cgroup/lab8/memory.current /sys/fs/cgroup/lab8/pids.current
   sudo nsenter -t "$CPID" -m -u -i -n -p -- /bin/sh -c 'hostname; ps'
   ```

   Expected output: distinct `mnt uts ipc pid net` namespace ids, `0::/lab8`, a few MB and 1-2 pids, and `nsenter` behaving like `docker exec`.

8. Prove the limits inside: `sh -c 'for i in $(seq 1 100); do sleep 60 & done'` fails after about 60 forks with "Resource temporarily unavailable"; `head -c 200m /dev/zero | tail` is killed (exit 137). Type `exit` to end the container.

9. Compare with runc's config (optional, if you install `runc`): `runc spec` generates `config.json`; every field (`linux.namespaces`, `linux.resources`, `process.capabilities`, `process.noNewPrivileges`, `root.path`, `mounts`) corresponds to one step above.

### Verify

Inside the container: `id` shows uid 10001, `cat /proc/1/status` shows `CapEff: 0000000000000000` and `NoNewPrivs: 1`, `ls /oldroot` fails, and `wget` reaches the internet. From the host, `cat /proc/<pid>/cgroup` shows `/lab8`.

### Cleanup

```bash
sudo umount /var/lib/lab8/merged 2>/dev/null
sudo iptables -t nat -D POSTROUTING -s 10.201.0.0/24 -o "$OUT" -j MASQUERADE
sudo ip netns delete lab8
sudo rmdir /sys/fs/cgroup/lab8
sudo rm -rf /var/lib/lab8
```

### What you learned

A container is: an overlay root, five or six namespaces, a cgroup, a veth with NAT, dropped capabilities and `no_new_privs`, then `execve`. Docker, containerd and Kubernetes orchestrate exactly these steps. From here, `securityContext`, `resources`, `hostNetwork`, `emptyDir` and image layers are no longer abstractions.

---

## Where to Go Next

- [05-production-patterns.md](05-production-patterns.md) turns these primitives into hardened node and service baselines.
- [../networking/04-hands-on-labs.md](../networking/04-hands-on-labs.md) extends Labs 6 and 8 into bridges, DNS and TLS.
- Cluster 04 Docker labs rebuild Lab 8 with a Dockerfile and compare `docker inspect` with what you did by hand.
