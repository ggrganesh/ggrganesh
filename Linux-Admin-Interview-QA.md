# Linux Admin Interview Questions & Answers

Covers RHEL 6 through 10 for L1–L3 Linux administration. Answers are written the way you'd say them in an interview: short definition first, then the commands and steps an interviewer expects to hear.

## How the troubleshooting answers are structured

Every scenario question (6, 14, 15, 16, 17, 18, 19, 20, 22, 30, 31) follows the same seven-part incident structure. Learn the structure once and you can answer any troubleshooting question you have never seen before, because you always know what comes next:

| Section | What goes in it |
|---|---|
| **SYMPTOM** | What the user reports and what is observable. Includes how to turn a vague complaint into a precise one, and what the exact error message tells you. |
| **TRIAGE** | The first 90 seconds. A short, fixed set of commands that narrows the problem to a layer before you go deep. |
| **RCA** | Ranked root-cause possibilities — most likely first, so you investigate in probability order instead of randomly. |
| **DIAGNOSIS** | Exact commands, configuration files, log locations and out-of-band consoles that confirm or eliminate each cause. |
| **TEMP FIX** | Immediate mitigation to restore service, ordered least disruptive first, with the risks of each stated. |
| **PERMANENT FIX** | The proper engineering remediation that addresses the cause rather than the symptom. |
| **PREVENTION** | Monitoring, automation, standards and process changes so it does not recur. |

Two things interviewers score heavily and candidates usually skip: **capturing evidence before you reboot** (a reboot often destroys the only chance at root cause), and **separating the temporary fix from the permanent one** — being explicit that you restored service first and then fixed the underlying cause shows you understand incident management, not just commands.

---

## Part 1 — Your Question List

### 1. Difference between RHEL 6, 7, 8, 9 and 10

This is the most common opening question, and the interviewer is really checking whether you understand *why* each version changed, not whether you memorized version numbers. Here is the story in plain English, followed by a table you can use for quick revision.

#### The short spoken answer

"The biggest jump was RHEL 6 to RHEL 7, because Red Hat replaced the old SysV init system with systemd, changed the default filesystem from ext4 to XFS, replaced iptables with firewalld, and moved network configuration to NetworkManager. RHEL 8 was the next big shift — it introduced the BaseOS and AppStream repository model with Application Streams so you can pick a language or database version independently of the OS, replaced yum with dnf, and replaced Docker with Podman. RHEL 9 was more of a modernization and hardening release — it is built on the new CentOS Stream 9 model, the kernel moved to 5.14, cgroups v2 is default, SHA-1 signatures and root SSH password login are disabled by default, and network scripts are gone entirely. RHEL 10 continues that direction with the 6.12 kernel, drops 32-bit support, ships image mode with bootable containers as a first-class way to build servers, and begins the move to post-quantum cryptography."

#### RHEL 6 to RHEL 7 — the architectural break

This is the version pair you will be asked about most, because it is the one that forced administrators to relearn daily commands.

**Init system.** RHEL 6 used SysV init with Upstart. You defined the default runlevel in `/etc/inittab`, started services with `service httpd start`, and made them persistent with `chkconfig httpd on`. Services started one after another, in a fixed order, which made booting slow. RHEL 7 replaced all of that with **systemd**, which starts services in parallel based on dependencies, tracks them in cgroups so no child process escapes, and gives one command for everything: `systemctl start|stop|enable|status httpd`. Runlevels became **targets** — runlevel 3 is now `multi-user.target` and runlevel 5 is `graphical.target`.

**Default filesystem.** RHEL 6 used **ext4**, which topped out around 16 TB. RHEL 7 moved to **XFS**, which scales to 500 TB and handles large files and parallel I/O much better. The practical catch every interviewer waits to hear: **XFS can be grown online with `xfs_growfs` but it can never be shrunk.** With ext4 you could shrink with `resize2fs`. Repair tools also changed from `e2fsck`/`fsck` to `xfs_repair`.

**Firewall.** RHEL 6 used raw **iptables** with a service and a rules file. RHEL 7 introduced **firewalld**, a dynamic zone-based front-end you manage with `firewall-cmd`, where rules can be reloaded without dropping established connections. iptables still worked underneath.

**Networking.** RHEL 6 used the `network` service with `ifconfig` and `route`. RHEL 7 made **NetworkManager** the standard, managed with `nmcli` or the text menu `nmtui`, and the old `ifconfig`/`netstat` commands were deprecated in favour of `ip` and `ss`. Interface names also changed from `eth0`/`eth1` to **consistent device naming** like `ens192`, `eno1` or `enp0s3`, so a NIC keeps the same name across reboots and hardware changes.

**Boot loader and logging.** GRUB 0.97 with `/boot/grub/grub.conf` became **GRUB 2**, where you edit `/etc/default/grub` and regenerate with `grub2-mkconfig` — you never hand-edit `grub.cfg`. Logging gained **journald** alongside rsyslog, so you now also use `journalctl`.

**Other notable changes.** The hostname moved from `/etc/sysconfig/network` to `/etc/hostname`, set with `hostnamectl`. The kernel went from 2.6.32 to 3.10. Docker became supported. `/tmp` could be a tmpfs. And the biggest structural change under the hood was the merge of `/bin`, `/sbin`, `/lib` into `/usr`.

#### RHEL 7 to RHEL 8 — the packaging and container shift

**Repository model.** This is the headline change. RHEL 8 splits content into two repositories: **BaseOS**, which holds the core operating system with a stable ten-year life cycle, and **AppStream**, which holds applications, languages and databases. AppStream introduced **Application Streams** (originally exposed as modules), so you can install Python 3.9 or PostgreSQL 13 on the same OS that someone else runs Python 3.6 on, each with its own shorter support life cycle. You manage them with `dnf module list`, `dnf module enable`, `dnf module install`.

**Package manager.** `yum` was replaced by **dnf** (yum is now just a symlink to dnf). dnf resolves dependencies more reliably, has a proper API, and handles modules. Your muscle memory still works because the syntax is nearly identical.

**Containers.** Docker was removed from RHEL 8. Red Hat replaced it with a **daemonless, rootless** toolset: **Podman** to run containers, **Buildah** to build images, and **Skopeo** to move images between registries. Because there is no daemon, a container crash cannot take down every container on the host, and ordinary users can run containers without root.

**Firewall backend.** firewalld remained the interface, but the backend became **nftables** instead of iptables.

**Networking and other changes.** The legacy `network-scripts` package became deprecated (still present, but NetworkManager is the supported path). The kernel moved to 4.18. Python 3 became the system Python — `python` no longer points anywhere by default, you use `python3` or set it with `alternatives`. Cockpit, the browser-based web console on port 9090, is installed by default. Stratis arrived as a new storage management layer, and VDO gave you inline deduplication and compression. Cryptographic policies became system-wide and centrally managed with `update-crypto-policies`. Authentication configuration moved from `authconfig` to **authselect**. And RHEL 8 was the first release where **in-place upgrade with `leapp`** became a genuinely supported path from the previous major version.

#### RHEL 8 to RHEL 9 — modernization, security defaults and CentOS Stream

**How it was built.** RHEL 9 is the first release developed entirely from **CentOS Stream 9**, meaning the upstream development branch is public and continuous rather than a periodic snapshot. This is also the release where CentOS Linux as a downstream rebuild ended, pushing people to CentOS Stream, AlmaLinux, Rocky Linux, or RHEL developer subscriptions.

**Security defaults got stricter — this is the part that breaks migrations.** **Root login over SSH with a password is disabled by default** (`PermitRootLogin prohibit-password`), so you need key-based auth or a sudo user. **SHA-1 is deprecated for cryptographic signatures**, which breaks older certificates, older SSH keys and some legacy appliances — you have to loosen it deliberately with `update-crypto-policies --set LEGACY`. OpenSSL moved to 3.0, and TLS 1.0/1.1 are gone.

**Networking.** The legacy `/etc/sysconfig/network-scripts` **ifcfg** format is no longer supported at all — NetworkManager keyfiles in `/etc/NetworkManager/system-connections/` are the only supported configuration, and you manage them with `nmcli` or `nmtui`. `iptables-nft` is deprecated in favour of pure nftables.

**Resource management.** **cgroups v2** is the default, which gives a single unified hierarchy and much better memory and I/O accounting per service — relevant if you run containers or need to cap a noisy process.

**Other changes.** Kernel 5.14. Application Streams are simpler — most content ships as ordinary RPMs with parallel-installable versions rather than requiring modules. GCC 11, Python 3.9, and updated language runtimes. Better Cockpit integration, and improved automated system role coverage via RHEL System Roles (Ansible). Podman gained full support for rootless containers, health checks and auto-updates.

#### RHEL 9 to RHEL 10 — image mode, post-quantum crypto, and dropping legacy

RHEL 10 (released 2025) is the newest release, built from CentOS Stream 10.

**Image mode is the flagship feature.** With **bootc / bootable containers**, you define your entire operating system — packages, configuration, everything — in a Containerfile, build it as a container image, and boot a physical or virtual machine directly from that image. Updates become "pull a new image and reboot," and rollback becomes "boot the previous image." It brings immutable, versioned, atomically-updated infrastructure to the OS layer, which is a big deal for fleet consistency and for teams already using container workflows. The traditional package-based install (now called **package mode**) is still fully supported.

**Post-quantum cryptography.** RHEL 10 ships the first supported post-quantum algorithms (such as ML-KEM and ML-DSA) in OpenSSL, GnuTLS, NSS and OpenSSH, so organizations can begin migrating ahead of quantum-capable attacks.

**Legacy removal.** **32-bit x86 packages and the i686 architecture are dropped entirely** — RHEL 10 is 64-bit only, and the x86-64-v3 CPU baseline is required, so very old hardware will not boot it. Older, unmaintained components have been retired.

**Other changes.** Kernel 6.12. `dnf5` is the new package manager, faster with a smaller footprint. Wayland is the only display server for graphical installs (X.org server removed). Deeper AI/ML enablement, including Red Hat Enterprise Linux AI and Lightspeed, a natural-language assistant built into the platform for administration guidance. Improved SELinux performance and stricter default crypto policy. Valkey replaces Redis as the shipped in-memory data store.

#### Quick revision table

| Area | RHEL 6 | RHEL 7 | RHEL 8 | RHEL 9 | RHEL 10 |
|---|---|---|---|---|---|
| Released | 2010 | 2014 | 2019 | 2022 | 2025 |
| Kernel | 2.6.32 | 3.10 | 4.18 | 5.14 | 6.12 |
| Init | SysV / Upstart | systemd | systemd | systemd | systemd |
| Service command | `service`, `chkconfig` | `systemctl` | `systemctl` | `systemctl` | `systemctl` |
| Default filesystem | ext4 (~16 TB) | XFS (500 TB) | XFS | XFS | XFS |
| Package manager | yum | yum | **dnf** (yum → dnf) | dnf | **dnf5** |
| Repo layout | single channel | single channel | **BaseOS + AppStream** | BaseOS + AppStream | BaseOS + AppStream |
| Firewall | iptables | **firewalld** (iptables backend) | firewalld (**nftables** backend) | firewalld / nftables only | firewalld / nftables |
| Network config | `network` service, ifcfg | NetworkManager + ifcfg | NetworkManager (ifcfg deprecated) | **NetworkManager keyfiles only** | NetworkManager keyfiles |
| NIC naming | eth0, eth1 | ens192 / eno1 / enp0s3 | consistent naming | consistent naming | consistent naming |
| Boot loader | GRUB 0.97 | GRUB 2 | GRUB 2 | GRUB 2 | GRUB 2 (+ bootc) |
| Logging | rsyslog | journald + rsyslog | journald + rsyslog | journald + rsyslog | journald + rsyslog |
| Containers | LXC (preview) | Docker | **Podman / Buildah / Skopeo** | Podman (rootless mature) | Podman + **image mode / bootc** |
| cgroups | v1 | v1 | v1 (v2 optional) | **v2 default** | v2 |
| Python | 2.6 | 2.7 | 3.6 (`python3`) | 3.9 | 3.12 |
| Default SSH root login | permitted | permitted | permitted | **denied (key only)** | denied |
| Crypto notes | — | — | system-wide crypto policies | **SHA-1 deprecated**, OpenSSL 3.0, no TLS 1.0/1.1 | **post-quantum algorithms** |
| Auth config tool | `authconfig` | `authconfig` | **authselect** | authselect | authselect |
| Architecture | 32 + 64-bit | 32 + 64-bit | 64-bit | 64-bit | **64-bit only, x86-64-v3** |
| Display server | X.org | X.org | X.org / Wayland | Wayland default | **Wayland only** |
| Upgrade path | — | — | `leapp` from 7 | `leapp` from 8 | `leapp` from 9 |

#### What the interviewer is really listening for

Do not just recite the table. Anchor each version to a problem it solved: systemd solved slow serial boots and unreliable service tracking; XFS solved the scale ceiling of ext4; AppStream solved "I need a newer Python but I cannot change the OS life cycle"; Podman solved "one Docker daemon is a single point of failure and requires root"; RHEL 9's crypto changes solved "SHA-1 and TLS 1.0 are no longer safe"; image mode solves "my servers drift apart over years of patching." Mentioning the migration pain — XFS not shrinking, SHA-1 breaking old certificates, network-scripts disappearing, 32-bit gone in RHEL 10 — is what makes you sound like someone who has actually done upgrades rather than read a comparison chart.

---

### 2. Linux booting process

**BIOS/Legacy sequence:**

1. **BIOS/UEFI POST** — hardware self-test, finds the boot device from the boot order.
2. **MBR / GPT** — first 512 bytes of the disk (446 bytes boot code + 64 bytes partition table + 2 bytes magic). UEFI instead reads the EFI System Partition (`/boot/efi`).
3. **GRUB 2** — stage 1 in MBR loads stage 1.5/2 from `/boot`; reads `/boot/grub2/grub.cfg`, shows the menu, loads the selected `vmlinuz` kernel and the `initramfs` into memory.
4. **Kernel** — decompresses, initializes memory/CPU/scheduler, mounts `initramfs` as a temporary root, loads the drivers needed for the real root (storage, RAID, LVM, multipath), then pivots to the real root filesystem read-only, remounts read-write.
5. **init / systemd (PID 1)** — kernel executes `/sbin/init` → symlink to `systemd`.
6. **systemd targets** — reads `default.target` (usually `multi-user.target` or `graphical.target`), resolves dependencies, starts units in parallel, mounts filesystems from `/etc/fstab`, activates swap, brings up network, starts services.
7. **Login** — getty/display manager presents the login prompt; PAM authenticates.

**Key files:** `/boot/grub2/grub.cfg`, `/etc/default/grub`, `/etc/fstab`, `/etc/systemd/system/default.target`.

**Useful commands:** `systemctl get-default`, `systemctl set-default multi-user.target`, `systemd-analyze blame`, `dmesg`, `journalctl -b`.

---

### 3. Difference between soft link and hard link

| | Soft link (symbolic) | Hard link |
|---|---|---|
| Create | `ln -s target link` | `ln target link` |
| Inode | new inode of its own | same inode as the original |
| Points to | the path name | the data blocks directly |
| Cross filesystem | yes | no, same filesystem only |
| Directories | allowed | not allowed |
| Delete original | link breaks (dangling) | data still accessible via the link |
| Size | size of the path string | same as original file |
| Link count | unchanged | increments the inode link count |

Verify with `ls -li` (inode number and link count) or `stat file`.

---

### 4. Difference between yum and rpm

- **rpm** is the low-level package manager. It installs/queries a single local `.rpm` file (`rpm -ivh pkg.rpm`, `rpm -qa`, `rpm -qf /path/file`, `rpm -e pkg`). It **does not resolve dependencies** — it just reports "failed dependencies".
- **yum** (and `dnf` in RHEL 8+) is the front-end on top of rpm. It works with **repositories** (`/etc/yum.repos.d/*.repo`), **automatically resolves and downloads dependencies**, supports groups, history and rollback (`yum history undo`), `yum update`, `yum search`, `yum provides`.

One-liner: "rpm installs what you hand it; yum figures out what else is needed and fetches it from the repo."

---

### 5. Difference between TCP and UDP

| TCP | UDP |
|---|---|
| Connection-oriented (3-way handshake SYN, SYN-ACK, ACK) | Connectionless |
| Reliable — ACKs, retransmission, sequencing | Unreliable — fire and forget |
| Ordered delivery guaranteed | No ordering |
| Flow control and congestion control | None |
| Higher overhead, 20-byte header | Low overhead, 8-byte header |
| Slower | Faster |
| SSH(22), HTTP(80), HTTPS(443), SMTP(25), FTP(21) | DNS(53), DHCP(67/68), NTP(123), SNMP(161), TFTP(69), syslog(514), streaming/VoIP |

---

### 6. How will you fix if a user is not able to log in to the server?

**SYMPTOM**

The user reports they cannot log in. Before doing anything, pin down which of these it actually is, because each points at a different layer:

- "Connection timed out" or "No route to host" → network, routing or a firewall silently dropping packets.
- "Connection refused" → you reached the host, but nothing is listening on the port, or the firewall is actively rejecting.
- "Permission denied (publickey,password)" → you reached sshd, so this is authentication or authorization.
- "Access denied" after the correct password → account locked, expired, or PAM/access rules blocking.
- Login succeeds then immediately disconnects → shell, home directory, profile script, or a full/read-only filesystem.

Also establish scope: one user or everyone, one server or several, and did it ever work or is this a new account.

**TRIAGE (first 90 seconds)**

```bash
ping <server>                          # is the host up
nc -zv <server> 22                     # is the SSH port reachable
systemctl status sshd                  # is the daemon running
ss -tulnp | grep :22                   # is it listening, on which address
id <user> ; getent passwd <user>       # does the account exist and resolve
passwd -S <user>                       # PS = usable, LK = locked, NP = no password
chage -l <user>                        # is the password or account expired
df -h / /home                          # full filesystem blocks login
mount | grep ' / '                     # is root read-only
tail -30 /var/log/secure               # the actual reason, almost always
```

Ninety seconds of this narrows it to a layer in nearly every case.

**RCA — ranked root-cause possibilities**

1. **Wrong password, or password expired** — by far the most common, especially after a policy change.
2. **Account locked by failed-attempt policy** (`pam_faillock`) — very common after a script or saved credential retried with a stale password.
3. **Account or password expiry** set by `chage`, often hit in bulk when a compliance policy is applied.
4. **Access restriction** — `AllowUsers`/`AllowGroups`/`DenyUsers` in `sshd_config`, or `/etc/security/access.conf`, or the user missing from a required group.
5. **Filesystem full or root mounted read-only** — sshd cannot write session files, login fails for everyone.
6. **Home directory missing, wrong ownership, or on a failed NFS mount** — often after a migration; login succeeds then drops.
7. **Login shell set to `/sbin/nologin` or `/bin/false`** — typical for a service account someone now wants interactive access to.
8. **SSH key problems** — wrong permissions on `~/.ssh` or `authorized_keys`, wrong key, or a bad SELinux context.
9. **Directory service failure** — SSSD/LDAP/AD not reachable, cached credentials expired, or Kerberos failing on clock skew.
10. **Firewall, security group or network ACL change** — usually affects everyone at once and coincides with a change window.
11. **sshd down, or misconfigured after an edit** — the daemon failed to reload.
12. **SELinux denial** — after a restore, a manual `mv` of home directories, or a relabel that never ran.
13. **Resource exhaustion** — `MaxStartups`/`MaxSessions` reached, PID limit hit, or the box is out of memory.

**DIAGNOSIS — exact commands, files and log evidence**

*Server-side log evidence (start here):*

```bash
tail -f /var/log/secure                # RHEL: all auth events
journalctl -u sshd -f                  # follow live while the user retries
journalctl -u sssd -f                  # if the account is AD/LDAP
grep -i "Failed password" /var/log/secure | tail
grep -i "Invalid user" /var/log/secure | tail
lastb | head                           # failed login history
```

Read the message literally, it names the cause: `Failed password for user` = wrong credentials; `Invalid user` = account not found; `User account has expired`; `Authentication failure` with `pam_faillock` = locked out; `Permission denied (publickey)` = key rejected; `User not allowed because listed in DenyUsers`; `Too many authentication failures`.

*Client-side evidence:*

```bash
ssh -vvv user@server                   # shows exactly which auth method is offered and rejected
```

*Account state:*

```bash
getent passwd <user>                   # verify UID, home, shell - and that it resolves at all
passwd -S <user>                       # lock status and aging
chage -l <user>                        # expiry dates
grep '^<user>:' /etc/shadow            # !! or ! prefix = locked, * = no password login
faillock --user <user>                 # failed-attempt counter (pam_tally2 -u on RHEL 6/7)
id <user>                              # group membership for AllowGroups checks
```

*SSH configuration:*

```bash
sshd -T | grep -Ei "permitrootlogin|passwordauth|allowusers|allowgroups|denyusers|maxsessions|maxstartups|usepam"
```

`sshd -T` dumps the **effective** configuration after all includes and matches, which is much more reliable than reading `sshd_config` by eye.

*Home directory, permissions and keys:*

```bash
ls -ld /home/<user>                    # exists, owned by the user, mode 700 or 750
ls -ld /home/<user>/.ssh               # must be 700
ls -l  /home/<user>/.ssh/authorized_keys   # must be 600, owned by the user
ls -Z  /home/<user>/.ssh               # SELinux context should be ssh_home_t
df -h /home ; mount | grep home
```

*PAM, access control and limits:*

```bash
cat /etc/security/access.conf          # + / - rules by user, group, origin
cat /etc/security/limits.conf          # maxlogins, nproc
grep -r faillock /etc/pam.d/ /etc/security/faillock.conf
authselect current                     # RHEL 8+: which PAM profile is active
cat /etc/hosts.allow /etc/hosts.deny   # TCP wrappers, still present on older builds
```

*Network and firewall:*

```bash
firewall-cmd --list-all
iptables -L -n | head -30
ip a ; ip r
nmap -p 22 <server>                    # from the client side
```

*Directory service (AD/LDAP accounts):*

```bash
sssctl domain-status <domain> ; systemctl status sssd
realm list
chronyc tracking                       # Kerberos fails on >5 min clock skew
klist ; kinit <user>
sss_cache -E                           # invalidate a stale cache
```

*SELinux:*

```bash
getenforce
ausearch -m avc -ts recent
sealert -a /var/log/audit/audit.log
```

**TEMP FIX — immediate mitigation**

Restore the user's access first, investigate the cause after.

```bash
faillock --user <user> --reset            # clear a lockout (fastest and most common fix)
passwd -u <user>                          # unlock the account
passwd <user>                             # reset the password
chage -d $(date +%Y-%m-%d) <user>         # clear an expired password
chage -E -1 <user>                        # remove account expiry
usermod -s /bin/bash <user>               # give a usable shell
chown -R <user>:<group> /home/<user> ; chmod 700 /home/<user>
restorecon -Rv /home/<user>/.ssh          # fix SELinux context on keys
systemctl restart sshd                    # only if the daemon itself is the problem
setenforce 0                              # ONLY to confirm SELinux is the cause, then set back
truncate -s 0 /var/log/hugefile.log       # if a full filesystem is blocking logins
```

If the whole server is unreachable, get on the console (iLO/iDRAC/vCenter) and work from there rather than guessing remotely. Always tell the user what you changed and have them confirm before you close the ticket.

**PERMANENT FIX — proper remediation**

- If it was a lockout, find *what* was retrying with the stale password — a cron job, a monitoring probe, an application connection string, a saved credential in a client tool — and fix the source. Otherwise it locks again within the hour.
- If it was expiry, correct the aging policy for the account class rather than clearing it repeatedly; service accounts should have `chage -M 99999` or be managed as non-expiring by policy, and human accounts should get the standard policy.
- If it was access restrictions, manage them through **group membership** (`AllowGroups sshusers`) rather than listing individual users, so onboarding and offboarding do not require editing sshd_config.
- If it was the home directory, fix ownership at the source and make the automount or NFS entry robust (`_netdev`, correct export). If home directories are on NFS, ensure the mount is monitored.
- If it was a full or read-only filesystem, extend the LV properly and fix log rotation instead of truncating again next week.
- If it was SELinux, create the correct file context rule (`semanage fcontext -a`) and relabel — do not leave SELinux permissive.
- If it was the directory service, fix DNS/site configuration, time synchronization with chrony, and configure SSSD credential caching (`cache_credentials = true`) so a brief outage does not lock everyone out.
- If it was a firewall or network change, get the rule added permanently and documented in the change record; test with `firewall-cmd --runtime-to-permanent` only after validating.
- Move to **key-based or centralized authentication** with sudo instead of shared passwords wherever possible.

**PREVENTION**

- Monitor and alert on: sshd down, `/var/log/secure` failed-login spikes, account lockouts, filesystems above 85%, root filesystem going read-only, and SSSD/LDAP availability.
- Alert users proactively before password expiry (`PASS_WARN_AGE`, plus a scheduled report from `chage -l` across all accounts).
- Standardize account provisioning through Ansible or a joiner/mover/leaver process so shell, home, groups and aging are always consistent — most of these incidents come from hand-built accounts.
- Never store passwords in scripts or monitoring configs; use keys or a vault, which eliminates the most common lockout cause.
- Keep an emergency break-glass access path (console access, a documented local admin account, or a serial/IPMI route) and test it, so an authentication outage never becomes a total lockout.
- Document the standard `sshd_config` as code and deploy it centrally, so a one-off manual edit cannot drift a server into blocking users.
- Add "verify login as a normal user" to your post-patch and post-migration checklist.

---

### 7. What is swap memory?

Swap is disk space (a dedicated partition, LVM LV, or a file) used as an extension of RAM. When physical memory is under pressure, the kernel pages out inactive memory pages to swap to free RAM. It's also used to hold the memory image during hibernation.

- Swap is far slower than RAM, so heavy swapping ("thrashing") means you need more memory.
- `swappiness` (`/proc/sys/vm/swappiness`, default 30–60) controls how aggressively the kernel swaps; 0–10 for DB servers.
- Sizing rule of thumb (RHEL): RAM ≤ 2 GB → 2× RAM; 2–8 GB → equal to RAM; 8–64 GB → at least 4 GB (commonly 0.5× RAM); > 64 GB → at least 4 GB.

**Check:** `free -h`, `swapon -s`, `cat /proc/swaps`, `vmstat 1` (si/so columns).

**Create a swap file:**

```bash
dd if=/dev/zero of=/swapfile bs=1M count=2048   # or fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile swap swap defaults 0 0" >> /etc/fstab
```

**Create swap on an LV:** `mkswap /dev/vg01/swaplv` → `swapon` → add UUID entry to `/etc/fstab`.

---

### 8. Fields in /etc/passwd

Seven colon-separated fields:

```
username:x:UID:GID:GECOS:home_directory:login_shell
oracle:x:1001:1001:Oracle DB owner:/home/oracle:/bin/bash
```

1. **Username**
2. **Password placeholder** — `x` means the hash lives in `/etc/shadow`
3. **UID** — 0 = root; system accounts < 1000; users ≥ 1000
4. **GID** — primary group
5. **GECOS/comment** — full name, contact info
6. **Home directory**
7. **Login shell** — `/sbin/nologin` disables interactive login

Permissions: `644 root:root`.

---

### 9. Fields in /etc/shadow

Nine colon-separated fields:

```
username:password_hash:lastchg:min:max:warn:inactive:expire:reserved
```

1. **Username**
2. **Encrypted password** — `$6$` = SHA-512, `$5$` = SHA-256, `$1$` = MD5, `$y$` = yescrypt. `!` or `!!` = locked, `*` = no password login, empty = no password at all.
3. **Last password change** — days since 1 Jan 1970
4. **Minimum days** before the password can be changed again
5. **Maximum days** the password is valid
6. **Warning days** before expiry
7. **Inactive days** — grace period after expiry before the account is disabled
8. **Account expiry date** — days since epoch
9. **Reserved** for future use

Permissions: `000` or `400 root:root` (readable only by root).

---

### 10. How to check password parameters

```bash
chage -l username                  # per-user aging: expiry, min, max, warn
passwd -S username                 # status: PS/LK/NP, last change, min, max, warn, inactive
grep username /etc/shadow

# System-wide defaults for new users
grep -v '^#' /etc/login.defs | grep PASS
# PASS_MAX_DAYS 90 / PASS_MIN_DAYS 7 / PASS_MIN_LEN 8 / PASS_WARN_AGE 7

# Complexity policy
cat /etc/security/pwquality.conf   # minlen, dcredit, ucredit, ocredit, lcredit, minclass
cat /etc/pam.d/system-auth         # pam_pwquality / pam_pwhistory / pam_faillock
authselect current                 # RHEL 8+ PAM profile
```

**Change parameters:**

```bash
chage -M 90 -m 7 -W 7 username     # max, min, warn
chage -E 2026-12-31 username       # account expiry
chage -I 30 username               # inactive days
chage -d 0 username                # force password change at next login
passwd -l / -u username            # lock / unlock
```

---

### 11. How to create an LVM filesystem

LVM stack: **PV → VG → LV → filesystem → mount point.**

```bash
# 1. Identify the new disk
lsblk ; fdisk -l ; cat /proc/scsi/scsi
# for a newly presented SAN LUN: echo "- - -" > /sys/class/scsi_host/hostX/scan

# 2. (Optional) partition and set type 8e / lvm
fdisk /dev/sdb        # n, p, 1, t, 8e, w
partprobe /dev/sdb

# 3. Physical Volume
pvcreate /dev/sdb1
pvs ; pvdisplay

# 4. Volume Group
vgcreate vg_app /dev/sdb1
vgs ; vgdisplay vg_app

# 5. Logical Volume
lvcreate -L 10G -n lv_data vg_app        # fixed size
lvcreate -l 100%FREE -n lv_data vg_app   # all free extents
lvs ; lvdisplay /dev/vg_app/lv_data

# 6. Filesystem
mkfs.xfs /dev/vg_app/lv_data             # or mkfs.ext4

# 7. Mount and persist
mkdir -p /data
blkid /dev/vg_app/lv_data                # get UUID
vi /etc/fstab
#  /dev/mapper/vg_app-lv_data  /data  xfs  defaults  0 0
mount -a
df -hT /data
```

Always add the fstab entry **before** `mount -a` so you validate the entry — a bad fstab is a classic cause of a boot failure.

---

### 12. How to extend an LVM filesystem

**Case A — free space already in the VG:**

```bash
vgs                                        # confirm VFree
lvextend -L +5G /dev/vg_app/lv_data        # or -l +100%FREE
xfs_growfs /data                           # XFS: grow by mount point
resize2fs /dev/vg_app/lv_data              # ext3/ext4: grow by device
df -h /data
```

**Case B — VG is full, add a new disk:**

```bash
pvcreate /dev/sdc
vgextend vg_app /dev/sdc
lvextend -r -L +20G /dev/vg_app/lv_data    # -r resizes the FS automatically
```

**Case C — existing disk was expanded on the storage/hypervisor side:**

```bash
echo 1 > /sys/class/block/sdb/device/rescan
pvresize /dev/sdb1
lvextend -r -l +100%FREE /dev/vg_app/lv_data
```

Extending is **online and safe**. `lvextend -r` (`--resizefs`) handles both XFS and ext4.

---

### 13. How to reduce an LVM filesystem

Only possible for **ext2/3/4**. **XFS cannot be shrunk** — the only path for XFS is backup → `lvreduce` → `mkfs.xfs` → restore.

```bash
# ext4 shrink (take a backup first — risk of data loss)
umount /data                                  # must be offline
e2fsck -f /dev/vg_app/lv_data                 # mandatory check
resize2fs /dev/vg_app/lv_data 8G              # shrink FS FIRST
lvreduce -L 8G /dev/vg_app/lv_data            # then shrink LV to the same size
mount /data
df -h /data
```

Order matters: **shrink the filesystem before the LV.** Reversing it destroys data. Safer one-shot form: `lvreduce -r -L 8G /dev/vg_app/lv_data`.

To free the underlying disk: `pvmove /dev/sdc` → `vgreduce vg_app /dev/sdc` → `pvremove /dev/sdc`.

---

### 14. If an LVM filesystem is unable to mount — causes and fixes

**SYMPTOM**

`mount: /data: can't find UUID=...`, `mount: special device /dev/mapper/vg_app-lv_data does not exist`, `wrong fs type, bad option, bad superblock`, `structure needs cleaning`, or the mount simply missing after a reboot with the server sitting in emergency mode. The application reports missing files or read-only errors. Note the exact wording — it points almost directly at the cause.

**TRIAGE (first 90 seconds)**

```bash
mount -v /data                    # reproduce and read the exact error
dmesg -T | tail -30               # kernel-level truth: I/O errors, superblock, log recovery
journalctl -xe | tail -30
lsblk                             # is the disk even visible
pvs ; vgs ; lvs                   # is the LVM stack intact and active
blkid /dev/mapper/vg_app-lv_data  # does it have a filesystem, and what UUID
findmnt /data                     # already mounted somewhere else?
cat /etc/fstab | grep data        # does the entry match reality
```

**RCA — ranked root-cause possibilities**

1. **Volume group or logical volume not active** — the most common after a reboot or SAN maintenance. `lvs` shows no `a` in the Attr column.
2. **Wrong or stale `/etc/fstab` entry** — a UUID that changed after a `mkfs`, a typo, or the wrong filesystem type. Very common after a rebuild or restore.
3. **Underlying PV or SAN path missing** — LUN unmapped, zoning change, multipath path failure. `pvs` reports "unknown device".
4. **Filesystem corruption or a dirty log** after an unclean shutdown or storage outage.
5. **Device-mapper nodes missing** under `/dev/mapper` even though LVM metadata is fine, usually a udev issue.
6. **No filesystem on the LV at all** — someone created the LV but never ran `mkfs`, or the wrong device was formatted.
7. **Mount point directory does not exist**, or is not empty and is shadowing data.
8. **Duplicate XFS UUID** — a cloned VM or a snapshot presented alongside the original.
9. **Read-only or failing hardware** — the kernel remounted read-only after I/O errors.
10. **Already mounted at another path**, so the second mount looks like a failure.
11. **SELinux context wrong** on the mount point — mounts, but the application is denied.

**DIAGNOSIS — exact commands and evidence**

```bash
# LVM stack, layer by layer
pvs -v ; pvdisplay              # any "unknown device" or missing PV
vgs -v ; vgdisplay vg_app       # VG present, correct PV count
lvs -a -o +devices              # LV state; Attr column: first char type, 5th 'a' = active
vgchange -ay                    # activate everything and watch for errors
ls -l /dev/mapper/              # do the device nodes exist

# Filesystem identity and integrity
blkid                           # UUID, TYPE - compare against fstab
file -s /dev/mapper/vg_app-lv_data
xfs_info /dev/mapper/vg_app-lv_data      # valid XFS?
xfs_repair -n /dev/mapper/vg_app-lv_data # dry-run check, unmounted only
e2fsck -n /dev/mapper/vg_app-lv_data     # dry-run for ext4
dumpe2fs -h /dev/mapper/vg_app-lv_data   # ext superblock state

# Storage path health
multipath -ll                   # all paths active? any 'failed faulty'
lsscsi ; lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,UUID
cat /sys/block/sdb/device/state
smartctl -a /dev/sdb            # media errors
dmesg -T | grep -iE "i/o error|medium error|scsi|multipath|xfs|ext4"

# fstab and systemd mount units
findmnt --verify                # validates fstab and reports problems
systemctl list-units --type=mount --all
systemctl status data.mount     # systemd's view of the failure

# Mount point and SELinux
ls -ld /data ; ls -Zd /data
ausearch -m avc -ts recent
```

Key log locations: `dmesg` / `journalctl -k` for kernel and I/O errors, `journalctl -xe` for the systemd mount unit failure, `/var/log/messages`, and `/etc/lvm/archive/` which holds LVM metadata backups from before every change.

**TEMP FIX — immediate mitigation**

```bash
# Inactive VG/LV - the most common and the quickest fix
vgchange -ay vg_app
lvchange -ay /dev/vg_app/lv_data
mount /data

# Missing device nodes
vgscan --mknodes ; dmsetup mknodes ; udevadm trigger ; udevadm settle

# Stale UUID in fstab - mount by device to restore service now
mount /dev/mapper/vg_app-lv_data /data

# Dirty log / corruption (device must be UNMOUNTED)
xfs_repair /dev/mapper/vg_app-lv_data
# xfs_repair -L   <- zeroes the log, DATA LOSS possible, last resort only
e2fsck -y /dev/mapper/vg_app-lv_data
e2fsck -b 32768 /dev/mapper/vg_app-lv_data   # backup superblock

# Duplicate XFS UUID (cloned VM)
mount -o nouuid /dev/mapper/vg_app-lv_data /data
xfs_admin -U generate /dev/mapper/vg_app-lv_data

# SAN path recovered - rescan and re-activate
for h in /sys/class/scsi_host/host*/scan; do echo "- - -" > $h; done
multipath -r ; pvscan --cache ; vgchange -ay

# Missing mount point
mkdir -p /data && mount /data

# Recover LVM metadata from the automatic archive
vgcfgrestore -l vg_app                        # list available backups
vgcfgrestore -f /etc/lvm/archive/vg_app_00012.vg vg_app
```

Before any repair on data you care about, take a snapshot of the LV or an image copy (`dd if=/dev/mapper/... of=/backup/img bs=1M`) if the situation allows — `xfs_repair -L` in particular is irreversible. Mount read-only first (`mount -o ro`) to check whether the data is readable before attempting a repair.

**PERMANENT FIX**

- Correct `/etc/fstab` to use the **current UUID** and the right filesystem type, then validate with `findmnt --verify` and `mount -a` **before** rebooting. Add `nofail` to non-critical mounts so a single bad entry can never hold the boot hostage, and `_netdev` for network-backed storage.
- If the VG needed manual activation, confirm `lvm2-monitor` and `lvm2-lvmetad`/`lvmpolld` are enabled, ensure the volume group is in the initramfs when it holds root (`rd.lvm.lv=` and `dracut -f`), and check for a filter in `/etc/lvm/lvm.conf` that is excluding the device.
- If a PV was missing, fix the storage side properly — zoning, LUN masking, multipath configuration with the vendor-recommended settings in `/etc/multipath.conf` — rather than repeatedly rescanning. Use `vgreduce --removemissing` only after the disk is confirmed permanently gone and you accept the data loss.
- If corruption came from I/O errors, replace the failing disk or path and restore from backup; a repaired filesystem on dying hardware will corrupt again.
- If the disk was expanded but not picked up, standardize the procedure: rescan, `pvresize`, `lvextend -r`.
- Document the storage layout (PV to VG to LV to mount point mapping) so recovery does not depend on one person's memory.

**PREVENTION**

- Monitor mount points themselves, not just disk space — alert when an expected filesystem is not mounted, and alert on filesystems going read-only.
- Monitor multipath path state and SAN health; a lost path is an early warning rather than an outage if you catch it.
- Always use UUIDs in fstab and always test with `mount -a` plus `findmnt --verify` before the reboot that would otherwise reveal the mistake.
- Use `nofail` on optional mounts as standard policy so no single storage problem drops the server into emergency mode.
- Take LVM configuration backups seriously — `/etc/lvm/backup` and `/etc/lvm/archive` should be included in your system backup so `vgcfgrestore` is available when you need it.
- Test the restore path, not just the backup job. An untested backup is not a recovery plan.
- Add a post-reboot validation step (compare `mount` output against a known-good baseline) to your patching checklist.

---

### 15. If a filesystem is 100% full — how do you troubleshoot?

**SYMPTOM**

A monitoring alert for a filesystem at or near 100%, or user-facing errors: "No space left on device", the application failing to write logs or temp files, a database going read-only or refusing connections, users unable to log in (sshd cannot write session data), cron jobs failing silently, or the root filesystem remounting read-only. If `/` is full the symptoms are broad and confusing, so always check `df` early on any strange server behaviour.

**TRIAGE (first 90 seconds)**

```bash
df -h                       # which filesystem, and how bad
df -i                       # inodes - "No space left" with free space means inode exhaustion
df -h | awk '$5+0 > 85'     # everything above 85%

# where is the space
du -xh /var --max-depth=1 | sort -rh | head -20

# deleted-but-open files (space allocated, invisible to du)
lsof +L1

dmesg -T | tail             # read-only remount or I/O errors
```

If `df` shows full but `du` totals far less, stop and go straight to `lsof +L1` — that is the deleted-open-file case and it needs a different fix.

**RCA — ranked root-cause possibilities**

1. **Application or system logs that are not rotating** — the single most common cause. Either logrotate is not configured for that log, or the app holds the file handle so rotation frees nothing.
2. **A deleted file still held open by a running process** — someone already "cleaned up" by deleting the log, which freed zero space.
3. **A sudden application problem generating huge logs** — a loop writing stack traces, debug logging left enabled after troubleshooting, or an audit rule flooding `/var/log/audit`.
4. **Core dumps** from a repeatedly crashing process, often multi-gigabyte each.
5. **Old kernels and package cache** filling `/boot` or `/var` — `/boot` filling is why kernel updates fail.
6. **Temp files never cleaned** in `/tmp`, `/var/tmp`, or an application's own temp directory.
7. **A backup, export, dump or transfer written to the wrong location** — a database dump landing on `/` instead of the data volume.
8. **Container/image sprawl** in `/var/lib/docker` or `/var/lib/containers`.
9. **Inode exhaustion** from millions of small files — mail queues, session files, cache directories.
10. **Genuine organic growth** — the data really did outgrow the volume, and the answer is capacity, not cleanup.
11. **A snapshot filling up** (LVM snapshot or VM-level), consuming space as the origin changes.

**DIAGNOSIS — exact commands, paths and evidence**

```bash
# Narrow down by directory, staying on one filesystem
du -xh / --max-depth=1 | sort -rh | head -20
du -xh /var --max-depth=2 | sort -rh | head -20
ncdu -x /var                                    # interactive, if available

# Largest individual files
find / -xdev -type f -size +500M -exec ls -lh {} \; 2>/dev/null | sort -k5 -rh
find /var/log -type f -size +100M -exec ls -lh {} \;

# Recently grown files - what is actively writing right now
find /var -xdev -type f -mmin -60 -size +50M -exec ls -lh {} \;

# Deleted but open - the invisible consumer
lsof +L1
lsof -nP | grep -i deleted
ls -l /proc/*/fd 2>/dev/null | grep deleted

# Inodes
df -i
for d in /var/*; do echo "$(find $d -xdev | wc -l) $d"; done | sort -rn | head

# Journald and audit
journalctl --disk-usage
du -sh /var/log/journal /var/log/audit

# Core dumps
coredumpctl list
ls -lh /var/lib/systemd/coredump/
find / -xdev -name "core.*" -size +10M -exec ls -lh {} \; 2>/dev/null
cat /proc/sys/kernel/core_pattern

# Package and kernel bloat
du -sh /var/cache/dnf /var/cache/yum
rpm -qa kernel | sort
df -h /boot

# Containers
podman system df ; docker system df

# LVM snapshots
lvs -o +snap_percent,data_percent

# Historical growth - when did it start
sar -f /var/log/sa/sa$(date +%d) | head
```

Key files and directories to check in order: `/var/log`, `/var/log/audit`, `/var/log/journal`, `/tmp`, `/var/tmp`, `/var/cache`, `/var/crash`, `/var/lib/systemd/coredump`, `/home`, application log and trace directories, and `/` itself for files written outside any mount point.

**TEMP FIX — immediate mitigation**

```bash
# Live log that must keep being written - truncate, never delete
truncate -s 0 /var/log/app/huge.log
: > /var/log/app/huge.log

# Deleted-but-open file: reclaim without restarting, if you must
> /proc/<pid>/fd/<fd_number>
# or cleanly:
systemctl reload <service>      # many daemons reopen their logs on reload
systemctl restart <service>     # releases the handle for certain

# Compress rather than delete, so evidence survives
gzip /var/log/app/old.log
xz -T0 /var/log/app/big.log

# Force log rotation now
logrotate -f /etc/logrotate.d/<app>

# Trim the journal
journalctl --vacuum-size=200M
journalctl --vacuum-time=7d

# Clear safe caches and temp
dnf clean all ; yum clean all
systemd-tmpfiles --clean
rm -rf /var/cache/dnf/*

# Remove old kernels (keep at least the running one plus one)
dnf remove $(dnf repoquery --installonly --latest-limit=-2 -q)
package-cleanup --oldkernels --count=2       # RHEL 7

# Core dumps
coredumpctl --vacuum-size=0
rm -f /var/lib/systemd/coredump/*

# Containers
podman system prune -a ; docker system prune -a

# Emergency headroom on an LVM volume
lvextend -r -L +5G /dev/vg_app/lv_data
```

Rules that matter here: **never `rm` a log the application is still writing to** — truncate it instead, or you free nothing and may break the writer. **Never delete files you do not own** without the application owner's confirmation, and never delete anything with an unclear purpose. Move suspicious large files aside rather than deleting them if you have anywhere to put them. If the filesystem is genuinely all legitimate data, the correct immediate action is extending the volume, not cleaning.

**PERMANENT FIX**

- Configure or fix **logrotate** for the offending log: appropriate `daily`/`size` trigger, `rotate` count, `compress`, and `copytruncate` for applications that will not release the file handle, or a `postrotate` block that signals the daemon to reopen. Test with `logrotate -d`.
- Turn off debug-level logging that was left enabled, and get the application team to fix whatever is generating the flood.
- Fix the root cause of core dumps rather than deleting them repeatedly, and cap them with `systemd-coredump` settings in `/etc/systemd/coredump.conf` (`MaxUse=`, `KeepFree=`).
- Cap the journal permanently with `SystemMaxUse=` in `/etc/systemd/journald.conf`.
- Set `installonly_limit=3` in `/etc/dnf/dnf.conf` so old kernels are cleaned automatically and `/boot` never fills.
- Configure `systemd-tmpfiles` policies for temp directories so cleanup is automatic.
- If growth is legitimate, **extend the LV properly through a change request** and right-size the volume with headroom for the observed growth rate — do not leave it at 92% and call it fixed.
- Separate volumes so a runaway log cannot take down the OS: `/var`, `/var/log`, `/tmp`, `/home` and application data should be their own filesystems, never all on `/`.
- For inode exhaustion, change the application's file-handling pattern (batch, archive, or use subdirectory sharding), or rebuild the filesystem as XFS which allocates inodes dynamically.

**PREVENTION**

- Alert at **80% warning and 90% critical**, not at 100%. For XFS, treat above 90% as urgent because performance degrades and writes can fail from fragmentation well before it reports full.
- Add a **rate-of-growth alert**, not just a threshold — a filesystem gaining 10% in an hour is more urgent than one sitting at 85% for months.
- Include inode usage (`df -i`) in monitoring; almost nobody does, and it produces very confusing incidents.
- Make logrotate configuration part of every application onboarding checklist and deploy it as code with Ansible.
- Trend capacity monthly and forecast, so extensions happen in planned change windows rather than at 2 a.m.
- Keep a small amount of reserved space or a spare LV in the volume group so you always have an immediate escape route.
- Automate the routine cleanup (journal caps, tmpfiles, kernel retention) rather than relying on someone remembering.
- Document a runbook for the on-call engineer with the exact safe commands, so the fix is never an improvised `rm -rf`.

---

### 16. If RAM or CPU crosses the threshold — how do you troubleshoot?

**SYMPTOM**

A monitoring alert for CPU or memory above threshold, and usually a complaint alongside it: the application is slow or timing out, batch jobs are overrunning their window, logins take a long time, or a process was killed unexpectedly. Establish immediately whether this is a **spike** (short burst, self-correcting) or **sustained saturation** (still climbing), and whether it started at a specific time that lines up with a change, a deployment, or a scheduled job.

**TRIAGE (first 90 seconds)**

```bash
uptime                                   # load average vs nproc; is 1-min > 15-min (rising)?
nproc                                    # how many CPUs you are comparing against
free -h                                  # look at AVAILABLE, not free
top -b -n1 | head -20                    # top consumers right now
ps -eo pid,user,%cpu,%mem,etime,cmd --sort=-%cpu | head -10
ps -eo pid,user,%cpu,%mem,etime,cmd --sort=-%mem | head -10
vmstat 1 5                               # r, b, si/so, wa, st columns
dmesg -T | tail -20                      # OOM kills, hardware errors
```

Interpret `vmstat` immediately: **r** greater than the CPU count means real CPU saturation; **b** high with **wa** high means you are blocked on I/O, not CPU; **si/so** non-zero means active swapping; **st** non-zero means the hypervisor is stealing CPU and the problem is not inside this VM at all.

**RCA — ranked root-cause possibilities**

*For CPU:*

1. **A single runaway application process** — an infinite loop, a stuck thread pool, a bad query plan burning CPU.
2. **A scheduled job overlapping production load** — backup, antivirus scan, patch job, index rebuild, log compression, or a cron job that now runs longer than its window and collides with the next run.
3. **Legitimate load growth** — more users or transactions than the box was sized for.
4. **High `%iowait` misread as a CPU problem** — the CPU is idle and waiting on storage; fixing CPU will do nothing.
5. **High `%steal` on a virtual machine** — the hypervisor is oversubscribed; this is a virtualization-team issue.
6. **High `%sys` / interrupt load** — network flood, driver problem, excessive context switching, or a chatty syscall pattern.
7. **Swap thrashing showing up as CPU load** — memory pressure causing constant paging.
8. **A fork bomb or process leak** — thousands of processes or threads spawned.

*For memory:*

1. **Application memory leak** — usage grows steadily and never returns after restart.
2. **Misconfigured application memory** — JVM heap or database buffer pool sized larger than physical RAM, connection pools too large, too many worker processes.
3. **Too many concurrent processes/sessions** — each modest, but multiplied.
4. **Filesystem cache misread as a problem** — Linux uses free RAM for cache by design; this is healthy and reclaimable.
5. **Insufficient or disabled swap** turning a survivable pressure event into an OOM kill.
6. **Kernel-side growth** — slab/dentry cache growth, or a leaking kernel module.
7. **Undersized VM** — the workload genuinely needs more RAM.
8. **A backup or data-processing job holding large buffers.**

**DIAGNOSIS — exact commands and evidence**

*CPU:*

```bash
top                                      # press 1 for per-core, P to sort by CPU, H for threads
htop
mpstat -P ALL 2 5                        # per-CPU breakdown; is one core pinned or all of them
pidstat -u 2 5                           # per-process CPU over time
pidstat -t -p <pid> 2 5                  # per-thread, for JVM/multithreaded apps
ps -eLo pid,tid,pcpu,comm --sort=-pcpu | head -20   # busiest threads
iostat -xz 2 5                           # %util, await - confirm or rule out I/O
perf top                                 # which functions are burning CPU (kernel or app)
strace -c -p <pid>                       # syscall count, for high %sys
cat /proc/<pid>/status ; cat /proc/<pid>/stack
```

*Memory:*

```bash
free -h ; cat /proc/meminfo
vmstat 1 10                              # si/so = swap in/out activity
ps aux --sort=-rss | head -20            # RSS is the real resident footprint
smem -rs uss -k | head -20               # USS excludes shared pages - most accurate per process
pmap -x <pid> | tail -3                  # memory map of one process
cat /proc/<pid>/status | grep -E "VmRSS|VmSwap|Threads"
slabtop -o | head -20                    # kernel slab consumers
cat /proc/meminfo | grep -E "Slab|SReclaimable|Committed_AS"
swapon -s
dmesg -T | grep -iE "out of memory|killed process|oom"
coredumpctl list
```

*Historical evidence — this is what proves when and what:*

```bash
sar -u -f /var/log/sa/sa$(date +%d)          # CPU by 10-min interval
sar -r -f /var/log/sa/sa$(date +%d)          # memory
sar -B                                       # paging
sar -q                                       # load average and run queue
sar -n DEV                                   # network
sar -W                                       # swapping
sar -u -s 02:00:00 -e 04:00:00 -f /var/log/sa/sa14   # a specific window
```

Also check: the monitoring platform's graphs to see the shape of the curve, `/var/log/messages` and `journalctl` around the start time, `crontab -l` and `/etc/cron.d/` for a job that matches the timing, `rpm -qa --last` and `yum history` for a recent change, and the application's own logs (GC logs for Java, slow query log for databases).

Correlate the start time with something. "It began at 02:15 every night" is an answer; "CPU is high" is not.

**TEMP FIX — immediate mitigation**

```bash
# Confirm with the application owner BEFORE touching a production process
renice -n 19 -p <pid>                    # deprioritize a batch job
ionice -c2 -n7 -p <pid>                  # deprioritize its I/O
kill -TERM <pid>                         # graceful stop of a runaway
kill -STOP <pid> / kill -CONT <pid>      # pause and resume instead of killing
systemctl restart <service>              # clears a leak temporarily

# Contain rather than kill, using cgroups
systemctl set-property <service> CPUQuota=200% MemoryMax=8G

# Memory relief
swapon /swapfile                         # add emergency swap
sync; echo 3 > /proc/sys/vm/drop_caches  # rarely needed - cache is NOT a leak
```

Two cautions to state explicitly in an interview. First, **do not clear caches to "fix" memory** — buffers and cache are reclaimable by design, and dropping them just slows the system down while it re-reads from disk; judge memory by `available` and by swap activity. Second, **never kill a database or application process without the owner's approval** — a hard kill can force a lengthy recovery and turn a performance problem into an outage. If the cause is `%steal`, there is nothing to fix inside the VM; escalate to the virtualization team with the `mpstat` evidence.

**PERMANENT FIX**

- **Fix the application**, which is where most of these end up: correct the memory leak, tune the JVM heap and garbage collector, right-size the database buffer pool and connection limits, cap worker/thread pools, add the missing index behind the CPU-burning query.
- **Reschedule conflicting jobs** so backups, antivirus scans, patching and reports do not overlap peak load or each other; add staggering and a maximum runtime with alerting.
- **Right-size the server** through a change request if the demand is legitimate — more vCPU or RAM, or scale out behind a load balancer rather than continuing to scale up.
- **Set resource limits properly** using systemd slices or cgroups so one service can never starve the rest of the host, and set `LimitNOFILE`/`TasksMax` in the unit rather than relying on `limits.conf`.
- **Configure swap correctly** and tune `vm.swappiness` for the workload (low, around 10, for databases) so the kernel prefers reclaiming cache over paging out the application.
- **Protect critical processes** from the OOM killer with `OOMScoreAdjust`, but treat that as a safety net, not a fix.
- If the cause was hypervisor contention, get the host's oversubscription ratio corrected, reservations set, or the VM migrated to a less contended host.
- If it was hardware (correctable memory errors causing CPU overhead, a failing disk causing I/O wait), replace the component and update firmware.

**PREVENTION**

- Alert on **sustained** breaches with a duration condition (for example, above 90% for 15 minutes) rather than instantaneous spikes, so alerts stay meaningful and are not ignored.
- Alert on **trend and saturation signals**, not just utilization: run-queue length, swap-in/out rate, `%steal`, and memory `available` — these predict trouble before utilization looks bad.
- Make sure **sysstat/sar is installed and collecting** on every server, with a sensible retention period. Without history you cannot prove when a problem started or whether it is growing, and this is the single most useful preventive step.
- Establish a **performance baseline** per server so you know what normal looks like at each hour of the day and each day of the month.
- Capacity-plan on trends: review growth monthly and resize in planned windows.
- Put resource limits on non-critical workloads by default so they cannot affect production services on shared hosts.
- Add memory and CPU behaviour to the acceptance criteria for application releases, including a soak test long enough to expose leaks.
- Maintain a runbook naming the expected top consumers on each server, so on-call can tell abnormal from normal in seconds.

---

### 17. If the server hangs, how will you fix it?

**SYMPTOM**

The server does not respond to SSH, or an existing session accepts keystrokes but nothing executes. It may still answer ping (network stack alive, userspace stuck) or not respond at all. The application is unreachable, monitoring shows the host down or shows metrics that stopped abruptly at a specific timestamp — that timestamp is your most valuable clue. Distinguish three different situations, because they need different handling: **completely frozen** (no console response), **partially responsive** (console works, commands hang), and **just extremely slow** (everything eventually completes, which is a performance problem, not a hang).

**TRIAGE (first 90 seconds)**

```bash
ping <server>                            # network stack alive?
nc -zv <server> 22
ssh -v <server>                          # where does it stall - TCP, banner, or after auth
```

If SSH fails, go straight to out-of-band console: **iLO, iDRAC, vCenter/VMware console, KVM, or IPMI serial-over-LAN**. Do not spend time retrying SSH. The console is also where the panic or error messages will be visible.

If you get a console prompt, work fast and in this order:

```bash
uptime                                   # load average - is it enormous
free -h                                  # any memory left
df -h                                    # is / full
mount | grep ' / '                       # is / read-only
dmesg -T | tail -50                      # the single most informative command
ps -eo pid,stat,wchan:25,cmd | awk '$2 ~ /D/'   # D-state processes and what they block on
```

**RCA — ranked root-cause possibilities**

1. **Memory exhaustion / swap thrashing** — the box is alive but spending all its time paging; often accompanied by OOM kills. This is the most common cause of an apparent hang.
2. **Hung storage** — a failed SAN path, an unresponsive NFS server, or a dying local disk. Processes pile up in D state, load average climbs into the hundreds while CPU sits idle.
3. **Full or read-only root filesystem** — nothing can write, so almost everything stalls, including login.
4. **A runaway process consuming all CPU or forking uncontrollably** — a fork bomb or process leak exhausting the PID/task limit so no new command can start.
5. **Kernel or driver bug** — a soft lockup, hung task, or deadlock. `dmesg` shows "BUG: soft lockup", "task blocked for more than 120 seconds", or a call trace.
6. **Hardware fault** — failing DIMM with correctable-error storms, CPU fault, failing disk, or a thermal event. The iLO/iDRAC system event log is the authority here.
7. **File descriptor or resource limit exhaustion** — cannot open files or sockets, so services stall.
8. **Hypervisor-level problem** — the underlying host is overloaded, its storage is unavailable, or the VM is stunned during a snapshot operation. The guest looks hung but is a victim.
9. **Network-layer problem only** — the server is fine and the network path is broken; the console will prove this in seconds.

**DIAGNOSIS — exact commands, logs and evidence**

*From a responsive console:*

```bash
dmesg -T | tail -80
dmesg -T | grep -iE "soft lockup|hung task|blocked for more than|oom|i/o error|nfs.*not responding|call trace"
journalctl -xe --no-pager | tail -60
journalctl -k -b
vmstat 1 5                               # r, b, si/so, wa
top -b -n1 | head -25
ps -eo pid,ppid,stat,wchan:30,etime,cmd | awk '$3 ~ /D/'     # blocked tasks
cat /proc/loadavg ; nproc
ps -e | wc -l ; cat /proc/sys/kernel/pid_max               # process/PID exhaustion
lsof | wc -l ; sysctl fs.file-nr                           # fd exhaustion
mount | grep nfs ; showmount -e <nfs-server>               # NFS reachable
multipath -ll                                              # SAN path state
df -h ; df -i
```

*If commands themselves hang:* that in itself is diagnostic. A `df` that hangs means a filesystem is unresponsive (usually NFS); an `ls` that hangs in a specific directory identifies the mount. Use `timeout 5 df -h` so a hung filesystem does not trap your session, and `cat /proc/mounts` instead of `mount` since it does not touch the filesystems.

*Out-of-band and hypervisor evidence:*

- iLO / iDRAC / IMM: **System Event Log** and **Integrated Management Log** for memory, CPU, power, thermal and disk faults. This is the fastest way to prove a hardware cause.
- vCenter: VM events and tasks, host CPU-ready and storage-latency metrics, snapshot operations, datastore all-paths-down events.
- Storage array and SAN switch logs for the same timestamp.

*After recovery, for root cause:*

```bash
sar -q -f /var/log/sa/sa<DD>            # load average leading up to the hang
sar -r -B -W -f /var/log/sa/sa<DD>      # memory and paging trend
sar -d -p -f /var/log/sa/sa<DD>         # disk service times
last reboot ; who -b
journalctl -b -1 -p err                 # errors from the boot that hung
ls -l /var/crash/                       # kdump vmcore, if configured
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/<dir>/vmcore
sosreport                               # bundle for a Red Hat/vendor case
```

**TEMP FIX — immediate mitigation**

Capture evidence **before** you reboot, because a reboot destroys the only chance of root cause.

```bash
# Enable and use SysRq from the console (magic keys go straight to the kernel)
echo 1 > /proc/sys/kernel/sysrq
# Then from the physical/virtual console keyboard:
#   Alt+SysRq+t   dump all task states to the console/log
#   Alt+SysRq+w   dump blocked (D-state) tasks
#   Alt+SysRq+m   dump memory information
#   Alt+SysRq+l   dump a backtrace of all CPUs
#   Alt+SysRq+c   deliberately crash and trigger kdump (best evidence, if kdump is set up)
#   Alt+SysRq+s   sync disks
#   Alt+SysRq+u   remount all filesystems read-only
#   Alt+SysRq+b   reboot immediately
# The safe shutdown sequence to minimise damage is: s, u, b  (sync, unmount, boot)
```

Then try graceful recovery in increasing order of disruption:

```bash
kill -TERM <runaway_pid>                 # stop the offending process
umount -l -f /hung_nfs_mount             # release a hung NFS mount (lazy + force)
systemctl restart <stuck_service>
swapoff -a && swapon -a                  # clear a thrashing swap situation (only if RAM allows)
systemctl isolate multi-user.target      # drop the GUI/extra services
reboot                                   # or: systemctl reboot
reboot -f                                # skip service shutdown, if reboot itself hangs
```

Only when nothing responds, do a hard reset: iLO/iDRAC power control, `virsh reset <vm>`, or the vCenter reset action. Note in the ticket that it was a hard reset, because it means filesystems were not cleanly unmounted and you should expect log replay (and check filesystems) on the way back up.

**PERMANENT FIX**

- **If memory:** fix the leak or misconfiguration, add RAM, configure swap correctly, and cap the offending service with `MemoryMax=` so it can never take the host down again.
- **If storage:** fix the SAN path, multipath configuration and timeouts to the vendor's recommended values; for NFS, use `soft` with sensible `timeo`/`retrans` where the application can tolerate errors, add `_netdev`, and make the NFS server highly available.
- **If a kernel or driver bug:** update to the kernel version with the fix, update HBA/NIC firmware and drivers, and open a vendor case with the vmcore. This is exactly why kdump matters.
- **If hardware:** replace the failing DIMM, disk or controller and update BIOS/firmware. Do not accept "it has been fine since the reboot" when the SEL shows correctable-error storms.
- **If process/resource exhaustion:** set `TasksMax`, `LimitNOFILE` and cgroup limits per service, and fix the application's leak.
- **If the hypervisor:** address host oversubscription, storage latency, or snapshot practices with the virtualization team.
- **Configure kdump properly on every server** (`systemctl enable --now kdump`, adequate `crashkernel=` reservation, `/var/crash` with space) and enable `kernel.sysrq`. If a hang recurs with no vmcore, you will be no better off the second time.
- Consider the kernel watchdog (`kernel.hung_task_timeout_secs`, `nmi_watchdog`, `kernel.panic_on_oops=1` with `kernel.panic=30`) so an unrecoverable state auto-reboots and captures a dump instead of sitting dead until someone notices.

**PREVENTION**

- **Enable kdump and persistent journald everywhere** as a build standard — this is the single highest-value preventive action for hangs, because it converts an unexplainable outage into a diagnosable one.
- Alert on the **precursors**, not the hang: swap-in/out rate, load average relative to core count, D-state process count, memory `available`, filesystem above 90%, root read-only, and multipath path failures.
- Monitor hardware out-of-band — feed iLO/iDRAC SNMP traps into monitoring so correctable memory errors and predictive disk failures are tickets, not surprises.
- Keep firmware, drivers and kernels on a supported, patched baseline; a large share of hangs are known bugs already fixed upstream.
- Ensure `sysstat` history is collecting so you can reconstruct the hour before the hang.
- Verify out-of-band console access works and credentials are current **before** you need it at 3 a.m. Test it during maintenance windows.
- Keep the OS on its own volumes with `/var`, `/var/log` and `/tmp` separated so a runaway writer cannot freeze the root filesystem.
- Run a post-incident review for every hang and track it to a root cause; repeated "rebooted and it was fine" entries are a sign of an unmanaged fault.

---

### 18. If the server goes into kernel panic mode, how will you fix it?

**SYMPTOM**

The console shows a kernel panic and the system stops dead — no SSH, no logging, nothing written to disk. The message text is the diagnosis, so read it and screenshot it before anything else:

- `Kernel panic - not syncing: VFS: Unable to mount root fs on unknown-block(0,0)` → the kernel cannot find or mount the root filesystem. Usually a broken initramfs, a missing storage driver, or a wrong `root=` parameter.
- `Kernel panic - not syncing: Attempted to kill init!` → init/systemd died, often a corrupted root filesystem or missing critical libraries.
- `Fatal exception in interrupt` / `BUG: unable to handle kernel NULL pointer dereference` with a call trace → a kernel or driver defect.
- `Machine check exception` / `Hardware Error` → genuine hardware, usually CPU or memory.
- `dracut-initqueue timeout` dropping to an emergency shell → the root device never appeared.

The most important context question: **did this start right after a kernel update, a patch, a hardware change, or a storage change?** Post-patch panics are the most common and the easiest to fix.

**TRIAGE (first 90 seconds)**

You have no shell, so triage is observational and done from the console.

1. Read and photograph the full panic output, including the call trace and the last few lines before it.
2. Note whether the panic happens **immediately at boot** (before any services) or **later in the boot**. Early means root device or initramfs; later means filesystem, driver or userspace.
3. Reboot and watch the GRUB menu: is there a **previous kernel** entry available? That is your fastest recovery.
4. Check whether the server is physical or virtual, and whether other servers patched in the same batch are also affected — a pattern means a bad kernel or a storage change, not a one-off.
5. Check the change record for what happened in the last 24 hours.

**RCA — ranked root-cause possibilities**

1. **Bad or incomplete kernel update** — the new kernel's initramfs was built without the storage driver, or `/boot` was full so the initramfs is truncated. Very common.
2. **Missing storage driver in initramfs** — multipath, HBA, RAID controller or virtio driver not included, so the root device is invisible.
3. **Wrong kernel command line** — `root=UUID=` or `rd.lvm.lv=` no longer matches reality after a restore, clone, or filesystem recreation.
4. **Root device genuinely unavailable** — SAN LUN unmapped, zoning changed, multipath misconfigured, boot disk failed.
5. **Corrupted root filesystem** after an unclean shutdown or storage outage.
6. **Bad `/etc/fstab`** — in some cases this panics rather than dropping to emergency mode, particularly if root itself is involved.
7. **Faulty hardware** — uncorrectable ECC memory errors, failing CPU, or a failing boot disk. Machine check exceptions point here.
8. **Kernel or driver bug** triggered by a specific workload or a newly loaded module.
9. **Incompatible or unsigned third-party module** (backup agent, antivirus, monitoring, GPU driver) built against a different kernel.
10. **Full `/boot`** — worth calling out separately because it causes several of the above at once.

**DIAGNOSIS — how to get evidence with no running system**

*The message itself:* the call trace names the subsystem. Anything mentioning `xfs`, `ext4`, `dm_`, `scsi`, `multipath`, `qla`/`lpfc` (HBAs), or `virtio` points at storage; a vendor module name points at third-party software.

*Boot the previous kernel and investigate from a working system:*

```bash
uname -r                                        # confirm which kernel you are on
rpm -qa kernel | sort                           # what is installed
yum history ; rpm -qa --last | head -20         # what changed and when
df -h /boot                                     # was /boot full when the initramfs was built
ls -lh /boot/initramfs-*                        # is one suspiciously small or missing
lsinitrd /boot/initramfs-<bad-version>.img | grep -iE "multipath|qla|lpfc|megaraid|virtio"
grubby --info=ALL                                # kernel command line for each entry
cat /proc/cmdline                                # the working kernel's parameters, to compare
blkid ; lsblk ; vgs ; lvs                        # does root=UUID match reality
journalctl -b -1 -p err                          # errors from the failed boot, if persistent journal is on
ls -l /var/crash/                                # vmcore, if kdump was configured
```

*Analyze a vmcore if kdump captured one:*

```bash
crash /usr/lib/debug/lib/modules/$(uname -r)/vmlinux /var/crash/<timestamp>/vmcore
# then inside crash:  bt   log   ps   kmem -i   sys
```

*If you cannot boot at all, use rescue media:*

```bash
# Boot the RHEL ISO -> Troubleshooting -> Rescue a Red Hat system -> option 1 (mount under /mnt/sysimage)
chroot /mnt/sysimage          # RHEL 7   (use /mnt/sysroot on RHEL 8/9/10)
cat /etc/fstab ; blkid
journalctl --directory=/var/log/journal -b -1
rpm -qa kernel
df -h /boot
```

*Hardware evidence:* iLO / iDRAC / IMM **System Event Log** for uncorrectable memory errors, CPU faults and thermal events; vendor diagnostics (HPE SUM, Dell SupportAssist); `dmidecode -t memory` and `edac-util` once booted. For virtual machines, check the hypervisor's event log and datastore health for the same timestamp.

**TEMP FIX — restore service now**

```bash
# 1. FASTEST: boot the previous working kernel from the GRUB menu (select it and press Enter)
#    Make it stick for now:
grubby --set-default=/boot/vmlinuz-<known-good-version>

# 2. If the panic is dracut/initramfs related, rebuild the initramfs for the bad kernel
#    (boot the good kernel first, or chroot from rescue media)
dracut -f /boot/initramfs-<bad-version>.img <bad-version>
dracut -f --regenerate-all                       # rebuild all of them
lsinitrd /boot/initramfs-<bad-version>.img | grep -i multipath   # verify the driver is in

# 3. Force required drivers into the initramfs
dracut -f --add-drivers "qla2xxx lpfc dm-multipath" /boot/initramfs-$(uname -r).img $(uname -r)
echo 'add_dracutmodules+=" multipath "' > /etc/dracut.conf.d/multipath.conf

# 4. Fix the kernel command line
grubby --update-kernel=ALL --args="root=UUID=<correct-uuid> rd.lvm.lv=rhel/root rd.lvm.lv=rhel/swap"
grubby --update-kernel=ALL --remove-args="rhgb quiet"    # so you can SEE the messages
grub2-mkconfig -o /boot/grub2/grub.cfg                   # BIOS
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg          # UEFI

# 5. Repair a corrupted root filesystem (from rescue media, UNMOUNTED)
xfs_repair /dev/mapper/rhel-root
fsck -y /dev/mapper/rhel-root

# 6. Fix or neutralize a bad fstab entry (from rescue chroot)
vi /etc/fstab                                     # comment out the bad line or add nofail

# 7. Remove the bad kernel or a broken third-party module
rpm -e kernel-<bad-version>
yum history undo <transaction-id>
rmmod / blacklist the offending module in /etc/modprobe.d/blacklist.conf

# 8. Free space in /boot then re-run the kernel install
dnf remove $(dnf repoquery --installonly --latest-limit=-2 -q)

# 9. Boot with minimal risk while you investigate
#    Add to the kernel line at the GRUB prompt (press 'e'):
#      single            -> minimal single-user boot
#      systemd.unit=rescue.target
#      nomodeset         -> if a graphics driver is panicking
#      selinux=0         -> rule out SELinux
#      panic=30          -> auto-reboot instead of sitting dead (for a recurring random panic)
```

For a virtual machine, reverting the pre-patch snapshot is often the fastest and cleanest recovery — mention that, because interviewers want to hear that you consider rollback rather than heroically debugging a production outage.

**PERMANENT FIX**

- **Get the working kernel supported, not just selected.** If a specific kernel is defective, install the fixed version from Red Hat rather than leaving the server permanently pinned to an old kernel that will fall out of security support.
- **Fix the initramfs build process:** make sure `/boot` has adequate space (1 GB or more), set `installonly_limit=3` so it stays that way, and put required storage modules in `/etc/dracut.conf.d/` so every future kernel includes them automatically.
- **Correct and standardize the kernel command line** with `grubby --update-kernel=ALL` so new kernels inherit the right parameters.
- **Fix the storage layer properly** — multipath configuration per the vendor guide, correct zoning and LUN masking, redundant paths — so root storage cannot vanish.
- **Replace faulty hardware** and update BIOS, firmware and drivers to a supported baseline. Do not return a server to production on a DIMM that produced an uncorrectable error.
- **Rebuild third-party kernel modules** against the new kernel before patching (DKMS or a vendor-supported package), and add that step to the patch procedure — an out-of-tree agent module is a very common panic source.
- **Enable kdump on every server** with a sufficient `crashkernel=` reservation and space in `/var/crash`, so the *next* panic produces a vmcore instead of a mystery.
- **Open a vendor case** with the vmcore and `sosreport` for anything that looks like a kernel defect, so the fix lands upstream rather than recurring.

**PREVENTION**

- Always **patch dev, then QA, then production**, with enough soak time in each. A defective kernel or a broken third-party module then panics a test box, not a production one.
- Take a **snapshot or verified backup before every patch cycle**, and keep the previous kernel installed so rollback is always one GRUB selection away.
- Add pre-patch checks to the runbook: free space in `/boot` and `/`, confirmation that third-party modules have compatible versions, and a note of the current kernel.
- Add post-patch validation: `uname -r`, `systemctl --failed`, all filesystems mounted, application health — and reboot during the window so a boot failure is discovered while you are watching, not weeks later during an unrelated reboot.
- **Remove `rhgb quiet` from the standard build** so boot messages are visible; diagnosing a panic is far harder behind a splash screen.
- Enable kdump, persistent journald, and out-of-band console access as build standards, and verify them periodically.
- Monitor hardware health out-of-band so memory and disk faults are caught as predictive alerts before they panic the kernel.
- Keep a documented, tested recovery procedure (rescue media location, chroot steps, GRUB repair) so recovery does not depend on improvisation under pressure.

---

### 19. How to recover a corrupted GRUB

**SYMPTOM**

The server never reaches the kernel. What you see on the console tells you how far it got:

- `grub rescue>` prompt → GRUB stage 1 loaded but cannot find its modules or configuration.
- `grub>` prompt (no menu) → GRUB loaded but `grub.cfg` is missing or unreadable.
- `error: no such partition` / `error: unknown filesystem` → the partition or `/boot` that GRUB expects is gone, renumbered, or corrupted.
- `error: file '/grub2/i386-pc/normal.mod' not found` → GRUB modules in `/boot` are missing.
- **No bootable device / Operating system not found** → the MBR or EFI boot entry is gone, or the BIOS boot order changed.
- Blank screen or an immediate reboot loop after POST.

Establish what happened just before: a disk clone or restore, an added or removed disk changing device order, a partitioning change, a failed kernel update, a `dd` to the wrong device, a Windows install overwriting the MBR, or a firmware/BIOS change that reset the boot order.

**TRIAGE (first 90 seconds)**

1. Read the exact error string — it distinguishes "MBR gone" from "grub.cfg missing" from "/boot corrupted", and each has a different fix.
2. Check the firmware boot order and whether the boot disk is even detected — in BIOS/UEFI setup or via iLO/iDRAC. A surprising number of "GRUB corrupted" cases are a boot-order change or a disk that dropped offline.
3. Determine **BIOS/legacy or UEFI**, because the repair procedure differs significantly. In UEFI setup you will see boot entries like "Red Hat Enterprise Linux"; the presence of an EFI System Partition is the giveaway.
4. If you get a `grub>` or `grub rescue>` prompt, use it to look around immediately — you may be able to boot manually within a minute.
5. Confirm you have rescue media (matching RHEL major version ISO) and console access before planning the repair.

**RCA — ranked root-cause possibilities**

1. **MBR / boot sector overwritten** — another OS installation, a cloning tool, a misdirected `dd`, or a bootloader written to the wrong disk.
2. **`/boot` contents missing or corrupted** — a full `/boot` during a kernel update, an interrupted update, or filesystem corruption.
3. **`grub.cfg` missing, empty or invalid** — hand-edited, or `grub2-mkconfig` run with a broken `/etc/default/grub`.
4. **Disk order or device naming changed** — a disk added, removed or re-presented, so GRUB's saved `(hd0,msdos1)` no longer points where it should.
5. **UEFI boot entry lost** — NVRAM cleared, battery replaced, firmware updated or reset, or the EFI System Partition not mounted when a GRUB update ran.
6. **Failing boot disk** — bad sectors precisely where the bootloader lives.
7. **A restore or clone** where `/boot`, the fstab UUIDs and the bootloader no longer agree.
8. **Wrong `grub2-install` target** — installed to a partition or a mirror member rather than the boot disk, or run on a UEFI system where it is not appropriate.

**DIAGNOSIS**

*From the `grub>` or `grub rescue>` prompt — inspect and confirm what GRUB can see:*

```
grub> ls                                  # lists devices: (hd0) (hd0,gpt1) (hd0,gpt2) ...
grub> ls (hd0,gpt2)/                      # look for vmlinuz and initramfs, or a /boot directory
grub> ls (hd0,gpt2)/grub2/                # is grub.cfg present
grub> set                                 # current root and prefix variables
grub> insmod normal                       # can it load modules at all
```

If `ls` shows no partitions, the problem is the disk or the partition table, not GRUB. If `ls (hdX,Y)/` shows your kernel files, GRUB can read the filesystem and you only need to point it correctly.

*From rescue media, with the system mounted:*

```bash
# Boot the RHEL ISO -> Troubleshooting -> Rescue a Red Hat system -> option 1
chroot /mnt/sysimage             # RHEL 7   (RHEL 8/9/10: chroot /mnt/sysroot)

ls -l /boot/                      # are vmlinuz-* and initramfs-* present
ls -l /boot/grub2/                # grub.cfg, grubenv, i386-pc modules
ls -l /boot/efi/EFI/redhat/       # UEFI: grubx64.efi, shimx64.efi, grub.cfg
df -h /boot /boot/efi             # full or unmounted
mount | grep boot
lsblk -f ; blkid                  # partition layout and UUIDs
cat /etc/fstab                    # do the UUIDs match blkid
rpm -qa | grep -E "grub2|shim"    # are the packages intact
rpm -V grub2-common grub2-pc      # verify files against the package database
efibootmgr -v                     # UEFI: list boot entries and their paths
grubby --info=ALL                 # boot entries and kernel arguments
parted -l                         # partition table type: msdos vs gpt
```

*Disk health:* `smartctl -a /dev/sda`, `dmesg | grep -i "i/o error"`, and the RAID controller's own utility (`ssacli`, `storcli`, `perccli`) to confirm the boot volume is healthy and not degraded.

**TEMP FIX — get it booted now**

*Option A — boot manually from the GRUB prompt. This restores service in about a minute, then you fix it properly from a running system.*

```
grub> ls                                          # find the partition with vmlinuz
grub> ls (hd0,gpt2)/
grub> set root=(hd0,gpt2)
grub> linux /vmlinuz-4.18.0-513.el8.x86_64 root=/dev/mapper/rhel-root ro
grub> initrd /initramfs-4.18.0-513.el8.x86_64.img
grub> boot
```

If `/boot` is a separate partition, the paths are `/vmlinuz-...`; if `/boot` is inside root, they are `/boot/vmlinuz-...`. For LVM root add `rd.lvm.lv=rhel/root`. In `grub rescue>` mode you may first need `set prefix=(hd0,gpt2)/grub2` followed by `insmod normal` and `normal`.

*Option B — repair from rescue media (the standard fix).*

```bash
# Boot RHEL ISO -> Troubleshooting -> Rescue a Red Hat system -> 1) Continue
chroot /mnt/sysimage                  # RHEL 8/9/10: chroot /mnt/sysroot

### BIOS / legacy systems
grub2-install /dev/sda                # write the bootloader to the BOOT DISK, not a partition
grub2-mkconfig -o /boot/grub2/grub.cfg

### UEFI systems - do NOT run grub2-install; reinstall the packages instead
mount /boot/efi                       # make sure the ESP is mounted first
dnf reinstall -y grub2-efi-x64 grub2-efi-x64-modules shim-x64 grub2-common
grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg
efibootmgr -c -d /dev/sda -p 1 -L "Red Hat Enterprise Linux" -l '\EFI\redhat\shimx64.efi'

### If /boot files are missing entirely, reinstall the kernel to regenerate them
dnf reinstall -y kernel-core kernel-modules
# or from the ISO packages:
rpm -ivh --force /run/install/repo/Packages/kernel-<version>.rpm
dracut -f --regenerate-all

### Rebuild boot entries
grub2-mkconfig -o /boot/grub2/grub.cfg
grubby --info=ALL                     # verify entries exist and paths are correct

exit
reboot
```

On RHEL 9 and 10, `grub.cfg` lives at `/boot/grub2/grub.cfg` for both firmware types (the EFI directory holds a small stub), so regenerate to `/boot/grub2/grub.cfg` and let the stub redirect. Verify with `ls -l /boot/efi/EFI/redhat/grub.cfg`.

Two rules to state: **never hand-edit `grub.cfg`** — change `/etc/default/grub` and regenerate, because the next kernel update will overwrite hand edits. And on UEFI, **do not run `grub2-install`** on RHEL 8+; it is not supported and can break the ESP.

**PERMANENT FIX**

- Regenerate the configuration properly from `/etc/default/grub` and confirm every installed kernel has a valid boot entry (`grubby --info=ALL`), including the previous kernel as a fallback.
- **Free space in `/boot` and keep it free** — set `installonly_limit=3`, remove old kernels, and monitor `/boot` explicitly. A full `/boot` during a kernel update is the most common self-inflicted cause of an unbootable server.
- On UEFI, ensure `/boot/efi` is in `/etc/fstab` and mounted at boot, so bootloader updates always land on the real ESP.
- If the disk order changed, correct the firmware boot order and make the layout stable; for mirrored boot disks, install the bootloader to **both** members so a single disk failure does not leave you unbootable.
- Replace a failing boot disk rather than repeatedly rewriting the bootloader onto bad sectors.
- After any restore or clone, reconcile `/etc/fstab` UUIDs, the GRUB configuration and the initramfs as a documented post-restore step.
- Set `GRUB_TIMEOUT` sensibly and enable a **GRUB password** (`grub2-setpassword`) as part of hardening — while you are in there, note that this also closes the `rd.break` root-password-reset path.
- Document the exact recovery procedure for your platform (BIOS versus UEFI, RHEL version, disk layout) in the runbook.

**PREVENTION**

- Monitor `/boot` usage as its own filesystem alert; nobody notices it until a kernel update fails.
- Take a **snapshot or full backup before kernel updates and any partitioning or bootloader work**, so recovery is a restore rather than a repair.
- Back up the boot configuration as part of the system backup: `/boot`, `/etc/default/grub`, `/etc/fstab`, and `efibootmgr -v` output. For BIOS systems, an MBR copy (`dd if=/dev/sda of=/backup/mbr.bin bs=512 count=1`) is cheap insurance.
- Test a reboot inside the maintenance window after any boot-related change, so problems appear while you are watching and have the window to fix them.
- Keep rescue media (the correct RHEL major-version ISO) available and mountable through iLO/iDRAC virtual media, and confirm out-of-band console access works before you need it.
- Install the bootloader on all mirror members for redundant boot disks, and verify after disk replacements.
- Standardize the boot configuration as code (Kickstart plus Ansible) so no server has a hand-crafted, undocumented boot setup.
- Keep at least two working kernels installed at all times so the GRUB menu always offers a fallback.

---

### 20. If the server goes into maintenance / emergency mode, how will you fix it?

**SYMPTOM**

The boot stops and the console shows one of:

```
Welcome to emergency mode! After logging in, type "journalctl -xb" to view system logs...
Give root password for maintenance (or press Control-D to continue):
You are in emergency mode. ...
```

The server is up but unreachable over SSH, because networking and most services never started. `Control-D` either continues the boot (if the failure was non-critical) or loops straight back to the same prompt. In almost every case this is a **storage or filesystem problem detected during boot** — systemd could not satisfy a mount that something critical depends on, so it refuses to continue.

Note the difference between the two related states: **rescue mode** (formerly single-user) starts the root filesystem and a minimal set of units and is often entered deliberately; **emergency mode** is the more severe one, with almost nothing started and root frequently mounted read-only.

**TRIAGE (first 90 seconds)**

```bash
# Log in with the root password at the prompt, then:
journalctl -xb | tail -40           # the -x explanations name the failed unit and why
systemctl --failed                  # exactly what did not start
systemctl list-units --type=mount --all
mount | grep ' / '                  # is root read-only
dmesg -T | tail -30                 # I/O errors, XFS/ext4 complaints
cat /etc/fstab
blkid                               # do the real UUIDs match fstab
lsblk ; pvs ; vgs ; lvs             # is the LVM stack present and active
df -h                               # is something 100% full
```

Ninety seconds here is usually enough, because `journalctl -xb` and `systemctl --failed` name the failing mount unit directly, and comparing `blkid` against `/etc/fstab` confirms or eliminates the most common cause immediately.

**RCA — ranked root-cause possibilities**

1. **Bad `/etc/fstab` entry** — a typo, a stale UUID after a `mkfs` or restore, the wrong filesystem type, or an entry for a device that no longer exists. By far the most common, and usually the result of an edit that was never tested with `mount -a`.
2. **Volume group or logical volume not activated** at boot, so the device never appears.
3. **Missing or unavailable device** — a SAN LUN unmapped, an iSCSI target unreachable, a USB or external disk removed, or a NFS mount without `_netdev` attempted before the network is up.
4. **Filesystem corruption** — the mount fails or a required `fsck` failed, typically after an unclean shutdown or storage outage.
5. **Root filesystem full** — systemd cannot write what it needs, so the boot fails in confusing ways.
6. **Root mounted read-only** because of I/O errors, which then cascades into service failures.
7. **A failed `fsck` on an ext filesystem** at boot, which deliberately drops you to a shell for manual repair.
8. **A required systemd unit failing** that other units depend on (rarer, but possible after a configuration change or a broken package).
9. **SELinux relabel problems** or a missing critical directory after a bad restore.
10. **Corrupted or missing initramfs / wrong kernel parameters** — if the failure is earlier than this, you get a panic or a dracut shell instead.

**DIAGNOSIS — exact commands and evidence**

```bash
# 1. What failed and why - start here, always
journalctl -xb                            # this boot, with explanatory text
journalctl -xb -p err                     # errors only
journalctl -xb -u data.mount              # a specific mount unit
systemctl --failed
systemctl status data.mount
systemctl list-dependencies local-fs.target

# 2. Root may be read-only, so make it writable before editing anything
mount -o remount,rw /

# 3. Compare fstab against reality - the decisive check
cat /etc/fstab
blkid
lsblk -f -o NAME,FSTYPE,UUID,SIZE,MOUNTPOINT
findmnt --verify --verbose                 # validates fstab and reports exactly what is wrong

# 4. LVM stack
pvs ; vgs ; lvs -a -o +devices
vgchange -ay                               # activate and watch for errors
ls -l /dev/mapper/

# 5. Storage path health
multipath -ll
dmesg -T | grep -iE "i/o error|scsi|multipath|xfs|ext4|read-only"
cat /sys/block/sd*/device/state

# 6. Space and inodes
df -h ; df -i

# 7. Reproduce safely
mount -a                                   # shows the failing entry immediately
mount -v /data                             # verbose, one mount at a time
mount -o ro /dev/mapper/vg-lv /mnt         # can the data be read at all
```

The key log evidence is `journalctl -xb`, because the `-x` output literally explains which unit failed and what depends on it. `findmnt --verify` is the underused command here and worth naming in an interview — it validates the whole fstab and reports unknown filesystem types, missing devices and bad options without you having to attempt each mount.

**TEMP FIX — get it booted now**

```bash
mount -o remount,rw /                      # first, so you can edit files

# Fix or neutralize the offending fstab entry
vi /etc/fstab
#   - correct the UUID (copy it from blkid output)
#   - correct the filesystem type
#   - comment out the line if the device is genuinely gone
#   - add nofail so this mount can never block boot again
#   - add _netdev for NFS/iSCSI so it waits for the network

systemctl daemon-reload                    # regenerate mount units from the new fstab
mount -a                                   # must complete with NO errors
df -h                                      # confirm everything expected is mounted

# Inactive LVM
vgchange -ay
lvchange -ay /dev/vg_app/lv_data

# Filesystem repair (device must be UNMOUNTED)
xfs_repair /dev/mapper/vg_app-lv_data
e2fsck -y /dev/mapper/vg_app-lv_data

# Full root filesystem
journalctl --vacuum-size=100M
truncate -s 0 /var/log/<the-huge-log>
dnf clean all

# Missing mount point directory
mkdir -p /data

# Then continue or reboot
systemctl default                          # try to continue booting normally
reboot
```

If you **do not have the root password**, you are not stuck: reboot, press `e` at the GRUB menu, append `rd.break enforcing=0` (or `init=/bin/bash`) to the kernel line, press Ctrl+X, then `mount -o remount,rw /sysroot` and `chroot /sysroot` and repair from there. That is the same entry point as the root-password reset in the next question.

The critical discipline: **always run `mount -a` successfully before you reboot.** If `mount -a` is clean, the boot will be clean. If you reboot hoping for the best, you will land right back at the same prompt — and that loop is what turns a ten-minute fix into an hour-long outage.

**PERMANENT FIX**

- Correct the fstab entry to use the **current UUID** and the right type, and validate with `findmnt --verify` plus `mount -a`. Make "test with `mount -a`" a mandatory step in any procedure that touches fstab.
- Add **`nofail`** to every non-essential mount as standard policy, and `_netdev` to every network-backed mount. This single change means a missing data volume degrades the server instead of preventing it from booting.
- If the VG required manual activation, fix the underlying reason: LVM filters in `/etc/lvm/lvm.conf`, `lvm2-monitor` not enabled, or the volume group missing from the initramfs (`rd.lvm.lv=` plus `dracut -f`) when it backs root.
- Fix the storage layer properly — multipath configuration and timeouts, zoning, iSCSI startup ordering — rather than manually re-activating after each reboot.
- If corruption was caused by I/O errors, replace the failing hardware and restore from backup; repairing a filesystem on a dying disk buys days, not a fix.
- Separate filesystems (`/var`, `/var/log`, `/tmp`, `/home`, application data) so a full data volume cannot fill root and stall the boot.
- Remove `rhgb quiet` from the standard kernel line so boot failures are visible rather than hidden behind a splash screen.
- Ensure the root password is stored in the privileged-access vault — being unable to log in at the maintenance prompt turns a simple fix into a rescue-media exercise.

**PREVENTION**

- **Never edit `/etc/fstab` without running `mount -a` and `findmnt --verify` immediately afterwards.** Most emergency-mode incidents are created by an untested edit hours or weeks before the reboot that exposes it.
- Reboot inside the maintenance window after any storage or fstab change, so failures surface while you are watching and have time to fix them.
- Deploy fstab through configuration management with validation, so entries are consistent and reviewed rather than hand-typed.
- Standardize `nofail` on optional mounts across the estate as a build policy.
- Monitor mount points (alert when an expected filesystem is not mounted), filesystems going read-only, and multipath path failures — all of these are precursors.
- Keep the root password in a vault and verify out-of-band console access works, since emergency mode is only reachable from the console.
- Add a post-reboot validation step comparing `mount` and `systemctl --failed` output against a known-good baseline.
- Keep tested backups and, for virtual machines, a pre-change snapshot, so you always have a rollback that does not depend on debugging under pressure.

---

### 21. How will you reset a forgotten root password?

**RHEL 7 / 8 / 9 (systemd + SELinux):**

1. Reboot; at the GRUB menu press **`e`** to edit the default entry.
2. Find the line starting with `linux16` / `linux`/`linuxefi`, go to the end, and add:

```
rd.break enforcing=0
```

(remove `rhgb quiet` so you can see messages)

3. Press **Ctrl+X** to boot. You land in the initramfs shell with the real root mounted read-only at `/sysroot`.

```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root                       # set the new password
touch /.autorelabel                # required so SELinux relabels /etc/shadow
exit
exit                               # system continues booting and relabels, then reboots
```

If you used `enforcing=0` instead of `/.autorelabel`, after booting run `restorecon -v /etc/shadow` and `setenforce 1`.

**RHEL 6:** at GRUB press `a` (or `e`), append `single` or `1` to the kernel line, boot into single-user mode, run `passwd root`, reboot.

**Alternative for any version:** boot the install media in rescue mode → `chroot /mnt/sysimage` → `passwd root` → `touch /.autorelabel`.

**Security note:** in a real environment, GRUB should be password-protected (`grub2-setpassword`) precisely to prevent this — mention that in the interview.

---

### 22. If server performance is slow, what troubleshooting will you do?

**SYMPTOM**

"The server is slow" is the vaguest ticket you will receive, so the first job is turning it into something measurable. Ask, and write down the answers:

- **Slow doing what?** Logging in, one specific application function, all applications, file transfers, database queries, or the console itself?
- **How slow, compared to what?** Get a number and a baseline — "the report took 4 minutes, it normally takes 20 seconds."
- **When did it start, and is it constant or intermittent?** A specific start time is the single most valuable piece of information you can get.
- **Who is affected?** One user, one location, everyone, or only sessions from a particular network.
- **Was anything changed?** Patching, a deployment, a configuration change, more users onboarded, a new scheduled job, a storage or network change.
- **Is it actually this server?** Very often the server is healthy and the bottleneck is the network path, a downstream database, an authentication service, or the client.

Without this, you will spend an hour reading `top` and prove nothing.

**TRIAGE (first 90 seconds)**

```bash
uptime                       # load average vs nproc - the single fastest orientation
nproc
free -h                      # memory pressure; look at available and swap used
vmstat 1 5                   # r, b, si/so, wa, st - tells you WHICH resource in one command
df -h ; df -i                # full filesystem or inodes
top -b -n1 | head -20        # top CPU and memory consumers
iostat -xz 2 3               # is storage the bottleneck (%util, await)
dmesg -T | tail -30          # I/O errors, OOM, hardware, network resets
uptime && who | wc -l        # how many sessions
```

Read `vmstat` first and let it route you: **r** above the CPU count means CPU saturation, **b** and **wa** high means I/O, **si/so** non-zero means memory pressure and swapping, **st** non-zero means the hypervisor is the problem, and all of them low with the application still slow means the bottleneck is **not on this server** — look at the network, a dependency, or the application itself.

**RCA — ranked root-cause possibilities**

1. **Storage latency** — the most common and the most frequently misdiagnosed. SAN congestion, a failed multipath path, a degraded RAID rebuild, a snapshot consolidation, or a failing disk. Shows as high `%iowait` and high `await` with idle CPU.
2. **Memory pressure and swapping** — the system is alive but paging constantly, which looks like everything being slow at once.
3. **A competing scheduled job** — backup, antivirus scan, log compression, patch job, database index rebuild, or a report that now overlaps the business day. This is why "it is slow every night at 1 a.m." is a solved case before you even log in.
4. **Application-level problem** — a bad query plan after statistics changed, a missing index, thread-pool exhaustion, connection-pool starvation, garbage-collection pauses, or a memory leak.
5. **Hypervisor contention** — CPU ready time, memory ballooning, or datastore latency caused by other VMs. `%steal` is your evidence.
6. **CPU saturation** — genuine, from load growth or a runaway process.
7. **Network problems** — packet loss, retransmissions, a duplex or speed mismatch, MTU issues, a saturated uplink, or DNS timeouts (which make everything feel slow while showing no resource usage at all).
8. **A slow dependency** — an NFS server, database, LDAP/AD, or an API the application calls. The server is fine and waiting.
9. **Filesystem nearly full or fragmented**, especially XFS above 90%.
10. **Too many concurrent users or sessions** for the sizing.
11. **A monitoring or security agent misbehaving** — agents are a real and often-overlooked cause.
12. **Hardware degradation** — correctable memory errors, a throttling CPU due to thermal or power capping, or a degraded RAID cache battery causing write-through mode. That last one is a classic: a failed cache battery can slow a database by an order of magnitude.

**DIAGNOSIS — exact commands, layer by layer**

Work the **USE method**: for each resource check Utilization, Saturation, and Errors. Do not jump around; go in order and eliminate.

*CPU:*

```bash
mpstat -P ALL 2 5           # per-core; %usr %sys %iowait %steal %idle
pidstat -u 2 5              # per-process over time
top -H                      # per-thread
ps -eLo pid,tid,pcpu,comm --sort=-pcpu | head
perf top                    # which functions, if %sys is high
```

*Memory:*

```bash
free -h ; cat /proc/meminfo
vmstat 1 10                 # si/so = active swapping
ps aux --sort=-rss | head
smem -rs uss -k | head
sar -B                      # major faults, page scan rate
dmesg -T | grep -i oom
```

*Disk and I/O — usually where the answer is:*

```bash
iostat -xz 2 5              # r_await/w_await = latency per request; %util; aqu-sz
                            # await > 20ms on SSD or > 50ms on spinning disk = a real problem
iotop -oPa                  # which process is generating the I/O
pidstat -d 2 5
lsof +D /data | head
multipath -ll               # failed or faulty paths
smartctl -a /dev/sda        # reallocated sectors, pending sectors
ssacli/storcli/perccli show  # RAID status, rebuild progress, CACHE BATTERY state
df -h ; xfs_db -c frag -r /dev/mapper/vg-lv    # XFS fragmentation
```

*Network:*

```bash
ip -s link                  # errors, dropped, overruns per interface
ethtool eth0                # negotiated speed and duplex - check for 100Mb/half
ethtool -S eth0             # driver-level error counters
ss -s                       # socket summary
ss -tan state time-wait | wc -l
ss -tni                     # retransmission counts per connection
sar -n DEV 2 5 ; sar -n EDEV 2 5    # throughput and errors
ping -c 100 <peer>          # loss and jitter
mtr -rwc 100 <peer>         # per-hop loss
tcpdump -i eth0 -nn host <peer>     # retransmits, resets
nfsstat -c ; mountstats /data       # NFS latency per operation
dig example.com             # DNS response time - test this early, it is a common hidden cause
```

*Historical evidence — this is what turns opinion into proof:*

```bash
sar -u -f /var/log/sa/sa$(date +%d)          # CPU
sar -r -B -W -f /var/log/sa/sa$(date +%d)    # memory, paging, swapping
sar -d -p -f /var/log/sa/sa$(date +%d)       # per-disk service time
sar -n DEV -f /var/log/sa/sa$(date +%d)      # network
sar -q -f /var/log/sa/sa$(date +%d)          # run queue and load
sar -u -s 08:00:00 -e 10:00:00 -f /var/log/sa/sa14
```

*Application and change evidence:*

```bash
systemctl status <service>
journalctl -u <service> --since "2 hours ago"
# application logs, database slow-query log, web server access log response times
jstack <pid> > /tmp/thread.dump      # Java: thread states, deadlocks
jstat -gcutil <pid> 1000 10          # Java: GC pauses
jmap -histo:live <pid> | head -30    # Java: heap composition

rpm -qa --last | head -20 ; yum history          # what changed
crontab -l ; ls -l /etc/cron.d/ ; systemctl list-timers   # what runs at that time
find /etc -mtime -7 -type f          # recent config changes
```

*Virtualization evidence:* in vCenter check **CPU Ready %** (above 5% means contention), memory ballooning and swapping, and **datastore read/write latency** (above 20 ms is a problem). On the storage array, check port utilization and queue depths. These often explain a "slow server" whose internal metrics all look normal.

**TEMP FIX — immediate mitigation**

```bash
# Deprioritize, rather than kill, the competing workload
renice -n 19 -p <pid>
ionice -c2 -n7 -p <pid>
systemctl set-property <service> CPUQuota=100% MemoryMax=4G IOWeight=10

# Stop or postpone a conflicting batch job
systemctl stop <backup-agent>
crontab -e                        # move the job out of the business window

# Restart a degraded service (clears leaks, GC pressure, stuck threads)
systemctl restart <service>

# Storage
multipath -r                      # restore failed paths
echo 1024 > /sys/block/sdb/queue/nr_requests
# switch scheduler if clearly wrong for the device type
cat /sys/block/sdb/queue/scheduler

# Memory
swapon /swapfile                  # emergency swap
sysctl -w vm.swappiness=10        # for a database host

# Network
ethtool -s eth0 speed 1000 duplex full autoneg on    # fix a negotiation problem
```

Two cautions. Get the application owner's agreement before restarting anything in production — a restart may mask the evidence you need and can trigger a lengthy recovery. And confirm the alert is real before acting: check whether monitoring itself is broken, and whether the users' complaint matches what the metrics say.

**PERMANENT FIX**

- **If storage:** fix multipath configuration and I/O timeouts to vendor specification, replace failing disks, replace a failed RAID cache battery, move the workload to faster tiers or spread it across more spindles/LUNs, correct queue depths and the I/O scheduler for the device type, and align filesystem parameters for the workload.
- **If application:** hand the evidence to the application team with specifics — the slow query and its plan, the GC pause profile, the thread dump showing the blocked pool. The fix is an index, a query change, a heap or pool setting, or connection handling.
- **If scheduling:** reschedule and stagger batch jobs, set maximum runtimes with alerting, and move heavy reporting off the transactional server.
- **If capacity:** right-size through a change request with the `sar` trend as justification, or scale out behind a load balancer.
- **If hypervisor:** reduce oversubscription, set reservations, migrate the VM, or move to a faster datastore. Fix snapshot practices — long-lived snapshots are a common silent performance killer.
- **If network:** fix duplex/speed negotiation on both switch and server, correct MTU end to end, address the saturated uplink, fix DNS resolver configuration and add local caching.
- **Apply a tuned profile** appropriate to the role (`tuned-adm profile throughput-performance`, `virtual-guest`, or a database vendor's recommended profile) rather than tuning individual sysctls ad hoc.
- **Fix or replace a misbehaving agent** and set resource limits on all agents so they can never compete with production workloads.
- Document the root cause and the resolution in the ticket with the supporting numbers — "slow" incidents that close without a measured cause come back.

**PREVENTION**

- **Install and retain `sysstat`/`sar` history on every server.** This is the most valuable single preventive measure for performance work: without history, every incident starts from zero and you can never prove when something changed or whether it is trending.
- Establish and document a **performance baseline** per server and per application — normal CPU, memory, disk latency and throughput by hour and by day — so "slow" becomes a comparison rather than an opinion.
- Monitor **latency and saturation, not just utilization**: disk `await`, run-queue length, swap rate, `%steal`, retransmission rate, and application response time. Utilization alone hides the problems users actually feel.
- Add **application-level monitoring** (transaction response time, error rate, queue depth) so you learn about degradation before users report it, and can tell "the server" from "the application".
- Alert on the precursors: filesystems above 85%, multipath path failures, RAID degraded or cache battery failed, correctable memory errors, and interface error counters climbing.
- Review capacity trends monthly and resize in planned windows rather than reactively.
- Keep scheduled jobs in a documented calendar so overlapping batch windows are visible and avoidable by design.
- Load-test before major releases and before onboarding significant new user volume, and include a soak test long enough to expose leaks and fragmentation.
- Maintain a per-server runbook noting expected top consumers, normal load, and the known scheduled jobs, so on-call can distinguish abnormal from normal in seconds.

---

### 23. How to configure NFS — server side and client side

**Server:**

```bash
yum install -y nfs-utils
mkdir -p /export/share
chown nobody:nobody /export/share ; chmod 755 /export/share

vi /etc/exports
# /export/share  192.168.10.0/24(rw,sync,no_root_squash,no_subtree_check)
# /data          client1(ro)  client2(rw,sync)

exportfs -rav                     # export without restarting
systemctl enable --now nfs-server rpcbind    # RHEL7: nfs-server nfs-lock nfs-idmap
firewall-cmd --permanent --add-service={nfs,rpc-bind,mountd} ; firewall-cmd --reload
setsebool -P nfs_export_all_rw 1  # if SELinux blocks
exportfs -v ; showmount -e localhost
```

**Client:**

```bash
yum install -y nfs-utils
showmount -e <nfs-server>
mkdir -p /mnt/share
mount -t nfs <nfs-server>:/export/share /mnt/share
mount -t nfs -o vers=4,rw,hard,intr,rsize=8192,wsize=8192 server:/export/share /mnt/share

# persistent
vi /etc/fstab
# nfs-server:/export/share  /mnt/share  nfs  defaults,_netdev,soft,timeo=100  0 0
mount -a ; df -hT /mnt/share
```

**Key export options:** `rw/ro`, `sync` (safe) vs `async` (fast), `root_squash` (default: maps remote root to nobody) vs `no_root_squash`, `all_squash`, `sec=sys|krb5`.
**Key mount options:** `hard` (retry forever — default, safest for data) vs `soft` (fail with error), `intr`, `_netdev`, `bg`, `noatime`, `vers=3|4`.
**Ports:** NFSv4 = TCP 2049 only. NFSv3 also needs rpcbind 111, mountd, statd (pin them in `/etc/nfs.conf` or `/etc/sysconfig/nfs` for firewall rules).
**Troubleshooting:** `rpcinfo -p server`, `showmount -e`, `nfsstat`, `mount -v`, check firewall/SELinux, `journalctl -u nfs-server`.

---

### 24. How to configure FTP (vsftpd) — server side and client side

**Server:**

```bash
yum install -y vsftpd ftp
cp /etc/vsftpd/vsftpd.conf{,.bak}
vi /etc/vsftpd/vsftpd.conf
```

Key directives:

```
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
chroot_local_user=YES
allow_writeable_chroot=YES
dirmessage_enable=YES
xferlog_enable=YES
xferlog_file=/var/log/xferlog
listen=YES
listen_ipv6=NO
pam_service_name=vsftpd
userlist_enable=YES
userlist_deny=NO            # with this, /etc/vsftpd/user_list = allow list
pasv_enable=YES
pasv_min_port=30000
pasv_max_port=31000
```

```bash
systemctl enable --now vsftpd
firewall-cmd --permanent --add-service=ftp
firewall-cmd --permanent --add-port=30000-31000/tcp
firewall-cmd --reload
setsebool -P ftpd_full_access 1        # SELinux
useradd -m ftpuser ; passwd ftpuser
# deny users listed in /etc/vsftpd/ftpusers and user_list (root is denied by default)
```

**Client:**

```bash
ftp <server-ip>            # login, then: ls, cd, lcd, get, put, mget, mput, binary, ascii, bye
lftp -u user <server>
wget ftp://user:pass@server/file
curl -u user:pass ftp://server/path/
# passive mode if behind a firewall: inside ftp> passive
```

For real environments, say you'd prefer **SFTP/FTPS** because plain FTP sends credentials in clear text (`ssl_enable=YES`, `rsa_cert_file=...` to enable FTPS).

**Troubleshooting:** `systemctl status vsftpd`, `/var/log/vsftpd.log`, `/var/log/secure`, `ss -tulnp | grep 21`, SELinux booleans (`getsebool -a | grep ftp`), passive port range vs firewall.

---

### 25. How to configure Samba — server side and client side

**Server:**

```bash
yum install -y samba samba-client samba-common cifs-utils
mkdir -p /srv/samba/share
chmod 2775 /srv/samba/share
chown -R root:sambagrp /srv/samba/share
groupadd sambagrp ; useradd -s /sbin/nologin sambauser ; usermod -aG sambagrp sambauser
smbpasswd -a sambauser                  # create the Samba (not Unix) password
pdbedit -L -v                           # list samba users

vi /etc/samba/smb.conf
```

```ini
[global]
   workgroup = WORKGROUP
   server string = Samba Server %v
   netbios name = rhelsmb
   security = user
   map to guest = bad user
   log file = /var/log/samba/%m.log
   max log size = 50

[share]
   comment = Shared folder
   path = /srv/samba/share
   browsable = yes
   writable = yes
   guest ok = no
   valid users = @sambagrp
   create mask = 0664
   directory mask = 0775
```

```bash
testparm                                          # validate the config
systemctl enable --now smb nmb
firewall-cmd --permanent --add-service=samba ; firewall-cmd --reload
setsebool -P samba_enable_home_dirs on
semanage fcontext -a -t samba_share_t "/srv/samba/share(/.*)?"
restorecon -Rv /srv/samba/share
```

**Client (Linux):**

```bash
smbclient -L //server -U sambauser              # list shares
smbclient //server/share -U sambauser           # interactive
mkdir /mnt/smb
mount -t cifs //server/share /mnt/smb -o username=sambauser,password=xxx,vers=3.0

# persistent, with a credentials file (chmod 600)
echo -e "username=sambauser\npassword=xxx" > /root/.smbcred ; chmod 600 /root/.smbcred
vi /etc/fstab
# //server/share  /mnt/smb  cifs  credentials=/root/.smbcred,vers=3.0,_netdev,uid=1001,gid=1001  0 0
mount -a
```

**Client (Windows):** `\\server\share` in Explorer or `net use Z: \\server\share /user:sambauser`.

**Troubleshooting:** `testparm`, `/var/log/samba/`, `smbstatus`, ports 445/139 TCP and 137/138 UDP, SELinux context `samba_share_t`, `vers=` mismatch (SMB1 disabled on modern clients).

---

### 26. How to check RAM and CPU

**RAM:**

```bash
free -h ; free -m                 # total/used/free/buff-cache/available
cat /proc/meminfo                 # MemTotal, MemAvailable, SwapTotal, Cached
top / htop                        # live, sort with 'M'
vmstat -s
dmidecode -t memory               # physical DIMM detail: size, speed, slots, manufacturer
lshw -short -C memory
ps aux --sort=-%mem | head
smem -rs uss                      # per-process real usage
sar -r 1 5                        # current + historical
```

**CPU:**

```bash
lscpu                             # sockets, cores per socket, threads, model, MHz, cache
nproc                             # total logical CPUs
cat /proc/cpuinfo | grep -c processor
top / htop                        # press '1' for per-core view
mpstat -P ALL 2 5
sar -u 1 5
vmstat 1 5
dmidecode -t processor
lshw -short -C cpu
uptime                            # load average
```

Quick summary line for an interview: "`free -h` and `/proc/meminfo` for memory, `lscpu`/`nproc` and `/proc/cpuinfo` for CPU, `top`/`vmstat`/`sar` for live and historical utilization, `dmidecode` for the physical hardware view."

---

### 27. What is a zombie process?

A zombie (state **Z**, shown as `<defunct>`) is a process that has **finished executing but whose exit status has not yet been reaped by its parent** via `wait()`/`waitpid()`. The process itself is gone — no memory, no CPU — but its entry remains in the process table so the parent can read the exit code.

- **Cause:** a badly written or busy parent that doesn't call `wait()`, or ignores `SIGCHLD`.
- **Impact:** consumes no CPU/RAM, only a PID slot. A few are harmless; thousands can exhaust the PID table.
- **You cannot kill a zombie** — it's already dead. `kill -9` does nothing.

```bash
ps aux | awk '$8 ~ /^Z/ {print}'
ps -eo pid,ppid,stat,cmd | grep -w Z
top                            # "zombie" count in the summary line

# Fix: signal the PARENT to reap its children
kill -s SIGCHLD <ppid>
# If that fails, restart or kill the parent process; the zombies are then
# re-parented to init/systemd (PID 1), which reaps them immediately.
```

Contrast with an **orphan process**: parent died first, child still running, adopted by systemd (PID 1).

---

### 28. What is load average in Linux?

Load average is the **average number of processes that are either running on the CPU or waiting for CPU/uninterruptible I/O**, over the last **1, 5 and 15 minutes**.

```bash
uptime
# 14:02:31 up 40 days,  load average: 2.15, 1.90, 1.65
cat /proc/loadavg
# 2.15 1.90 1.65 3/512 28914   -> running/total threads, last PID
top ; w ; sar -q
```

**How to read it:** compare against the CPU count from `nproc`.

- On an 8-core box, load 8.00 = 100% utilized, load 4.00 = ~50%, load 16.00 = 2× oversubscribed.
- Rising 1-min vs falling 15-min = load is increasing now; the reverse = the spike is passing.
- **Linux load includes processes in uninterruptible sleep (D state)**, so a high load with idle CPUs usually means an **I/O or NFS/SAN bottleneck**, not a CPU shortage. Verify with `vmstat 1` (`r` vs `b` columns) and `iostat -xz`.

---

### 29. What are SUID, SGID and the sticky bit?

Special permission bits beyond rwx, set with the leading digit in `chmod 4755 / 2755 / 1777`.

**SUID (Set User ID) — 4000, shown as `s` in the owner execute position**
When an executable with SUID runs, it runs with the **file owner's** privileges instead of the caller's. This is how ordinary users can change their own password.

```bash
ls -l /usr/bin/passwd
-rwsr-xr-x. 1 root root ... /usr/bin/passwd
chmod u+s file      # or chmod 4755 file
```

Capital `S` means SUID set but the execute bit is missing. SUID is a major security concern — audit with `find / -perm -4000 -type f 2>/dev/null`. SUID is ignored on shell scripts on Linux and on filesystems mounted `nosuid`.

**SGID (Set Group ID) — 2000, `s` in the group execute position**

- On a **file**: the program runs with the file's group privileges (e.g. `/usr/bin/write`, `/usr/bin/locate`).
- On a **directory** (the common admin use): every new file/subdirectory created inside **inherits the directory's group**, not the creator's primary group. Essential for shared project/team directories.

```bash
chmod g+s /project      # or chmod 2775 /project
find / -perm -2000 -type f 2>/dev/null
```

**Sticky bit — 1000, `t` in the others execute position**
On a directory, it makes the directory **append-only for deletion**: a user can delete or rename a file inside only if they own the file (or the directory, or are root) — even though the directory is world-writable. Classic example `/tmp`:

```bash
ls -ld /tmp
drwxrwxrwt. 20 root root ... /tmp
chmod +t /shared     # or chmod 1777 /shared
find / -perm -1000 -type d 2>/dev/null
```

Summary table:

| Bit | Octal | Symbol | Applies to | Effect |
|---|---|---|---|---|
| SUID | 4000 | `s` on user | executables | run as file owner |
| SGID | 2000 | `s` on group | executables, directories | run as file group / inherit group |
| Sticky | 1000 | `t` on other | directories | only the owner can delete their files |

---

### 30. If a user is not able to access the server, what troubleshooting will you do?

This is the broader version of Q6. Q6 assumes the user reached the login prompt and authentication failed; this question also covers "cannot reach the server at all" and "logged in but cannot do what they need." Same framework, wider scope.

**SYMPTOM**

Pin down what "cannot access" actually means, because the three cases are investigated completely differently:

1. **Cannot reach the server** — timeout, no route to host, connection refused. This is network, firewall or host-down.
2. **Reaches it but cannot log in** — permission denied, access denied, immediate disconnect. This is authentication and authorization, covered in detail in Q6.
3. **Logs in but cannot access something** — a directory, a file, a command, a mount, or a service. This is permissions, sudo, SELinux or a missing mount.

Then establish scope and blast radius: one user or everyone, one server or many, one network/location or all, and whether it ever worked. "Everyone, all servers, started at 09:15" is an infrastructure incident; "one user, one server, new starter" is a provisioning gap.

**TRIAGE (first 90 seconds)**

```bash
# From the client / your workstation
ping <server>
nc -zv <server> 22
getent hosts <server>              # is the name resolving to the right IP
ssh -vvv user@server 2>&1 | tail -30

# On the server (console if SSH is unavailable)
systemctl status sshd ; ss -tulnp | grep :22
ip a ; ip r
firewall-cmd --list-all
df -h ; mount | grep ' / '         # full or read-only root blocks everything
uptime ; free -h
id <user> ; passwd -S <user> ; chage -l <user>
tail -30 /var/log/secure
```

The exact client error narrows it instantly: **timed out** means a firewall dropping or the host unreachable; **connection refused** means the host is up but nothing is listening or the firewall is rejecting; **permission denied** means you got to sshd and the problem is credentials or policy; **no route to host** is routing.

**RCA — ranked root-cause possibilities**

*Cannot reach the server:*

1. **Firewall, security-group or network ACL change** — often affects everyone simultaneously and correlates with a change window.
2. **Server down, hung, or in emergency mode** after a reboot or patch.
3. **DNS resolving to the wrong or an old IP** — very common after a migration or IP change.
4. **Network path problem** — routing, VPN not connected, a VLAN or switch change, a failed NIC or bond member.
5. **sshd not running or bound to the wrong address/port**, typically after a configuration change or a failed patch.
6. **Full or read-only root filesystem** preventing sshd from creating sessions.
7. **Resource exhaustion** — `MaxStartups` reached, out of memory, PID exhaustion, so connections are refused or hang.
8. **TCP wrappers** (`/etc/hosts.allow`, `/etc/hosts.deny`) on older builds.

*Can reach it but cannot log in:* see Q6 — account locked, expired, wrong shell, home directory missing, key permissions, `AllowUsers`/`AllowGroups`, PAM `faillock`, SSSD/AD failure, clock skew, SELinux.

*Logged in but cannot access a resource:*

9. **Filesystem permissions or group membership** — the user is not in the required group, or was added but has not re-logged-in so the new group is not in their session token.
10. **ACLs or SELinux context** denying access despite correct-looking `ls -l` permissions.
11. **A mount not present** — the data directory is empty because the NFS or LVM mount failed.
12. **sudo rules missing or too narrow.**
13. **The application's own authorization**, not the OS at all.

**DIAGNOSIS — exact commands and evidence**

*Reachability, from the client outward:*

```bash
ping -c 4 <server>
traceroute <server> ; mtr -rwc 50 <server>
nc -zv <server> 22
nmap -Pn -p 22,443 <server>
getent hosts <server> ; dig +short <server> ; nslookup <server>
cat /etc/hosts                      # stale entry overriding DNS
ssh -vvv user@server                # where exactly it stalls
```

*On the server:*

```bash
systemctl status sshd ; journalctl -u sshd -n 50
sshd -T | grep -Ei "port|listenaddress|allowusers|allowgroups|denyusers|permitrootlogin|passwordauth|maxstartups|maxsessions"
ss -tulnp | grep :22                # listening on 0.0.0.0 or only 127.0.0.1?
ip a ; ip r ; ip -s link            # interface up, errors, correct IP
firewall-cmd --list-all --zone=public ; iptables -L -n -v
cat /etc/hosts.allow /etc/hosts.deny
df -h ; df -i ; mount | grep ' / '
free -h ; ps -e | wc -l ; sysctl fs.file-nr
```

*Account and authorization:*

```bash
id <user> ; getent passwd <user> ; getent group <group>
passwd -S <user> ; chage -l <user>
faillock --user <user>
ls -ld /home/<user> /home/<user>/.ssh ; ls -l /home/<user>/.ssh/authorized_keys
sudo -l -U <user>
namei -l /data/app/file             # shows permissions at EVERY level of the path
getfacl /data/app
ls -Z /data/app ; ausearch -m avc -ts recent
findmnt /data                       # is the mount actually there
```

`namei -l` is the underused command for "permission denied on a file that looks readable" — it walks the whole path and shows where the traversal actually fails, usually a parent directory missing execute permission.

*Directory services:*

```bash
systemctl status sssd ; sssctl domain-status <domain>
realm list ; id <aduser>
chronyc tracking                    # Kerberos fails beyond ~5 minutes of skew
klist ; kinit <user>
sss_cache -E
journalctl -u sssd --since "30 min ago"
```

*Key log locations:* `/var/log/secure` (authentication), `journalctl -u sshd` and `-u sssd`, `/var/log/audit/audit.log` (SELinux and audit denials), `/var/log/messages`, plus the firewall or network team's logs for dropped connections.

**TEMP FIX — restore access now**

```bash
# Service and network layer
systemctl restart sshd
firewall-cmd --add-service=ssh                       # runtime, to confirm the firewall is the cause
firewall-cmd --add-source=10.1.1.0/24 --zone=trusted
nmcli con up ens192

# Account layer (see Q6 for the full set)
faillock --user <user> --reset
passwd -u <user> ; passwd <user>
chage -d $(date +%Y-%m-%d) <user> ; chage -E -1 <user>
usermod -s /bin/bash <user>

# Authorization layer
usermod -aG appgrp <user>        # user must log out and back in for this to take effect
setfacl -m u:<user>:rx /data/app
restorecon -Rv /data/app
mount -a                         # restore a missing mount

# Space / read-only root
mount -o remount,rw /
truncate -s 0 /var/log/<huge.log>
```

If the server is unreachable, go to the **out-of-band console** rather than guessing — iLO, iDRAC, vCenter or KVM. If it turns out to be in emergency mode, that is question 20. Always have the user confirm access is restored before closing, and record what you changed.

**PERMANENT FIX**

- If it was a firewall or network change, get the rule added permanently and recorded in the change management system, with `firewall-cmd --permanent` plus `--reload` on the server side and the equivalent on network devices. Find out why the change was made without accounting for this access path.
- If it was DNS, correct the record and the TTL, and fix any stale `/etc/hosts` entries that are masking DNS across the estate.
- If it was account provisioning, fix the joiner/mover/leaver process so shell, home directory, groups and aging are set correctly by automation rather than by hand.
- Manage SSH access by **group** (`AllowGroups`) rather than individual users, so access changes are a group membership update rather than an sshd_config edit on every server.
- If it was a lockout, eliminate the source — a saved credential, script, or monitoring check retrying an old password.
- If it was permissions, fix the group and directory design properly (SGID directories plus default ACLs for shared areas) instead of granting one-off ACLs repeatedly.
- If it was SELinux, add the correct file context rule with `semanage fcontext` and relabel; do not leave the system permissive.
- If it was SSSD or AD, fix DNS/site configuration, enable credential caching, and ensure chrony keeps time in sync so a brief directory outage does not lock out everyone.
- If a full or read-only root was the cause, extend the volume, fix log rotation, and separate filesystems.

**PREVENTION**

- Monitor the access path itself: an external check that actually opens port 22 and authenticates, so you learn about access failure before users do.
- Alert on sshd down, `/var/log/secure` failure spikes, account lockouts, SSSD/LDAP unavailability, root filesystem read-only, and filesystems above 85%.
- Standardize `sshd_config`, firewall rules, PAM configuration and account provisioning as code (Ansible), so no server drifts into a state that blocks users and every change is reviewed.
- Require change records for firewall, DNS and network changes, with an explicit check on management and monitoring access paths.
- Maintain and periodically **test** a break-glass access route — out-of-band console, a documented emergency account, vaulted root password — so an authentication or network failure never becomes a total lockout.
- Use keys or centralized authentication with sudo instead of shared passwords, removing the most common lockout cause.
- Add "verify normal-user login and application access" to the post-patch and post-change checklist.
- Document the expected access model per server (who, from where, via what) so deviations are obvious during troubleshooting.

---

### 31. Trying to unmount a mount point but it says "device is busy" — what will you do?

**SYMPTOM**

```
umount: /data: target is busy.
umount: /data: device is busy. (In some cases useful info about processes
        that use the device is found by lsof(8) or fuser(1))
```

The kernel is refusing because something still references the mount: a process with an open file, a process whose current working directory is inside it, a running executable or mapped library from it, a nested or bind mount underneath it, an active swap file, an NFS export, or a loop device. Establish the context first, because it changes how aggressive you can be: is this a planned decommission or migration, or an emergency? Is the mount serving live production data? And crucially, **is the mount responsive or hung?** A hung NFS mount behaves completely differently from a busy local filesystem.

**TRIAGE (first 90 seconds)**

```bash
fuser -vm /data                     # PIDs plus access type - the fastest single answer
lsof +f -- /data                    # which files are open, by whom
findmnt -R /data                    # nested/child mounts underneath
mount | grep /data                  # is it mounted more than once, or bind-mounted
pwd                                 # is it YOU sitting in the directory
```

In `fuser -vm` output, the ACCESS column tells you what kind of reference it is: **c** = current working directory, **o** = open file, **e** = running executable, **r** = root directory, **m** = mmapped file or shared library, **f** = open file (not displayed by default). A `c` is trivially fixed by having someone `cd` out; an `m` usually means a running binary and needs the service stopped.

If `fuser` or `lsof` themselves hang, you have a hung filesystem rather than a busy one — skip to the lazy/force unmount path.

**RCA — ranked root-cause possibilities**

1. **You or another admin has a shell whose working directory is inside the mount** — the most common and the most embarrassing. Includes forgotten `screen`/`tmux` sessions.
2. **A running service or application has files open** on the filesystem — database, web server, application server, log writer.
3. **A running executable or shared library is loaded from the mount** — you cannot unmount the filesystem a running binary came from.
4. **A nested mount or bind mount underneath the mount point** — including `/data/sub` mounted separately, or a bind mount elsewhere referencing it.
5. **A background job you did not think of** — backup agent, antivirus scan, monitoring plugin, `find` from a cron job, log shipper, or an indexing process.
6. **A `tail -f`, `less`, or editor left open** by someone, often in a detached session.
7. **A swap file located on the filesystem.**
8. **The filesystem is NFS-exported** from this server, so the kernel holds it.
9. **A loop device backed by a file** on the mount, or a device-mapper target on top of it.
10. **A container** (Podman/Docker) with a volume or bind mount into it.
11. **A hung NFS mount** where processes are stuck in uninterruptible D state and cannot be killed at all.
12. **Deleted-but-open files** still referenced by a process.

**DIAGNOSIS — exact commands**

```bash
# Who and what, in order of usefulness
fuser -vm /data                          # verbose: user, PID, access type, command
fuser -cu /data
lsof +f -- /data                          # scoped to the mount point (fast)
lsof +D /data                             # walks the whole tree (slower, more thorough)
lsof -n | grep '/data'
lsof /dev/mapper/vg_app-lv_data           # by device rather than path

# Processes whose CWD or root is inside the mount
ls -l /proc/*/cwd  2>/dev/null | grep /data
ls -l /proc/*/root 2>/dev/null | grep /data
ls -l /proc/*/exe  2>/dev/null | grep /data
grep -l /data /proc/*/maps 2>/dev/null     # mmapped files / loaded libraries
for p in /proc/[0-9]*; do ls -l $p/fd 2>/dev/null | grep -q /data && echo "$p"; done

# Nested mounts, bind mounts, shared subtrees
findmnt -R /data
findmnt -o TARGET,SOURCE,FSTYPE,PROPAGATION
cat /proc/mounts | grep /data
cat /proc/self/mountinfo | grep /data      # shows peer groups and bind relationships

# Other kernel-level holders
swapon -s ; cat /proc/swaps                # swap file on this filesystem
exportfs -v                                # is it NFS-exported
losetup -a                                 # loop devices
dmsetup ls --tree                           # device-mapper stack above it
podman ps -a --format '{{.ID}} {{.Mounts}}' ; docker ps -a
systemctl list-units --type=mount | grep data

# Is it hung rather than busy?
ps -eo pid,stat,wchan:25,cmd | awk '$2 ~ /D/'    # D-state = hung I/O, usually NFS
timeout 5 df -h /data || echo "filesystem not responding"
dmesg -T | grep -iE "nfs.*not responding|i/o error"
```

**TEMP FIX — release it, least disruptive first**

Escalate in this order. Do not start at `fuser -km`.

```bash
# 1. The obvious one - get out of the directory yourself
cd /
# and check for your own detached sessions
screen -ls ; tmux ls

# 2. Ask the owner / stop the service cleanly - the correct answer in production
systemctl stop <service>

# 3. Stop the specific processes, gracefully first
kill -TERM <pid>
sleep 5
kill -9 <pid>                             # only if it refuses, and only if safe

# 4. Signal everything using the mount (blunt - confirm impact first)
fuser -k -TERM -m /data                   # SIGTERM to all users of the mount
fuser -km /data                           # SIGKILL to all users - can cause data loss

# 5. Clear non-process holders
swapoff /data/swapfile                    # swap file
exportfs -u <client>:/data                # remove the NFS export
exportfs -ua                              # unexport everything
losetup -d /dev/loop0                     # detach loop device
podman stop <container> / docker stop <container>

# 6. Nested mounts - unmount children first, or recursively
umount -R /data

# 7. Fallback unmount methods
umount -l /data          # LAZY: detach from the namespace now, release when the last
                         # reference closes. Best choice when you cannot stop the holder.
umount -f /data          # FORCE: mainly for NFS where the server is unreachable
umount -f -l /data       # both, for a hung NFS mount
umount -f -t nfs /data

# 8. Verify it is really gone
findmnt /data ; mount | grep /data ; df -h | grep /data
cat /proc/mounts | grep /data
```

Understand what you are choosing between: **`umount -l` (lazy)** removes the mount from the filesystem namespace immediately so new access cannot reach it, but the filesystem stays busy in the background until the last handle closes — safe, and usually the right answer when a service cannot be stopped, but the device is still not free for `lvremove`. **`umount -f` (force)** actively aborts pending requests and is intended for unreachable NFS servers; on a healthy local filesystem it risks losing buffered writes. And **`fuser -km`** SIGKILLs every process touching the mount, which on a database means an unclean shutdown and a recovery cycle.

For a **genuinely hung NFS mount**, the processes are in D state and cannot be killed by any signal. Use `umount -f -l`, and if the NFS server is permanently gone you may need to restart `nfs-client.target`/`rpc-statd` or, in the worst case, reboot the server. Say this explicitly in an interview — knowing that D-state processes are unkillable and that a reboot is sometimes the only option shows real experience.

If you are unmounting in order to remove the storage entirely:

```bash
umount /data
vi /etc/fstab                             # REMOVE or comment the entry first - critical,
                                          # or the next reboot drops you into emergency mode
systemctl daemon-reload
lvremove /dev/vg_app/lv_data
vgreduce vg_app /dev/sdc
pvremove /dev/sdc
dmsetup remove vg_app-lv_data             # if the mapper device lingers
multipath -f <wwid>                       # release the multipath map before the LUN is unmapped
```

**PERMANENT FIX**

- Build a proper **decommission or maintenance procedure** for the mount: stop the dependent services in the right order, unmount, then remove storage. "Device busy" is usually a symptom of doing these steps out of order rather than a problem in itself.
- Identify every consumer of shared filesystems and **document the dependency** so the next person knows what to stop. If you had to hunt with `lsof`, that documentation did not exist.
- Use **systemd dependencies** so services that require a filesystem are ordered against it (`RequiresMountsFor=/data`), which makes systemd stop them before the mount goes away and prevents this situation entirely.
- Remove the fstab entry as part of the same change, so a partially completed decommission cannot break the next boot.
- If a backup or scanning agent was the holder, exclude the path or schedule it so it never overlaps maintenance windows.
- For NFS, use appropriate mount options (`_netdev`, `soft` with sensible `timeo`/`retrans` where the application tolerates errors) so an unreachable server produces errors instead of unkillable processes, and make the NFS server highly available.
- Move swap off application data volumes so it never blocks a maintenance action.

**PREVENTION**

- Schedule storage changes in a **maintenance window with the dependent services stopped**, rather than trying to unmount a live filesystem.
- Add a **pre-unmount checklist** to the runbook: stop services, `fuser -vm`, `findmnt -R`, check `swapon -s` and `exportfs -v`, confirm no containers, then unmount, then update fstab.
- Encode service-to-filesystem dependencies with `RequiresMountsFor=` so ordering is automatic and enforced.
- Discourage working directly inside shared mount points, and be disciplined about closing `screen`/`tmux` sessions — the single most common cause is an admin's own forgotten shell.
- Monitor mount points and their consumers so you have a current picture of dependencies before you need it.
- Configure NFS mounts and timeouts correctly across the estate so hung-mount incidents become rare, and document the recovery procedure for when they happen.
- Always update `/etc/fstab` in the same change as the unmount, and validate with `findmnt --verify` plus `mount -a` before rebooting.

---

## Part 2 — Additional Questions Interviewers Commonly Add

Part 1 covered your list. This part covers the follow-up questions that almost always come next, grouped by topic. Each answer is written the way you would actually speak it — explanation first, then the commands.

### Filesystem, storage and LVM

**Q: How do you scan a newly added disk or SAN LUN without rebooting?**

When the storage team presents a new LUN, the kernel does not see it until you tell it to rescan the SCSI bus. You issue a rescan on every SCSI host adapter, then confirm the device appeared.

```bash
for h in /sys/class/scsi_host/host*/scan; do echo "- - -" > $h; done
# or, if sg3_utils is installed:
rescan-scsi-bus.sh -a

lsblk                # new device should appear as sdX
multipath -ll        # on SAN, confirm all paths are active
multipath -r         # reload multipath maps if the new LUN is not grouped
```

For an existing disk that was *expanded* rather than newly added, you rescan just that device with `echo 1 > /sys/class/block/sdb/device/rescan` and then run `pvresize` so LVM notices the extra space.

**Q: Why do `df` and `du` sometimes disagree?**

This trips up a lot of people, and it is a very common real-world incident. `df` reads the filesystem's own accounting of allocated blocks, while `du` walks the directory tree and adds up the files it can actually see. The usual reason they disagree is a **deleted file that is still held open by a running process**. When you delete a file that a process has open, the directory entry disappears — so `du` no longer counts it — but the blocks stay allocated until the last file handle closes, so `df` still reports the space as used. Classic case: someone deletes a huge application log while the application is still writing to it, and the disk stays full.

```bash
lsof +L1              # files with link count 0 - deleted but still open
lsof | grep deleted
```

The fix is to restart or signal the process holding the file, or truncate it in place via `> /proc/<pid>/fd/<n>`. Two other causes of disagreement: sparse files, where the apparent size is bigger than the blocks actually used, and a mount point that has another filesystem mounted over it, hiding files underneath from `du`.

**Q: What is `/etc/fstab` and what are its fields?**

`/etc/fstab` is the table the system reads at boot to decide what to mount where. Each line has six fields:

```
UUID=1a2b-3c4d   /data   xfs   defaults   0   0
device/UUID      mount   type  options    dump  fsck_order
```

Always use `UUID=` or `LABEL=` rather than `/dev/sdb1`, because device letters can change when disks are added, removed or re-presented — and a wrong device name here is one of the top causes of a server dropping into emergency mode. The dump field is effectively obsolete (leave it 0), and the fsck order should be `1` for root, `2` for other ext filesystems, and `0` for XFS and swap. Two options worth mentioning: `nofail` so an optional mount that is missing does not block the boot, and `_netdev` so network filesystems wait for the network to come up.

**Q: What is an inode?**

An inode is the data structure that stores everything about a file *except* its name and its contents — permissions, owner and group, size, timestamps, link count, and pointers to the data blocks. The filename lives in the directory entry, which simply maps a name to an inode number. That is exactly why hard links work: two names can point to the same inode.

The practical reason interviewers ask this is inode exhaustion. A filesystem has a fixed number of inodes created at `mkfs` time, so a directory holding millions of tiny files can run out of inodes while `df -h` still shows plenty of free space. You then get the confusing error "No space left on device" on a disk that looks half empty.

```bash
df -i                                    # inode usage per filesystem
for d in /var/*; do echo "$(find $d | wc -l) $d"; done | sort -rn | head
```

Typical culprits are mail queues, PHP session directories, and application temp directories. The fix is to delete the small files, and long term either recreate the filesystem with more inodes or move to XFS, which allocates inodes dynamically.

**Q: What is an LVM snapshot and when do you use it?**

A snapshot is a point-in-time, copy-on-write view of a logical volume. When you create it, LVM does not copy the data — it just records the state, and from then on any block that changes on the original is first copied into the snapshot area. That makes snapshots almost instant to create and cheap in space, as long as the amount of change stays small.

```bash
lvcreate -s -L 5G -n lv_data_snap /dev/vg_app/lv_data   # create
mount /dev/vg_app/lv_data_snap /mnt/snap                # mount read-only and back it up
lvs                                                     # watch the Data% column
lvconvert --merge /dev/vg_app/lv_data_snap              # roll the original BACK to snapshot state
lvremove /dev/vg_app/lv_data_snap                       # discard
```

The two things to say: it is used for **consistent backups** (freeze a snapshot, back it up while the application keeps running) and as a **rollback point before patching or an application upgrade**. And the warning — **if a snapshot fills up it becomes invalid and unusable**, so you must size it for the expected rate of change and monitor `Data%`.

**Q: What RAID levels do you know?**

- **RAID 0** — striping across disks. Fast, uses all capacity, but zero redundancy; one disk lost means everything lost.
- **RAID 1** — mirroring. Full redundancy, but only 50% of raw capacity is usable. Typical for OS/boot disks.
- **RAID 5** — striping with a single parity block distributed across disks. You get n-1 usable capacity and survive one disk failure. The concern is the long, risky rebuild on large modern disks.
- **RAID 6** — double parity. Survives two simultaneous failures, n-2 usable, slightly slower writes. Preferred over RAID 5 for large arrays.
- **RAID 10** — striped mirrors. Best combination of performance and resilience, 50% usable. Standard choice for databases and write-heavy workloads.

For software RAID on Linux you use `mdadm`: `cat /proc/mdstat` for status, `mdadm --detail /dev/md0`, `mdadm --manage /dev/md0 --fail /dev/sdb1 --remove /dev/sdb1 --add /dev/sdd1` to replace a disk.

**Q: How do you check and repair a filesystem?**

The filesystem must be **unmounted** first — running a repair on a mounted filesystem can destroy it.

```bash
# XFS
xfs_repair -n /dev/vg_app/lv_data     # dry run, report only
xfs_repair /dev/vg_app/lv_data        # actual repair
xfs_repair -L /dev/vg_app/lv_data     # zeroes the dirty log - LAST RESORT, data loss possible

# ext2/3/4
e2fsck -f /dev/vg_app/lv_data         # force a check
fsck -y /dev/vg_app/lv_data           # auto-answer yes
e2fsck -b 32768 /dev/vg_app/lv_data   # use a backup superblock if the primary is damaged
tune2fs -l /dev/vg_app/lv_data        # view superblock details, mount count, check interval
```

To force a check on the next boot for ext filesystems, `touch /forcefsck` or set the interval with `tune2fs -c 30`.

**Q: What is the difference between a partition, a physical volume, a volume group and a logical volume?**

Think of it as a supply chain. A **partition** (or a whole raw disk) is the physical slice of storage. You initialize it as a **physical volume (PV)** so LVM can use it. You pool one or more PVs into a **volume group (VG)**, which is just a big flexible bucket of space measured in extents. Out of that bucket you carve **logical volumes (LVs)**, which behave like partitions but can be resized, moved between disks, snapshotted and striped. Finally you put a **filesystem** on the LV and mount it. The whole point of LVM is that you can grow storage online and are no longer limited by fixed partition boundaries.

**Q: What is the difference between XFS and ext4 in practice?**

XFS is the RHEL 7+ default. It scales far higher (500 TB versus about 16 TB), handles large files and highly parallel I/O better, and grows online with `xfs_growfs`. Its one significant limitation is that **it cannot be shrunk** — ever. ext4 is more conservative, supports both growing and shrinking with `resize2fs`, and is slightly better for very small filesystems and workloads with huge numbers of tiny files. In an interview: "I choose XFS by default on RHEL 7 and later, but if I know a filesystem may need to be reduced later, I use ext4."

### systemd and services

**Q: What are the systemctl commands you use daily?**

```bash
systemctl status httpd            # is it running, recent log lines, PID, memory
systemctl start|stop|restart httpd
systemctl reload httpd            # re-read config WITHOUT dropping connections
systemctl enable --now httpd      # start now and at every boot
systemctl disable httpd
systemctl is-active / is-enabled / is-failed httpd
systemctl list-units --type=service --state=running
systemctl --failed                # anything broken right now
systemctl daemon-reload           # MUST run after editing any unit file
systemctl get-default             # current boot target
systemctl set-default multi-user.target
systemd-analyze blame             # what is making boot slow
```

The one people forget is `daemon-reload`. If you edit a unit file and just restart the service, systemd keeps using the cached old definition and you will chase a ghost.

**Q: What is the difference between `disable` and `mask`?**

`disable` only removes the symlink that starts the service at boot. The service can still be started by hand, and — importantly — it can still be pulled in automatically as a dependency of something else. `mask` is the hard stop: it links the unit to `/dev/null`, so the service **cannot be started at all**, by anyone or anything, until you `unmask` it. You use mask when you need a guarantee, for example masking a monitoring agent or a database during a maintenance window so nothing accidentally brings it back up.

**Q: What is the difference between `restart` and `reload`?**

`restart` stops the process and starts a new one, which drops all active connections and briefly takes the service down. `reload` sends the daemon a signal (usually SIGHUP) telling it to re-read its configuration while continuing to serve traffic. For production web servers, load balancers and DNS, always prefer `reload`. `reload-or-restart` is a safe middle ground when you are not sure the unit supports reload.

**Q: Where do unit files live and how do you customize one properly?**

Vendor-shipped units live in `/usr/lib/systemd/system/`. You never edit those, because a package update will overwrite your changes. Administrator units and overrides go in `/etc/systemd/system/`, which takes precedence. The clean way to change one setting is a drop-in override:

```bash
systemctl edit httpd            # creates /etc/systemd/system/httpd.service.d/override.conf
systemctl cat httpd             # shows the effective, merged unit - very useful for verifying
systemctl show httpd            # every resolved property
systemctl daemon-reload && systemctl restart httpd
```

`systemctl edit --full` gives you a full copy to edit if you need to change more than a few lines.

**Q: How do you look at logs with journalctl?**

```bash
journalctl -u sshd                     # one service
journalctl -u sshd -f                  # follow live, like tail -f
journalctl -u sshd --since "1 hour ago" --until "10 min ago"
journalctl -p err -b                   # errors and worse, this boot
journalctl -b -1                       # the PREVIOUS boot - essential after a crash
journalctl --list-boots
journalctl -k                          # kernel messages only
journalctl _PID=1234
journalctl --disk-usage
journalctl --vacuum-time=30d           # trim old logs when /var/log is full
```

By default on some systems the journal is stored only in memory and is lost at reboot. To make it persistent so you can investigate crashes afterwards, create the directory and restart journald:

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

**Q: What is a systemd target and how does it relate to runlevels?**

A target is a named group of units that represents a system state — the replacement for runlevels. `multi-user.target` is the old runlevel 3 (full multi-user, no GUI), `graphical.target` is runlevel 5, `rescue.target` is single-user mode, and `emergency.target` is the minimal shell you land in when even rescue cannot start. You switch with `systemctl isolate multi-user.target` and set the default with `systemctl set-default`.

### Networking

**Q: How do you configure a static IP address?**

On RHEL 7 through 10 the supported way is NetworkManager via `nmcli`. Note that on RHEL 9 and 10 the old `ifcfg` files are gone entirely, so `nmcli` is not optional any more.

```bash
nmcli con show                                    # list connections
nmcli dev status

nmcli con mod ens192 ipv4.method manual \
  ipv4.addresses 10.1.1.20/24 \
  ipv4.gateway 10.1.1.1 \
  ipv4.dns "10.1.1.5 8.8.8.8" \
  ipv4.dns-search example.com \
  connection.autoconnect yes

nmcli con down ens192 && nmcli con up ens192      # apply
ip a ; ip r ; cat /etc/resolv.conf                # verify
```

`nmtui` gives you the same thing in a text menu, which is safer if you are working over the very SSH session you are about to reconfigure. On RHEL 7/8 you may still find `/etc/sysconfig/network-scripts/ifcfg-ens192`; after editing it directly you must run `nmcli con reload`.

**Q: How do you see what is listening on a port and which process owns it?**

```bash
ss -tulnp                     # t=tcp u=udp l=listening n=numeric p=process
ss -tulnp | grep :443
ss -tan state established      # current connections
lsof -i :443
fuser -n tcp 443
netstat -tulnp                 # deprecated but still asked about
```

`ss` replaced `netstat` and is significantly faster on busy servers. Mention that difference — interviewers like it.

**Q: How do you troubleshoot a DNS problem?**

Work from the client's own configuration outward. Check `/etc/resolv.conf` for the nameservers and search domain, check `/etc/nsswitch.conf` to confirm the lookup order (`hosts: files dns`), and check `/etc/hosts` for a stale static entry that is overriding DNS — that is a surprisingly common cause.

```bash
dig +short myserver.example.com        # forward lookup
dig -x 10.1.1.20                       # reverse lookup
dig @10.1.1.5 myserver.example.com     # test a specific DNS server directly
nslookup myserver
getent hosts myserver                  # what the OS resolver actually returns, honouring nsswitch
host myserver
```

If `dig @server` works but `getent` does not, the problem is local configuration, not the DNS server. Also confirm UDP/TCP 53 is not blocked and that the search domain is correct if short names fail but FQDNs work.

**Q: What is NIC bonding or teaming?**

Bonding combines two or more physical NICs into one logical interface, either for redundancy or for extra throughput. The modes you should know: **mode 1 (active-backup)** is the most widely used in enterprise — one NIC carries traffic and the other takes over instantly if the link fails; **mode 4 (802.3ad / LACP)** aggregates bandwidth and needs matching configuration on the switch; **mode 0 (round-robin)** and **mode 6 (balance-alb)** are less common. Teaming was the RHEL 7 alternative implementation but bonding is what survived and is recommended again in RHEL 9+.

```bash
nmcli con add type bond ifname bond0 mode active-backup
nmcli con add type ethernet ifname ens192 master bond0
nmcli con add type ethernet ifname ens224 master bond0
nmcli con mod bond0 ipv4.addresses 10.1.1.20/24 ipv4.gateway 10.1.1.1 ipv4.method manual
nmcli con up bond0
cat /proc/net/bonding/bond0            # which slave is active, link status, failure counts
```

**Q: How do you work with firewalld?**

firewalld is zone-based: every interface belongs to a zone, and the zone decides what is allowed. The critical habit is that any change you want to survive a reload or reboot needs `--permanent` **and** a `--reload` afterwards.

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all                                  # rules for the default zone
firewall-cmd --permanent --add-service=https
firewall-cmd --permanent --add-port=8080/tcp
firewall-cmd --permanent --add-source=10.1.1.0/24 --zone=trusted
firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=10.2.0.0/24 service name=ssh accept'
firewall-cmd --permanent --remove-port=8080/tcp
firewall-cmd --reload
firewall-cmd --runtime-to-permanent                      # save rules you tested live
```

If you add a rule without `--permanent` it works immediately but vanishes on reload — and if you add it only with `--permanent` and forget to reload, nothing happens now. Both mistakes are classic interview traps.

**Q: How do you check whether a remote port is reachable?**

```bash
nc -zv dbserver 1521
timeout 3 bash -c "</dev/tcp/dbserver/1521" && echo open
telnet dbserver 1521
curl -v telnet://dbserver:1521
nmap -p 1521 dbserver
```

Interpret the result properly: **connection refused** means you reached the host but nothing is listening (or the firewall is rejecting); **timed out** means a firewall is silently dropping packets or the host is unreachable; **no route to host** is a routing or gateway problem.

### Users, groups and permissions

**Q: How do you create a user with a specific UID, group, home directory and shell?**

```bash
groupadd -g 2000 appgrp
useradd -u 1500 -g appgrp -G wheel,dba -d /home/appuser -s /bin/bash -c "Application owner" appuser
passwd appuser
chage -d 0 appuser              # force a password change at first login
id appuser                      # verify UID, GID and secondary groups
```

Two details worth mentioning. The home directory is populated from `/etc/skel`, so if you want every new user to get a standard `.bashrc` or profile settings, that is where you put them. And when adding secondary groups always use `usermod -aG` — **without `-a` you replace the entire secondary group list** and silently remove the user from every other group, which is a genuinely dangerous mistake.

Related commands: `usermod -l newname olduser` to rename, `usermod -L`/`-U` to lock/unlock, `userdel -r appuser` to delete along with the home directory, `getent passwd appuser` to look up an account including LDAP/AD sources.

**Q: How do you give a user sudo rights safely?**

The clean way is group membership plus a dedicated file, never direct editing of `/etc/sudoers`.

```bash
usermod -aG wheel john                          # wheel already has full sudo on RHEL

# For limited, specific rights:
visudo -f /etc/sudoers.d/appteam
# appuser ALL=(root) NOPASSWD: /bin/systemctl restart myapp, /bin/systemctl status myapp
# %dba    ALL=(oracle) ALL

sudo -l -U appuser                              # verify what they can actually run
```

Always use `visudo` (or `visudo -f`) rather than a normal editor, because it validates the syntax before saving. A broken sudoers file can lock everyone out of root access on the box. Also mention the principle of least privilege — grant specific commands rather than `ALL`, and avoid `NOPASSWD` on anything that can spawn a shell (like `vi` or `less`), because that is a trivial privilege escalation path.

**Q: What is umask?**

umask is the mask of permission bits that are *removed* from the default when a new file or directory is created. Defaults start at 666 for files and 777 for directories, so with the common umask of 022 you get files at 644 and directories at 755. A umask of 027 gives 640 and 750, which is what hardening standards usually require. It is set in `/etc/profile`, `/etc/bashrc`, `/etc/login.defs`, or a user's own `~/.bashrc`. Check with plain `umask` and set with `umask 027`.

**Q: What are ACLs and when do you need them?**

Standard Unix permissions only let you express owner, one group, and everyone else. ACLs let you grant specific rights to additional individual users or groups — for example, giving an auditor read access to one directory without changing its group ownership or adding them to a group.

```bash
setfacl -m u:john:rw report.txt              # grant john read/write
setfacl -m g:auditors:rx /var/log/app
setfacl -m d:g:devteam:rwx /project          # DEFAULT acl - inherited by new files
setfacl -R -m u:john:rX /project             # recursive
getfacl /project                             # view
setfacl -x u:john report.txt                 # remove one entry
setfacl -b report.txt                        # remove all ACLs
```

A `+` at the end of the permission string in `ls -l` tells you an ACL is present. The default ACL (`d:`) is the important one for shared team directories, because it makes new files inherit the rights automatically.

**Q: What are the SELinux modes and the commands you use?**

SELinux is mandatory access control: even root-owned processes are confined to what their security context allows. There are three modes — **enforcing** (blocks and logs), **permissive** (allows but logs, ideal for troubleshooting), and **disabled**.

```bash
getenforce ; sestatus
setenforce 0                                  # permissive, temporary
vi /etc/selinux/config                        # SELINUX=enforcing|permissive|disabled - persistent
ls -Z /var/www/html ; ps -Z ; id -Z           # view contexts

restorecon -Rv /var/www/html                  # reset to policy default - the usual fix
semanage fcontext -a -t httpd_sys_content_t "/webdata(/.*)?"   # define a permanent context
restorecon -Rv /webdata
semanage port -a -t http_port_t -p tcp 8080   # allow httpd on a non-standard port
getsebool -a | grep httpd                     # list booleans
setsebool -P httpd_can_network_connect on     # -P makes it persistent

ausearch -m avc -ts recent                    # what was denied
sealert -a /var/log/audit/audit.log           # human-readable explanation and suggested fix
```

The single most valuable thing to say here: **do not disable SELinux to fix a problem.** Set it to permissive, reproduce the issue, read the AVC denial, fix the context or set the right boolean, then put it back to enforcing. Also note that switching from disabled back to enforcing requires a full relabel (`touch /.autorelabel` and reboot).

**Q: How do you find out who logged in and what they did?**

```bash
who ; w                          # who is on right now, and what they are running
last                             # login history from /var/log/wtmp
last -n 20 username
lastb                            # FAILED login attempts from /var/log/btmp
lastlog                          # last login time for every account
who -b                           # last boot time

grep -i "Failed password" /var/log/secure | tail -20
grep -i "Accepted" /var/log/secure
journalctl -u sshd --since today

ausearch -ua 1500 -ts today      # audit trail for a specific UID
aureport --summary
```

For real accountability you mention auditd rules and centralized logging, because local logs can be tampered with by whoever had root.

### Processes and performance

**Q: What are the process states you see in `ps`?**

- **R** — running or runnable, sitting on or waiting for the CPU.
- **S** — interruptible sleep, waiting for an event. This is where most processes are, and it is normal.
- **D** — uninterruptible sleep, almost always blocked on I/O. **This is the one that matters when troubleshooting.** A process stuck in D cannot be killed, not even with `kill -9`, and a pile of D-state processes usually means a hung NFS mount, a failed SAN path, or a dying disk.
- **T** — stopped, either by a job-control signal or by a debugger.
- **Z** — zombie, finished but not yet reaped by its parent.
- **I** — idle kernel thread (newer kernels).

You will also see modifier flags: `s` for session leader, `+` for foreground process group, `l` for multi-threaded, `<` for high priority, `N` for low priority.

```bash
ps -eo pid,ppid,stat,wchan:20,cmd | awk '$3 ~ /D/'    # find blocked processes and what they wait on
```

**Q: What is the difference between `kill`, `kill -9` and `kill -1`?**

`kill <pid>` sends **SIGTERM (15)**, which politely asks the process to shut down. The process can catch it, flush its buffers, close files and database connections, and exit cleanly. This should always be your first attempt.

`kill -9 <pid>` sends **SIGKILL**, which is handled by the kernel and cannot be caught, blocked or ignored. The process dies instantly with no chance to clean up — which means unflushed data is lost, temp and lock files are left behind, and databases may need recovery. Use it only after SIGTERM has failed.

`kill -1 <pid>` sends **SIGHUP**, which historically meant "the terminal hung up" but which most daemons now interpret as "re-read your configuration file" — a way to apply config changes without dropping connections.

Also useful: `kill -0 <pid>` sends no signal and just tests whether the process exists and you have permission to signal it; `pkill -f pattern` and `killall name` match by name; `kill -l` lists all signals; SIGSTOP (19) and SIGCONT (18) pause and resume a process, which is handy for throttling a runaway job without killing it.

**Q: What are nice and renice?**

They control the CPU scheduling priority of a process. The nice value runs from **-20 (highest priority) to +19 (lowest)**, with 0 as default. Confusingly, a *higher* nice number means the process is *nicer* to others, so it gets less CPU. Any user can lower their own priority, but **only root can raise it** (use a negative value).

```bash
nice -n 10 ./bulk_import.sh          # start a batch job at low priority
renice -n 5 -p 4821                  # change a running process
renice -n 10 -u batchuser            # every process owned by a user
ps -eo pid,ni,pri,cmd | head
ionice -c2 -n7 -p 4821               # I/O priority, often more useful than CPU nice
```

In practice on modern systems, systemd cgroup limits (`CPUQuota`, `MemoryMax`, `IOWeight` in a unit or slice) are the better tool for containing a noisy process — worth mentioning as the modern answer.

**Q: How do you keep a job running after you log out?**

```bash
nohup ./longjob.sh > /tmp/longjob.log 2>&1 &     # immune to SIGHUP at logout
./longjob.sh & disown                            # detach an already-started job
jobs ; fg %1 ; bg %1                             # job control in the current shell

screen -S patching        # then Ctrl+A D to detach, screen -r patching to reattach
tmux new -s patching      # then Ctrl+B D to detach, tmux attach -t patching

setsid ./longjob.sh                              # fully detached session
systemd-run --unit=myjob --scope ./longjob.sh     # tracked by systemd, survives logout
```

For anything long-running on a production server — a patch run, a large copy, a database import — always use `screen` or `tmux`. If your SSH session drops mid-way through `yum update`, you can end up with a half-patched system, and a detached session protects you from that.

**Q: What is the difference between cron and at?**

`cron` runs jobs **repeatedly** on a schedule; `at` runs a job **once** at a future time.

```bash
crontab -e                     # edit your own
crontab -l -u appuser          # view another user's (as root)
crontab -r                     # remove - dangerous, no confirmation

# format:  minute hour day-of-month month day-of-week command
# 30 2 * * *        /usr/local/bin/backup.sh          -> every day at 02:30
# */15 * * * *      /usr/local/bin/check.sh           -> every 15 minutes
# 0 4 * * 0         /usr/local/bin/weekly.sh          -> Sundays at 04:00
# 0 0 1 * *         /usr/local/bin/monthly.sh         -> 1st of each month

echo "/usr/local/bin/onetime.sh" | at 22:00
at 10:00 tomorrow
atq ; atrm 3
```

System-wide cron lives in `/etc/crontab`, `/etc/cron.d/`, and the `/etc/cron.hourly|daily|weekly|monthly` directories. Access is controlled by `/etc/cron.allow` and `/etc/cron.deny`. Three troubleshooting points that come up constantly: cron runs with a **minimal environment and a bare PATH**, so always use absolute paths; cron does not source your `.bash_profile`, so export any variables the script needs inside the script; and always redirect output (`>> /var/log/myjob.log 2>&1`) or you will get mail instead of logs. Check `/var/log/cron` to confirm a job fired. The modern alternative is **systemd timers**, which you inspect with `systemctl list-timers` — they give you logging via journald, dependency handling, and randomized delays.

**Q: What is the OOM killer?**

When the system genuinely runs out of usable memory and cannot reclaim any more, the kernel must free memory or halt entirely, so it invokes the **Out Of Memory killer**. It scores every process — largely on memory footprint — and kills the one with the highest `oom_score` to save the system. The frustrating part is that "biggest memory user" is often the important database or application server, so the OOM killer frequently kills exactly the thing you cared about.

```bash
dmesg -T | grep -i -E "out of memory|killed process"
grep -i oom /var/log/messages
journalctl -k | grep -i oom
cat /proc/<pid>/oom_score
echo -1000 > /proc/<pid>/oom_score_adj        # exempt a critical process
```

In a unit file you can set `OOMScoreAdjust=-900`. But the real answer in an interview is that an OOM event is a symptom — you should find the leak or the misconfiguration (a JVM heap larger than physical RAM, a runaway worker pool, no swap configured), not just protect processes from being killed.

**Q: What is the difference between a process and a thread?**

A process has its own memory address space, file descriptors and resources. A thread is a unit of execution *inside* a process that shares that memory space with its sibling threads, so threads are cheaper to create and communicate faster, but a crash or memory corruption in one thread can take down the whole process. On Linux both are scheduled as tasks by the kernel. See threads with `ps -eLf`, `top -H`, or `cat /proc/<pid>/status | grep Threads`.

### Patching, packages and the kernel

**Q: Walk me through how you patch a production RHEL server.**

This question is really about process discipline, not commands. Structure your answer as before, during, after.

**Before:** confirm the approved change window and get sign-off from the application owner. Take a VM snapshot or confirm a current backup. Record the current state so you can compare and roll back — `uname -r`, `rpm -qa > /tmp/pkgs.before.txt`, `systemctl list-units --state=running > /tmp/svc.before.txt`, `df -h`, `mount > /tmp/mounts.before.txt`. Check there is free space in `/boot` (a full `/boot` is the classic reason a kernel update fails) and in `/var`. Verify the subscription or repository is reachable with `subscription-manager status` and `yum repolist`. Stop the application and monitoring alerts cleanly if required.

**During:** run inside `screen` or `tmux` so a dropped SSH session cannot leave you half-patched.

```bash
yum check-update                       # or dnf
yum update --security                  # security-only patching
yum update                             # full update
yum history                            # note the transaction ID for rollback
reboot                                 # required for kernel, glibc, systemd updates
```

**After:** verify `uname -r` shows the new kernel, `systemctl --failed` is empty, all filesystems from the before-list are mounted, the application starts and responds, and `dmesg`/`journalctl -p err -b` is clean. Compare the package list. Then hand over to the application team for their validation and close the change record.

**Rollback:** `yum history undo <id>` reverts the transaction, `grubby --set-default` boots the previous kernel, or you restore the snapshot. Mention that in a mature environment patching is driven by **Satellite, Ansible, or Red Hat Insights** across a fleet rather than server by server, and that you always patch dev, then QA, then production.

**Q: How do you find which package a file belongs to, or what a package installed?**

```bash
rpm -qf /usr/sbin/sshd            # which package owns this file
rpm -ql openssh-server            # list all files in a package
rpm -qa | grep ssh                # is it installed
rpm -qi openssh-server            # package info, vendor, install date
rpm -qc httpd                     # just the config files
rpm -qd httpd                     # just the documentation
rpm -q --changelog kernel | head  # what changed, useful for CVE verification
rpm -V httpd                      # verify files against the RPM database - detects tampering
rpm -qa --last | head             # recently installed packages, great for "what changed?"
yum provides */sshd               # which package would provide a file you do not have
yum deplist httpd
yum history info <id>
```

`rpm -V` is a good one to volunteer: it compares size, permissions, checksum and ownership against the database, so it tells you whether someone modified a binary or config outside of package management.

**Q: How do you boot an older kernel permanently?**

```bash
grubby --info=ALL                              # list all boot entries with their indexes
grubby --default-kernel                        # what boots now
grubby --set-default=/boot/vmlinuz-4.18.0-425.el8.x86_64
grub2-editenv list                             # confirm saved_entry
rpm -qa kernel                                 # which kernels are installed
```

Alternatively set `GRUB_DEFAULT=saved` in `/etc/default/grub` and use `grub2-set-default`. Keep at least the previous working kernel installed — `installonly_limit=3` in `/etc/yum.conf` controls how many are retained — because "boot the previous kernel" is your fastest recovery from a bad kernel update.

**Q: How do you check what changed on a server recently?**

```bash
rpm -qa --last | head -30                      # package changes
yum history                                    # transactions with dates and users
find /etc -mtime -7 -type f -ls                # config files changed in the last week
last -20                                       # who logged in
journalctl --since "3 days ago" -p warning
ausearch -ts recent -m CONFIG_CHANGE
```

This is the question behind most "it was working yesterday" incidents, so having a crisp answer is valuable.

### Miscellaneous rapid-fire

- **`/proc` vs `/sys`** — `/proc` is a virtual filesystem exposing per-process information (`/proc/<pid>/`) and kernel/system state (`/proc/meminfo`, `/proc/cpuinfo`, `/proc/loadavg`). `/sys` (sysfs) exposes the kernel's device and driver model — you use it to rescan SCSI buses, check block-device queues, and read hardware attributes. Both live only in memory; nothing on disk.
- **`/etc/hosts` vs DNS order** — controlled by `/etc/nsswitch.conf` (`hosts: files dns`), so a stale `/etc/hosts` entry silently wins over correct DNS. Always check it when a name resolves to the wrong IP.
- **Runlevels vs targets** — 0 = `poweroff.target`, 1 = `rescue.target`, 3 = `multi-user.target`, 5 = `graphical.target`, 6 = `reboot.target`. `systemctl get-default` / `set-default` / `isolate`.
- **Hard vs soft NFS mount** — `hard` retries forever, so data is never silently lost but a process can hang indefinitely if the server disappears (this is the default and the right choice for important data). `soft` gives up after `timeo`/`retrans` and returns an I/O error, which keeps the application alive but risks data loss. Add `intr` on older systems so hung processes can be interrupted.
- **`tar` and backups** — `tar -czvf backup.tar.gz /data` to create, `-xzvf` to extract, `-tzvf` to list without extracting, `--exclude=` to skip paths, `-C /target` to extract elsewhere. For filesystem-level copies use `rsync -avz --delete src/ dst/` and mention `rsync -n` for a dry run.
- **Log rotation** — `/etc/logrotate.conf` plus per-application files in `/etc/logrotate.d/`. Key directives: `daily|weekly`, `rotate 14`, `compress`, `missingok`, `notifempty`, `copytruncate` (for apps that will not release the file handle), and `postrotate` to signal the daemon. Test safely with `logrotate -d /etc/logrotate.conf` (debug, no changes) and force with `logrotate -f`. Unrotated logs are the number one cause of a full `/var`.
- **Time synchronization** — `chronyd` replaced `ntpd` in RHEL 7+. `chronyc sources -v`, `chronyc tracking`, `chronyc makestep` to force a step correction, `timedatectl` for the overview, `timedatectl set-timezone Asia/Kolkata`. Emphasize that clock skew breaks Kerberos and Active Directory authentication (default tolerance is five minutes), invalidates TLS certificates, and makes cross-server log correlation useless.
- **Find large files and directories** — `du -xh / --max-depth=2 | sort -rh | head -20` for directories, `find / -xdev -type f -size +1G -exec ls -lh {} \;` for files. The `-xdev` flag keeps `find` on one filesystem so it does not wander into NFS mounts or `/proc`.
- **Find recently modified files** — `find /etc -mtime -1 -type f` for the last 24 hours, `-mmin -30` for the last 30 minutes, `-newer /path/reference` relative to another file's timestamp.
- **Check server hardware and OS details** — `cat /etc/redhat-release`, `uname -a`, `hostnamectl`, `lscpu`, `free -h`, `lsblk`, `lspci`, `lsusb`, `dmidecode -t system -t memory -t bios`, `lshw -short`, `ip a`, `ethtool eth0`, and `sosreport` when you need a full bundle for a vendor case.
- **Check uptime and last reboot** — `uptime`, `who -b`, `last reboot | head`, `journalctl --list-boots`.
- **Copy files between servers** — `scp file user@host:/path`, `rsync -avz --progress` (resumable and far better for large or repeated transfers), `sftp`. Mention `rsync --bwlimit` when you must not saturate the link during business hours.
- **Check open files limit** — `ulimit -n` for the current shell, `/etc/security/limits.conf` and `/etc/security/limits.d/` for persistent per-user limits, `LimitNOFILE=` in a systemd unit for services (limits.conf does **not** apply to systemd services — a very common gotcha), and `cat /proc/<pid>/limits` to see what a running process actually has.
- **Compare files and directories** — `diff -u a b`, `diff -r dir1 dir2`, `vimdiff`, `md5sum`/`sha256sum` to verify a transfer, `cmp` for binaries.
- **Useful text-processing one-liners** — `grep -ri pattern /etc`, `awk '{print $1}' file | sort | uniq -c | sort -rn` (top talkers in a log), `sed -i 's/old/new/g' file`, `cut -d: -f1 /etc/passwd`, `tail -f /var/log/messages | grep -i error`, `wc -l`.

### Scenario and experience questions

These come at the end of the interview and are scored on judgement and communication rather than command recall. Prepare a real example for each from your own work.

**Q: Tell me about the most difficult production issue you have handled.**

Use a simple structure: what the symptom and business impact were, how you narrowed it down, what the root cause turned out to be, what you did to restore service, and what permanent fix or monitoring you added afterwards. Keep the timeline clear and quantify the impact if you can ("about 200 users could not log in for 40 minutes"). Interviewers care most about the last part — the preventive action — because it separates someone who fixes symptoms from someone who closes issues out.

**Q: A user asks you to delete a large file to free space. What do you do?**

You do not delete it. You verify who owns the data and get confirmation from the application owner, check whether the file is still open by a running process (`lsof`), confirm there is a backup, and prefer archiving or compressing over deleting. If it is an active log, you truncate it rather than removing it, because deleting an open log frees nothing and can break the writing application. And if the growth is legitimate, the right answer is extending the filesystem through a change request, not repeatedly deleting files.

**Q: You are asked to make a change on a production server during business hours. How do you handle it?**

Push back politely and follow process: is there an approved change record, an agreed window, a rollback plan, and an application owner available to validate afterwards? If it is a genuine emergency, get verbal approval from the right authority, take a snapshot or backup first, document what you did as you do it, and raise the retrospective change record. The answer the interviewer wants to hear is that you never make unlogged changes to production.

**Q: How do you hand over an unresolved issue at the end of your shift?**

Document the symptom, the exact impact and start time, everything you have already checked and ruled out, the current state of the system (what you changed, what is still running), the next planned step, and any relevant ticket or vendor case number. Then brief the next engineer verbally as well. Vague handovers are the main reason incidents drag across shifts.

**Q: How do you keep your skills current?**

Mention concrete sources: Red Hat documentation and the Knowledgebase, release notes for each major version, a home lab or VMs for testing, certification tracks (RHCSA/RHCE), and following the changes in the version your organization is migrating to next. Naming something specific and recent — image mode in RHEL 10, or the RHEL 9 crypto policy changes — shows you actually do it.

---

## How to Answer in the Interview

- **For knowledge questions:** definition → why it matters → the commands → a real example you handled.
- **For troubleshooting questions:** use the seven-part structure from the top of this document — symptom, triage, ranked causes, diagnosis, temporary fix, permanent fix, prevention. Say the section names out loud as you go ("first I'd confirm the symptom, then my 90-second triage would be…"). It makes you sound organized and it stops you from rambling, which is the main way candidates lose these questions.
- **Lead with the ranked cause list.** Saying "in my experience this is usually one of five things, most likely first…" immediately signals experience, because it shows you troubleshoot in probability order rather than trying everything.
- **Always separate temporary from permanent.** Restore service first, then fix the root cause. Interviewers listen specifically for this distinction.
- **Always mention capturing evidence before rebooting** — `sosreport`, kdump vmcore, SysRq dumps, copying the relevant logs. A reboot that fixes the symptom and destroys the cause guarantees a repeat incident.
- **Always mention backup or snapshot, application-owner confirmation, and a change request** before disruptive actions. This is scored as heavily as the command knowledge, and it is the difference between sounding like an operator and sounding like an engineer.
- **Quantify when you can.** "Load average was 180 on an 8-core box with the CPU 95% idle, which told me it was I/O" is far stronger than "the server was slow."
- **If you don't know something, say how you would find out:** `man`, `--help`, `/usr/share/doc`, the Red Hat Knowledgebase and documentation, `sosreport` for a vendor case, and the vendor's own support. Never invent a command; an honest, methodical answer scores better than a confident wrong one.
