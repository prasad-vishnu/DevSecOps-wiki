# Linux: Commands and Cheatsheet

| | |
|---|---|
| **Cluster** | 00 Foundations |
| **Role** | Prerequisite |
| **Prerequisites** | [02-core-concepts.md](02-core-concepts.md) |
| **Target version** | GNU coreutils 9.x, util-linux 2.39+, iproute2 6.x, systemd 255+, procps-ng 4.x |
| **Capstone phase** | Phase 1 (Months 1-4) |

> Reference organised by task. Every command has a one-line explanation; both long and short flags appear where both are common. Commands marked `sudo` need root. Nothing here is destructive unless labelled **destructive**.

## 1. Orientation: What Machine Am I On?

```bash
hostnamectl                               # hostname, OS, kernel, virtualisation type
cat /etc/os-release                       # distro ID and version
uname -r                                  # kernel release
uptime                                    # uptime and load averages (1, 5, 15 min)
nproc; lscpu | head -20                   # CPU count and topology
free -h                                   # memory incl. buffers/cache and swap
df -hT -x tmpfs -x devtmpfs               # disk usage by real filesystem
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS   # block devices
ip -brief address; ip -brief link         # interfaces and addresses, compact
ip route show                             # routing table
systemd-detect-virt                       # kvm, microsoft (Hyper-V/WSL), none
cat /proc/cmdline                         # kernel boot parameters
last -n 5; who                            # recent logins; who is on now
sudo dmidecode -t system | head           # hardware vendor/model (cloud: Amazon EC2)
curl -s -m 2 http://169.254.169.254/latest/meta-data/instance-id   # EC2 IMDSv1 (v2 needs a token; see cluster 01)
```

## 2. Filesystem and Files

### Navigation and inspection

```bash
ls -la --time-style=long-iso             # long listing with ISO timestamps
ls -li                                   # show inode numbers
tree -L 2 -a /etc/systemd                # directory tree, 2 levels, including dotfiles
stat file                                # inode metadata: mode, links, uid, size, atime/mtime/ctime
file /usr/bin/ls                         # identify file type by magic bytes
realpath ./link; readlink -f ./link      # resolve symlinks to absolute path
du -sh /var/log/* | sort -h | tail       # space per directory, human sorted
du -xsh /                                # size of root fs only (-x = one filesystem)
ncdu -x /                                # interactive du (install ncdu)
df -i                                    # inode usage
findmnt -D; findmnt /var                 # mounts with usage; mount for a path
```

### Finding things

```bash
find /var/log -type f -name '*.log' -size +100M                 # big logs
find /etc -type f -mtime -1                                     # modified in last 24 h
find / -xdev -perm -4000 -type f 2>/dev/null                    # setuid binaries on root fs
find /tmp -type f -atime +7 -delete                             # destructive: delete files not accessed in 7 days
find . -type f -print0 | xargs -0 grep -l 'TODO'                # NUL-safe pipeline
find /proc/*/fd -lname '*(deleted)' 2>/dev/null                 # deleted-but-open files
locate sshd_config; sudo updatedb                               # indexed search (plocate)
which kubectl; type -a python3; command -v jq                   # locate executables in PATH
```

### Create, copy, move, delete

```bash
mkdir -p a/b/c                           # create nested dirs idempotently
install -d -m 0750 -o api -g api /var/lib/api     # mkdir with mode and owner in one step
install -m 0644 src.conf /etc/api/api.conf        # copy file with mode
cp -a src/ dst/                          # archive copy: preserve mode, owner, timestamps, links
rsync -aHAX --delete --info=progress2 src/ dst/   # efficient sync; trailing slash copies contents
mv -i old new                            # interactive move/rename
rm -rf dir                               # destructive: recursive force delete
shred -u secret.txt                      # destructive: overwrite then delete (ineffective on SSDs/CoW fs)
ln -s /opt/app-1.4.2 /opt/app            # symlink
ln -sfn /opt/app-1.4.3 /opt/app          # atomically repoint symlink (release switch pattern)
truncate -s 0 big.log                    # empty a file that a process holds open (frees space)
: > big.log                              # same, shell builtin
mktemp -d; mktemp /tmp/api.XXXXXX        # safe temp dir/file
dd if=/dev/zero of=test.img bs=1M count=100 status=progress   # create a 100 MB file
fallocate -l 1G swap.img                 # preallocate quickly
```

### Permissions and ownership

```bash
chmod 0640 file; chmod -R u=rwX,g=rX,o= dir      # numeric; symbolic (X = x only for dirs/executables)
chmod u+s /usr/local/bin/tool; chmod g+s shared/ # setuid; setgid directory
chmod +t /shared                                 # sticky bit
chown api:api -R /opt/api                        # owner and group recursively
chgrp adm /var/log/app.log                       # group only
umask; umask 027                                 # show / set default mask for this shell
getfacl file; setfacl -m u:deploy:rx /opt/api    # ACLs
setfacl -d -m g:devs:rwx /srv/shared             # default ACL inherited by new files
getcap -r / 2>/dev/null                          # file capabilities on the system
sudo setcap cap_net_bind_service=+ep ./server    # let a binary bind :80 without root
lsattr file; sudo chattr +i file                 # immutable attribute (even root cannot modify until -i)
```

### Archives and compression

```bash
tar -czvf app.tar.gz -C /opt app                 # create gzip tarball of /opt/app
tar -xzvf app.tar.gz -C /tmp                     # extract to /tmp
tar -tzf app.tar.gz | head                       # list contents
tar --use-compress-program=zstd -cvf app.tar.zst app     # zstd (fast, used by container tooling)
zstd -19 -T0 big.log; unzstd big.log.zst         # compress with all cores
gzip -k file; gunzip file.gz; zcat file.gz | less        # gzip; keep original with -k
xz -9 file; unxz file.xz                         # high ratio, slow
zip -r site.zip site/; unzip -l site.zip         # zip
sha256sum file; sha256sum -c SHA256SUMS          # checksums; verify downloads
```

## 3. Text Processing

### grep

```bash
grep -n 'error' app.log                          # line numbers
grep -rn --include='*.py' 'TODO' src/            # recursive, filtered by glob
grep -E '^(ERROR|WARN)' app.log                  # extended regex (alternation, +, ?)
grep -P '\buid=\d+' app.log                      # PCRE
grep -o -E '[0-9]{1,3}(\.[0-9]{1,3}){3}' app.log | sort -u     # extract IPs
grep -v -E '^\s*(#|$)' /etc/ssh/sshd_config      # strip comments and blank lines
grep -c 'GET /health' access.log                 # count matching lines
grep -l 'pattern' *.conf                         # filenames only; -L for files without
grep -A3 -B1 'Traceback' app.log                 # context after/before
grep -i 'warn' app.log                           # case-insensitive
grep -F 'literal.string[0]' file                 # fixed string, no regex
rg -n 'pattern' .                                # ripgrep: faster, respects .gitignore (install ripgrep)
```

### sed

```bash
sed 's/foo/bar/' file                            # first occurrence per line, to stdout
sed 's/foo/bar/g' file                           # all occurrences
sed -i.bak 's/foo/bar/g' file                    # in place with backup
sed -E 's/(user)=[^ ]+/\1=<REDACTED>/g' app.log  # extended regex with capture group
sed -n '10,20p' file                             # print lines 10-20 only
sed -n '/^\[Service\]/,/^\[/p' unit.service      # print a section between two patterns
sed '/^#/d; /^$/d' file                          # delete comment and blank lines
sed '1d; $d' file                                # delete first and last line
sed 's#/old/path#/new/path#g' file               # alternate delimiter
sed -i '/^PermitRootLogin/c\PermitRootLogin no' /etc/ssh/sshd_config   # replace whole line
sed -i '$a\new last line' file                   # append at end
sed 's/^[ \t]*//; s/[ \t]*$//' file              # trim whitespace
```

### awk

```bash
awk '{print $1, $NF}' file                       # first and last field
awk -F: '$3 >= 1000 {print $1}' /etc/passwd      # custom delimiter, condition
awk '{sum += $5} END {print sum/1024/1024 " MB"}' <(ls -l)     # sum a column
awk '{count[$1]++} END {for (k in count) print count[k], k}' access.log | sort -rn | head     # group-by
awk 'NR==FNR {a[$1]; next} $1 in a' list.txt data.txt          # join: lines of data.txt whose col1 is in list.txt
awk 'length($0) > 120' file                      # lines longer than 120 chars
awk -v th=500 '$10 > th' access.log              # pass shell variable
awk 'BEGIN{OFS=","} {$1=$1; print}' file         # re-join fields with comma
awk '!seen[$0]++' file                           # dedupe preserving order
awk '/start/,/end/' file                         # range
ps -eo pid,rss,comm --sort=-rss | awk 'NR<=11 {printf "%-8s %8.1f MB %s\n", $1, $2/1024, $3}'   # top memory, formatted
```

### cut, sort, uniq, tr, column, paste, comm, diff

```bash
cut -d: -f1,7 /etc/passwd                        # fields by delimiter
cut -c1-10 file                                  # characters 1-10
sort -t, -k2,2n -k1,1 file.csv                   # numeric sort on column 2, then column 1
sort -u file; sort | uniq -c | sort -rn          # unique; frequency table
sort -h sizes.txt                                # human sizes (1K, 2M)
uniq -d file                                     # only duplicated lines (input must be sorted)
tr 'a-z' 'A-Z' < file; tr -d '\r' < dos.txt      # translate; strip CR
tr -s ' ' < file                                 # squeeze repeated spaces
column -t -s, file.csv                           # align columns
paste -d, a.txt b.txt                            # join line by line
comm -23 <(sort a) <(sort b)                     # lines only in a
diff -u old new; diff -r dir1 dir2               # unified diff; recursive
wc -l file; wc -c file                           # lines; bytes
head -n 20 file; tail -n 50 -f app.log           # first 20 lines; follow last 50
tail -F app.log                                  # follow across rotation
tac file; rev file                               # reverse lines; reverse chars
nl file                                          # number lines
split -l 1000 big.csv part_                      # split into 1000-line chunks
shuf -n 5 file                                   # random sample
fold -w 80 -s file                               # wrap
```

### jq

```bash
jq . file.json                                   # pretty-print
jq -c . file.json                                # compact one line per document
jq -r '.name' file.json                          # raw string (no quotes)
jq '.items[] | .metadata.name' pods.json         # iterate array
jq '.items | length' pods.json                   # count
jq -r '.items[] | [.metadata.namespace, .metadata.name, .status.phase] | @tsv' pods.json   # TSV
jq -r '.items[] | select(.status.phase != "Running") | .metadata.name' pods.json           # filter
jq '.items[] | select(.metadata.labels.app == "api")' pods.json
jq '[.items[] | .spec.containers[].image] | unique' pods.json  # all images
jq 'map(select(.age > 30)) | sort_by(.name)' people.json
jq 'group_by(.region) | map({region: .[0].region, count: length})' instances.json
jq --arg ns prod '.items[] | select(.metadata.namespace == $ns)' pods.json    # pass variable
jq --argjson n 3 '.[] | select(.replicas > $n)' deploys.json  # numeric variable
jq -n '{name: "api", ports: [8000, 8443]}'       # construct JSON from nothing
jq '.spec.replicas = 3' deploy.json              # set a field
jq 'del(.metadata.managedFields)' obj.json       # delete a field
jq '.a + .b' ; jq 'add'                          # arithmetic; sum array
jq -r 'to_entries[] | "\(.key)=\(.value)"' env.json     # object to KEY=VALUE lines
jq -s '.' a.json b.json                          # slurp multiple files into an array
jq -e '.ok' file.json                            # exit code 1 if false/null (for scripts)
jq '.. | .image? // empty' manifest.json         # recursive descent
jq 'paths(type == "string")' file.json           # all paths to string leaves
jq -r '@base64d' <<< '"aGVsbG8="'                # decode base64
jq '.timestamp | todate' ; jq 'now'              # epoch to ISO; current time
```

### yq (mikefarah v4)

```bash
yq '.spec.replicas' deploy.yaml                  # read
yq -i '.spec.replicas = 3' deploy.yaml           # write in place
yq '.spec.template.spec.containers[0].image' deploy.yaml
yq '.spec.template.spec.containers[] | select(.name == "api") | .image' deploy.yaml
yq 'select(.kind == "Deployment") | .metadata.name' manifests.yaml   # multi-document
yq eval-all 'select(.kind == "Service")' manifests.yaml              # across documents
yq -o json deploy.yaml; yq -P file.json          # YAML to JSON; JSON to YAML (pretty)
yq '.spec.template.spec.containers[0].env += [{"name":"LOG_LEVEL","value":"info"}]' deploy.yaml   # append to list
yq 'del(.metadata.annotations)' deploy.yaml
yq '. as $a | load("values.yaml") * $a' base.yaml       # merge
yq '.data | to_entries | .[] | .key' configmap.yaml
yq --split-exp '.metadata.name + ".yaml"' manifests.yaml  # split multi-doc into files
yq 'explode(.)' anchors.yaml                     # resolve YAML anchors/aliases
yq '... comments=""' file.yaml                   # strip comments
```

## 4. Processes

```bash
ps aux                                           # BSD style, all processes
ps -ef                                           # System V style with PPID
ps -eo pid,ppid,user,stat,ni,%cpu,%mem,rss,etime,comm --sort=-%mem | head    # custom columns
ps -eLf | wc -l                                  # thread count system-wide
ps -o pid,stat,wchan:20,comm -p <pid>            # what a process is waiting on
ps --forest -eo pid,ppid,comm                    # tree
pstree -p <pid>; pstree -ps <pid>                # tree below; ancestors
pgrep -af uvicorn                                # PIDs and command lines matching
pidof nginx                                      # PIDs by exact name
top -b -n 1 | head -20                           # batch snapshot
top -H -p <pid>                                  # threads of one process
htop -p "$(pgrep -d, java)"                      # htop limited to java PIDs
cat /proc/<pid>/status | grep -E 'State|Threads|VmRSS|Cap|Seccomp|NoNewPrivs'
cat /proc/<pid>/cgroup; cat /proc/<pid>/cmdline | tr '\0' ' '
ls -l /proc/<pid>/fd | wc -l; ls -l /proc/<pid>/fd  # open fds
cat /proc/<pid>/limits                           # effective ulimits
cat /proc/<pid>/oom_score /proc/<pid>/oom_score_adj
sudo cat /proc/<pid>/stack                       # kernel stack (why it is in D state)
nice -n 10 cmd; renice -n 5 -p <pid>             # priority
ionice -c3 cmd                                   # idle I/O class
taskset -cp 0-3 <pid>                            # CPU affinity
chrt -p <pid>                                    # scheduling policy
nohup cmd > out.log 2>&1 &                       # survive hangup
setsid cmd &                                     # new session
disown -h %1                                     # detach job from shell
jobs; fg %1; bg %1                               # job control
timeout --signal=TERM --kill-after=10s 60s cmd   # kill after 60 s, SIGKILL 10 s later
watch -n 2 -d 'ss -s'                            # rerun and highlight differences
ulimit -n; ulimit -a; ulimit -c unlimited        # file descriptor limit; all; enable core dumps
prlimit --pid <pid> --nofile=65536:65536         # change a running process's limit
```

### Signals

```bash
kill -l                                          # list signals
kill <pid>; kill -TERM <pid>; kill -15 <pid>     # SIGTERM (default)
kill -KILL <pid>; kill -9 <pid>                  # destructive: uncatchable
kill -HUP $(cat /run/nginx.pid)                  # reload
kill -STOP <pid>; kill -CONT <pid>               # pause/resume
kill -USR1 <pid>                                 # application defined
pkill -f 'uvicorn app.main'                      # by pattern
pkill -TERM -u api                               # all processes of a user
killall -w nginx                                 # by name, wait for exit
kill -0 <pid> && echo alive                      # test existence without signalling
trap 'echo caught' TERM                          # handler in a shell script
```

## 5. systemd and Logging

### systemctl

```bash
systemctl status api.service                     # state, PID, cgroup, recent log lines
systemctl start|stop|restart|reload api          # control
systemctl reload-or-restart api                  # reload if supported, else restart
systemctl enable --now api                       # enable at boot and start now
systemctl disable --now api
systemctl is-active api; systemctl is-enabled api; systemctl is-failed api
systemctl list-units --type=service --state=running
systemctl list-units --failed                    # what is broken
systemctl list-unit-files --state=enabled
systemctl list-dependencies api                  # tree of dependencies
systemctl list-timers --all
systemctl cat api                                # effective unit incl. drop-ins
systemctl show api -p MainPID -p ActiveState -p MemoryCurrent -p CPUUsageNSec   # properties
systemctl show api --property=ExecStart
sudo systemctl edit api                          # create drop-in override (opens editor)
sudo systemctl edit --full api                   # edit the whole unit copy in /etc
sudo systemctl daemon-reload                     # after editing unit files
sudo systemctl reset-failed api                  # clear failed state / start-limit
sudo systemctl mask tmp.mount                    # prevent a unit from ever starting
sudo systemctl set-property api.service MemoryMax=1G   # live cgroup change, persisted
systemctl kill -s SIGUSR1 api                    # signal all processes in the unit's cgroup
systemd-run --unit=batch1 --property=MemoryMax=256M -p CPUQuota=50% /opt/job.sh   # transient unit with limits
systemd-run --scope --user -p MemoryMax=1G bash  # interactive shell in a limited scope
systemd-cgls; systemd-cgtop                      # cgroup tree; live cgroup resource usage
systemd-analyze blame; systemd-analyze critical-chain   # boot timing
systemd-analyze security api.service             # sandboxing score
systemd-analyze verify /etc/systemd/system/api.service  # lint a unit
systemd-analyze cat-config systemd/system.conf   # effective config with drop-ins
systemctl --user status                          # per-user instance
loginctl list-sessions; loginctl enable-linger api      # keep user services running after logout
```

### journalctl

```bash
journalctl -u api -f                             # follow one unit
journalctl -u api -n 200 --no-pager              # last 200 lines
journalctl -u api --since "2025-01-01 10:00" --until "2025-01-01 11:00"
journalctl -u api --since "30 min ago"
journalctl -p err -b                             # priority err and above, this boot (-p warning, -p 3)
journalctl -b -1                                 # previous boot
journalctl --list-boots
journalctl -k; journalctl -k -b -1               # kernel messages (dmesg), also previous boot
journalctl _PID=1234; journalctl _COMM=sshd; journalctl _UID=1000
journalctl -u api -o json-pretty | head -40      # structured fields
journalctl -u api -o cat                         # message only
journalctl -u api -g 'timeout|refused'           # grep (PCRE)
journalctl --disk-usage
sudo journalctl --vacuum-size=500M; sudo journalctl --vacuum-time=14d    # destructive: prune
sudo journalctl --rotate; sudo journalctl --verify
dmesg -T | tail -50; dmesg -T --level=err,warn   # kernel ring buffer with human timestamps
dmesg -Tw                                        # follow kernel messages
sudo dmesg -T | grep -iE 'oom|killed process'    # OOM evidence
logger -t deploy -p user.notice "deployed v1.4.2"   # write to the journal/syslog from scripts
```

## 6. Users, Groups, Authentication

```bash
id; id api; whoami; groups                       # identity
getent passwd api; getent group docker           # NSS lookup (works with LDAP/SSSD too)
sudo adduser --system --group --home /opt/api --shell /usr/sbin/nologin api   # Debian-style service account
sudo useradd --system --user-group --home-dir /opt/api --shell /sbin/nologin api   # portable
sudo useradd -m -s /bin/bash -G sudo,docker jane # interactive user with groups
sudo usermod -aG docker jane                     # append to group (-a is essential; without it groups are replaced)
sudo gpasswd -d jane docker                      # remove from group
sudo userdel -r jane                             # destructive: delete with home
sudo passwd -l root; sudo passwd -e jane         # lock; force change at next login
sudo chage -l jane; sudo chage -M 90 jane        # password ageing
sudo usermod -L jane; sudo usermod -U jane       # lock/unlock
sudo -l; sudo -u api -i; sudo -E cmd             # list rights; shell as user; preserve env
sudo visudo -f /etc/sudoers.d/deploy             # safe sudoers edit
su - api -s /bin/bash                            # switch user (if shell is nologin, -s overrides)
newgrp docker                                    # activate new group in current session
w; who; last -a | head; lastb | head             # sessions; login history; failed logins (sudo)
faillock --user jane; sudo faillock --user jane --reset   # pam_faillock status/reset (RHEL)
sudo cat /etc/shadow | head -3                   # hashed passwords; never paste these anywhere
```

## 7. Capabilities, seccomp, LSM

```bash
capsh --print                                    # current capability sets
capsh --decode=00000000a80425fb                  # decode hex from /proc/<pid>/status (Docker default set)
getpcaps <pid>                                   # capabilities of a process
grep -E 'Cap|NoNewPrivs|Seccomp' /proc/<pid>/status
sudo capsh --drop=cap_net_raw -- -c 'ping -c1 1.1.1.1'   # run without a capability (ping fails)
sudo setcap cap_net_bind_service=+ep /usr/local/bin/server; getcap /usr/local/bin/server
sudo setcap -r /usr/local/bin/server             # remove file capabilities
sudo aa-status                                   # AppArmor profiles loaded (Ubuntu)
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx; sudo aa-enforce ...   # mode switch
sudo journalctl -k | grep -i apparmor            # denials
getenforce; sudo setenforce 0                    # SELinux mode; 0 = permissive (insecure, for diagnosis only)
ls -Z /var/www; ps -eZ | grep nginx              # SELinux labels
sudo ausearch -m AVC -ts recent; sudo sealert -a /var/log/audit/audit.log   # denials
sudo semanage port -l | grep http; sudo semanage port -a -t http_port_t -p tcp 8081
sudo restorecon -Rv /var/www                     # fix labels
sudo setsebool -P httpd_can_network_connect 1    # boolean, persistent
```

## 8. Namespaces and cgroups

### Namespaces

```bash
lsns; lsns -t net; lsns -p <pid>                 # list namespaces
ls -l /proc/<pid>/ns                             # namespace inodes of a process
sudo unshare --mount --uts --ipc --net --pid --fork --mount-proc bash   # new namespaces, shell inside
unshare --user --map-root-user bash              # unprivileged user namespace with fake root
sudo nsenter --target <pid> --net ip addr        # run a command in another process's net ns
sudo nsenter --target <pid> --mount --uts --ipc --net --pid -- bash   # "docker exec" by hand
sudo nsenter -t <pid> -n ss -tlnp                # listening sockets inside a container
sudo ip netns add blue; sudo ip netns exec blue ip link   # named network namespaces
sudo ip netns list; sudo ip netns delete blue
sudo ip link add veth0 type veth peer name veth1 # veth pair (see ../networking/)
sudo ip link set veth1 netns blue
cat /proc/<pid>/uid_map /proc/<pid>/gid_map      # user namespace mappings
cat /etc/subuid /etc/subgid                      # rootless subordinate ranges
```

### cgroups v2

```bash
stat -fc %T /sys/fs/cgroup                       # cgroup2fs means unified v2
cat /sys/fs/cgroup/cgroup.controllers            # available controllers
cat /sys/fs/cgroup/cgroup.subtree_control        # enabled for children
cat /proc/<pid>/cgroup                           # which cgroup a process is in (0::/path)
cat /sys/fs/cgroup/system.slice/api.service/memory.current
cat /sys/fs/cgroup/system.slice/api.service/memory.max
cat /sys/fs/cgroup/system.slice/api.service/memory.events   # oom_kill counter
cat /sys/fs/cgroup/system.slice/api.service/cpu.max         # quota period
cat /sys/fs/cgroup/system.slice/api.service/cpu.stat        # nr_throttled throttled_usec
cat /sys/fs/cgroup/system.slice/api.service/pids.current
cat /sys/fs/cgroup/system.slice/api.service/memory.pressure # PSI for the group
cat /proc/pressure/cpu /proc/pressure/memory /proc/pressure/io   # system PSI
sudo mkdir /sys/fs/cgroup/lab                    # create a cgroup
echo "+cpu +memory +pids" | sudo tee /sys/fs/cgroup/cgroup.subtree_control
echo "50000 100000" | sudo tee /sys/fs/cgroup/lab/cpu.max     # 0.5 CPU
echo "256M" | sudo tee /sys/fs/cgroup/lab/memory.max
echo 100 | sudo tee /sys/fs/cgroup/lab/pids.max
echo $$ | sudo tee /sys/fs/cgroup/lab/cgroup.procs            # move current shell in
sudo rmdir /sys/fs/cgroup/lab                    # remove (must be empty of processes)
systemd-cgls /sys/fs/cgroup/kubepods.slice       # tree view of kubelet cgroups
systemd-cgtop -m                                 # live by memory
cgcreate -g cpu,memory:lab; cgexec -g cpu,memory:lab cmd   # libcgroup helpers (if installed)
```

## 9. Package Management

```bash
# apt
sudo apt-get update && sudo apt-get upgrade -y   # refresh and upgrade
sudo apt-get install -y --no-install-recommends pkg
sudo apt-get purge -y pkg && sudo apt-get autoremove -y
apt-cache policy pkg; apt list --installed | grep pkg; apt-cache madison pkg   # versions available
apt-cache depends pkg; apt-cache rdepends pkg
dpkg -l | grep '^ii' | wc -l; dpkg -L pkg; dpkg -S /usr/bin/jq; dpkg -s pkg
sudo dpkg -i file.deb; sudo apt-get install -f  # install local .deb; fix deps
sudo apt-mark hold linux-image-generic           # pin from upgrades
sudo apt-get install -y pkg=1.2.3-1ubuntu1       # exact version
sudo unattended-upgrade --dry-run --debug        # test automatic security updates
apt-get changelog pkg | head                     # changelog
sudo apt-get clean                               # drop downloaded .debs

# dnf
sudo dnf check-update; sudo dnf upgrade -y; sudo dnf upgrade --security -y
sudo dnf install -y pkg; sudo dnf remove -y pkg; sudo dnf autoremove
dnf info pkg; dnf provides '*/bin/jq'; dnf repoquery --requires pkg
rpm -qa | sort; rpm -ql pkg; rpm -qf /usr/bin/jq; rpm -qi pkg; rpm -V pkg
sudo dnf versionlock add pkg; sudo dnf history; sudo dnf history undo 12
sudo dnf config-manager --add-repo https://example.com/repo.repo
sudo dnf clean all

# apk
sudo apk update; sudo apk upgrade
sudo apk add --no-cache pkg; sudo apk del pkg
apk info; apk info -L pkg; apk info --who-owns /usr/bin/jq; apk search -v pkg
apk add pkg=1.2.3-r0; apk policy pkg
cat /etc/apk/repositories
```

## 10. Performance and Diagnostics

```bash
uptime; cat /proc/loadavg                        # load: runnable + D-state
vmstat 1 5                                       # r b swpd free buff cache si so bi bo in cs us sy id wa st
mpstat -P ALL 1 3                                # per-CPU utilisation incl. %steal, %iowait
pidstat 1 5; pidstat -d 1; pidstat -w 1; pidstat -t -p <pid> 1   # per-process CPU, disk, ctx switches, threads
iostat -xz 1 5                                   # per-device r/s w/s await aqu-sz %util
iotop -oPa                                       # processes doing I/O, accumulated (sudo)
sar -u 1 5; sar -n DEV 1; sar -r 1; sar -q 1     # CPU, network, memory, run queue
sar -u -f /var/log/sysstat/sa$(date +%d -d yesterday)   # historical
free -m; cat /proc/meminfo | head -20
smem -rs pss | head                              # proportional set size per process (install smem)
pmap -x <pid> | tail -1                          # memory map summary
cat /proc/<pid>/smaps_rollup                     # RSS/PSS breakdown
slabtop -o | head -15                            # kernel slab usage
ss -s                                            # socket summary: TCP states, retransmits implied by counters
ss -tulpn                                        # listening TCP/UDP with processes
ss -tan state established | wc -l                # established count
ss -tanp '( dport = :5432 or sport = :5432 )'    # connections to Postgres
ss -ti dst 10.0.1.5                              # TCP internals: rtt, cwnd, retrans, pacing
nstat -az | grep -E 'TcpRetransSegs|TcpExtListenOverflows|TcpExtListenDrops'   # listen queue overflow = somaxconn/backlog too low
cat /proc/net/softnet_stat                       # dropped packets in softirq (col 2)
ethtool -S eth0 | grep -iE 'drop|err' ; ip -s link show eth0
lsof -p <pid>; lsof -i :8000; lsof -u api; lsof +L1   # open files; by port; by user; deleted-open
fuser -v /var/log/app.log; fuser -k 8000/tcp     # who has a file/port; kill them (destructive)
strace -f -tt -T -s 128 -o /tmp/trace.txt -p <pid>     # attach, follow forks, timestamps, durations, long strings
strace -c -p <pid>                               # syscall summary (Ctrl-C to stop)
strace -e trace=network -p <pid>; strace -e trace=%file cmd    # filter by class
strace -e trace=openat -e status=failed cmd      # only failing syscalls (e.g. config not found)
ltrace -c cmd                                    # library call summary
ldd /usr/bin/python3; LD_DEBUG=libs python3 -c '' 2>&1 | head   # library resolution
sudo perf top -g                                 # live hottest functions
sudo perf stat -d cmd                            # counters: cycles, instructions, cache misses
sudo perf record -F 99 -g -p <pid> -- sleep 20 && sudo perf report --stdio | head -50
sudo perf trace -p <pid>                         # low-overhead strace
sudo perf sched latency                          # scheduler latency
sudo bpftrace -l 'tracepoint:syscalls:sys_enter_*' | wc -l      # list probes
sudo bpftrace -e 'kprobe:do_sys_openat2 { @[comm] = count(); }' # count opens by process
sudo execsnoop-bpfcc; sudo opensnoop-bpfcc; sudo tcplife-bpfcc; sudo runqlat-bpfcc; sudo biolatency-bpfcc; sudo offcputime-bpfcc 5
cat /proc/interrupts | head; cat /proc/softirqs | head          # IRQ distribution
sudo turbostat --quiet sleep 5                   # CPU frequency/C-states (bare metal)
numactl --hardware; numastat -p <pid>            # NUMA
```

## 11. Kernel and sysctl

```bash
sysctl -a 2>/dev/null | wc -l                    # all parameters
sysctl net.ipv4.ip_forward                       # read one
sudo sysctl -w net.core.somaxconn=16384          # set at runtime (not persisted)
echo 16384 | sudo tee /proc/sys/net/core/somaxconn     # same via procfs
sudo sysctl --system                             # reload all /etc/sysctl.d/*.conf
sudo sysctl -p /etc/sysctl.d/90-node.conf        # reload one file
sysctl -a --pattern 'net.ipv4.tcp_(tw|fin|keepalive)'  # filter
lsmod | head; modinfo overlay; sudo modprobe br_netfilter; echo br_netfilter | sudo tee /etc/modules-load.d/k8s.conf
cat /sys/kernel/btf/vmlinux > /dev/null && echo "BTF present"
cat /proc/sys/kernel/random/boot_id              # unique per boot
cat /proc/sys/fs/file-nr                         # allocated, free, max fds system-wide
ulimit -Hn; cat /etc/security/limits.d/*.conf    # hard fd limit; PAM limits
sudo sysctl kernel.sysrq=1; echo b | sudo tee /proc/sysrq-trigger   # destructive: immediate reboot (emergency)
```

## 12. SSH

```bash
ssh-keygen -t ed25519 -a 64 -C "workstation-2025" -f ~/.ssh/id_ed25519     # new key
ssh-keygen -lf ~/.ssh/id_ed25519.pub            # fingerprint
ssh-keygen -p -f ~/.ssh/id_ed25519              # change passphrase
ssh-keygen -R <HOST>                             # remove stale known_hosts entry
ssh-keyscan -t ed25519 <HOST> >> ~/.ssh/known_hosts   # pre-trust (verify fingerprint out of band)
ssh-copy-id -i ~/.ssh/id_ed25519.pub lab@<HOST>  # install public key
eval "$(ssh-agent -s)"; ssh-add ~/.ssh/id_ed25519; ssh-add -l   # agent
ssh -J bastion.example.com lab@10.0.1.10         # jump host (ProxyJump)
ssh -L 5432:db.internal:5432 -N bastion          # local port forward, no shell
ssh -R 8080:localhost:8000 remote                # reverse forward
ssh -D 1080 -N bastion                           # SOCKS proxy
ssh -vvv lab@<HOST>                              # debug negotiation and auth
ssh -o StrictHostKeyChecking=accept-new lab@<HOST>  # TOFU without prompt (scripts)
ssh -o BatchMode=yes -o ConnectTimeout=5 lab@<HOST> true   # non-interactive health check
ssh lab@<HOST> 'sudo journalctl -u api -n 50'    # remote command
ssh -t lab@<HOST> 'sudo -i'                      # force TTY for interactive sudo
scp -r ./deploy lab@<HOST>:/tmp/; rsync -avz -e ssh ./deploy lab@<HOST>:/tmp/   # copy
sftp lab@<HOST>
ssh -Q cipher; ssh -Q kex; ssh -Q key            # supported algorithms
sudo sshd -T | grep -iE 'permitrootlogin|passwordauthentication|pubkey'   # effective sshd config
sudo sshd -t                                     # test config before restart
sudo systemctl reload ssh                        # Ubuntu unit name is ssh; RHEL is sshd
sudo journalctl -u ssh -f; sudo journalctl _COMM=sshd | grep -E 'Failed|Accepted'
ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n file message.txt   # sign a file (Git uses -n git)
```

## 13. Disks, Storage, Swap

```bash
lsblk -f; blkid                                  # devices, filesystems, UUIDs
sudo fdisk -l /dev/nvme1n1; sudo parted /dev/nvme1n1 print   # partitions
sudo mkfs.xfs /dev/nvme1n1; sudo mkfs.ext4 -L data /dev/nvme1n1   # destructive: format
sudo mkdir -p /data && sudo mount /dev/nvme1n1 /data
echo 'UUID=<UUID> /data xfs defaults,nofail 0 2' | sudo tee -a /etc/fstab   # persistent; nofail avoids boot hang
sudo mount -a; sudo umount /data; sudo umount -l /data     # mount all; unmount; lazy unmount
sudo mount -o remount,ro /; sudo mount -o remount,rw /     # flip root ro/rw
sudo mount -t tmpfs -o size=512M,mode=1777 tmpfs /mnt/ram  # RAM disk
sudo mount --bind /src /dst; sudo mount -o bind,ro /src /dst
sudo xfs_growfs /data; sudo resize2fs /dev/nvme1n1  # grow after EBS resize
sudo growpart /dev/nvme0n1 1                     # grow partition (cloud-utils)
sudo e2fsck -f /dev/nvme1n1; sudo xfs_repair /dev/nvme1n1   # fsck (unmounted)
sudo tune2fs -m 1 /dev/nvme1n1                   # reserved blocks 1%
swapon --show; free -h | grep -i swap
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
sudo swapoff -a                                  # Kubernetes nodes: swap off
sudo smartctl -a /dev/nvme0 | head -30           # SMART health (bare metal)
sudo nvme list                                   # NVMe devices
sudo blkdiscard /dev/nvme1n1; sudo fstrim -av    # destructive: discard all; trim mounted fs
ionice -c2 -n7 rsync ...                         # low I/O priority
```

## 14. Time, Locale, Misc

```bash
timedatectl; timedatectl set-timezone UTC        # servers run UTC
chronyc tracking; chronyc sources -v             # NTP status (chrony); timesyncd: timedatectl timesync-status
date -u +%Y-%m-%dT%H:%M:%SZ; date -d @1700000000 # ISO UTC; epoch to date
date +%s; date -d 'tomorrow 02:00' +%s           # epoch
localectl; echo $LANG; locale -a                 # locale
hostnamectl set-hostname node-01.example.com     # hostname
cat /etc/hosts; getent hosts example.com         # local resolution
resolvectl status; resolvectl query example.com  # systemd-resolved
env | sort; printenv HOME; export VAR=1; unset VAR
history | tail; !!; !$; fc -l; Ctrl-R            # history
alias; unalias ll; type ll
man 5 sshd_config; man -k cgroup; apropos signal; tldr tar   # documentation
script -q /tmp/session.log                       # record terminal session
tmux new -s lab; tmux attach -t lab; Ctrl-b d    # persistent sessions
screen -S lab; screen -r
curl -fsSL -o file https://example.com/file; wget -q https://example.com/file
base64 -w0 file; base64 -d                       # encode/decode
openssl rand -hex 32                             # random token for labs
shutdown -h now; shutdown -r +5 "reboot in 5"; reboot; systemctl poweroff   # destructive: power
```

## 15. One-Liner Investigations

```bash
# Top 10 processes by RSS
ps -eo rss,pid,user,comm --sort=-rss | head -11 | awk 'NR==1{print; next}{printf "%7.1f MB %6s %-10s %s\n",$1/1024,$2,$3,$4}'

# Which process is listening on each port
sudo ss -tlnpH | awk '{print $4, $6}' | sed -E 's/users:\(\("([^"]+)",pid=([0-9]+).*/\1 \2/' | column -t

# Count TCP connections by state
ss -tan | awk 'NR>1{s[$1]++} END{for (k in s) print s[k], k}' | sort -rn

# Which cgroup is being throttled
for f in /sys/fs/cgroup/*/*/cpu.stat; do t=$(awk '/nr_throttled/{print $2}' "$f"); [ "${t:-0}" -gt 0 ] && echo "$t $f"; done | sort -rn | head

# OOM kills in the last day
sudo journalctl -k --since "1 day ago" | grep -iE 'out of memory|oom-kill|killed process'

# Files open by a service and their sizes
sudo ls -l /proc/$(systemctl show api -p MainPID --value)/fd | awk '{print $NF}' | xargs -I{} sh -c 'test -f "{}" && du -h "{}"' 2>/dev/null | sort -h | tail

# Disk space: largest directories under /var (one filesystem)
sudo du -xh --max-depth=2 /var 2>/dev/null | sort -h | tail -15

# Every unit using more than 200 MB
systemctl show '*' -p Id -p MemoryCurrent --value 2>/dev/null | paste - - | awk '$1!="[not set]" && $1>200*1024*1024 {printf "%7.0f MB %s\n", $1/1024/1024, $2}'

# Who sudo'ed what today
sudo journalctl _COMM=sudo --since today -o cat | grep COMMAND

# Reverse-engineer a container from the host
PID=$(pgrep -f 'uvicorn' | head -1); sudo nsenter -t "$PID" -m -n -p -- sh -c 'hostname; cat /etc/os-release | head -1; ss -tlnp'
```

Next: [04-hands-on-labs.md](04-hands-on-labs.md).
