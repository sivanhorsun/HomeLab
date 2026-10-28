# Phase 9 — VM Permissions, PHP Review, and SSH Hardening

**Result:** I gave the collaborator scoped Proxmox permissions to create and manage lab VMs. The task history later showed VM 101 being created and operated through that account. I checked Ubuntu, the Pi, and the Proxmox host for PHP packages and running services, then updated seven PHP packages on the Pi. Finally, I disabled password and keyboard-interactive SSH authentication on my Ubuntu VM, prohibited direct root login, and verified a fresh key login alongside rejection of a password-only attempt.

> **Historical scope:** October 22–28, 2026, following Phase 8. The SSH change applies to **VM 100 only**. VM 101's guest installation was not verified. The lab retained its shared LAN, and no Tailscale policy change was made during this phase. Versions, resource figures, and listener lists are observations from the recorded sessions.

Previous phase: [Shared Remote Access and the First Ubuntu VM](08-shared-remote-access-and-ubuntu-vm.md). Project overview: [README](../README.md).

> **Publication convention:** LAN addresses are retained. Tailscale addresses remain illustrative: Pi `100.101.10.10`, MacBook `100.101.10.20`, Proxmox `100.101.10.30`, and Ubuntu `100.101.10.40`. `vm-admin`, `lab-user@pve`, and `collaborator-lab` replace personal account and pool names. Actual overlay identifiers, fingerprints, personal tailnet names, passwords, and keys are omitted. Commands record completed work; examples require the real account and destination before use.

## 1. Objective and starting point

Phase 8 ended with Ubuntu running as VM 100, the MacBook's public key installed, a fresh key login tested, and `sudo` verified. Password authentication still worked. A separate collaborator Proxmox account existed, but its ability to create VMs had not yet been established.

This phase addressed three practical tasks: delegate VM work without sharing root, review PHP exposure across the existing systems, and finish the Ubuntu SSH authentication change.

## 2. Work sequence

| When | Work recorded |
| --- | --- |
| October 22 | Created the collaborator pool; assigned pool, storage, and network roles; checked Create VM through the collaborator account |
| After the permission change | Inspected Ubuntu's SSH configuration and the cloud-init password setting; tested the existing key |
| October 28 | Checked packages, processes, and TCP listeners on Ubuntu, Kali, and the Proxmox host; updated PHP on Kali |
| October 28 | Reviewed the October 22 task history and VM 101's hardware and stopped state |
| October 28, final session | Applied the Ubuntu SSH configuration, reloaded the service, and tested key success and password rejection |

## 3. Temporary remote-access interruption

Before continuing the Proxmox work, access to its Tailscale address timed out from work. A TCP connection test to port 8006 also timed out. Access later worked at home, with a brief Tailscale interruption reported before the connection recovered.

The cause was not established. There was no confirmed diagnosis of a workplace restriction, host failure, or firewall error, and no network change was recorded as the fix. Phase 8's successful hotspot test remains a valid observation, but neither that test nor the later recovery establishes continuous availability.

## 4. Creating the collaborator pool

The existing account, represented here as `lab-user@pve`, needed to create and manage VMs without becoming a host administrator. I created a pool, represented here as `collaborator-lab`, and configured permissions through **Datacenter → Permissions** while using the administrator account.

A pool is a logical grouping of Proxmox resources. It is not another computer, another guest OS, or a network segment. Creating it alone does not grant access: the account needs suitable roles at the relevant paths.

## 5. Recorded permission assignments

All entries below were assigned to `lab-user@pve`, with **Propagate** enabled. Account and pool names are normalized for publication; the roles and path structure follow the recorded setup.

| Path | Role | Purpose in this setup |
| --- | --- | --- |
| `/pool/collaborator-lab` | `PVEVMAdmin` | VM operations within the pool's scope |
| `/pool/collaborator-lab` | `PVEPoolUser` | Pool use within the assigned scope |
| `/storage/local-lvm` | `PVEDatastoreUser` | Allocate guest disk storage |
| `/storage/local` | `PVEAuditor` | Inspect the storage and existing content |
| `/sdn/zones/localnetwork/vmbr0` | `PVESDNUser` | Use the existing `vmbr0` network |

Propagate extends an assignment through the applicable permission hierarchy; it does not make every user an administrator. Storage remained shared, and these assignments did not create a personal storage quota.

## 6. Narrowing the network permission

The network permission was initially assigned at the broader `/sdn/zones/localnetwork` path. It was then changed to `/sdn/zones/localnetwork/vmbr0`, and removal of the broader assignment was confirmed.

No global `Administrator` role or host root account was granted. The table records the configuration that was applied and confirmed, not a complete audit of every effective permission.

## 7. How the access layers fit together

| Layer | What it controls |
| --- | --- |
| Tailscale grant | Whether the remote client can reach Proxmox TCP 8006 |
| Proxmox account and realm | Which identity signs in to the interface |
| ACL path and role | Which management operations that identity can perform, and where |
| Resource pool | How resources are grouped for management and permission assignment |
| Guest Linux account | Permissions inside a VM's operating system |

The collaborator selects **Proxmox VE authentication server** for the `@pve` account. The administrator's existing `root@pam` account uses **Linux PAM standard authentication**. Neither a Tailscale connection nor a Proxmox account automatically grants access to a Linux account inside a guest.

## 8. Permission validation

I checked the collaborator account in a separate browser session and confirmed that **Create VM** was available. The collaborator was unavailable at that moment, so an enabled button was not treated as proof of a completed VM deployment.

On October 28, the task history showed VM 101 being created and operated under the collaborator's account on October 22. That provided evidence of actual account activity beyond the initial UI check.

## 9. Boundaries not established by the pool

No per-user RAM, CPU, or disk quota was configured. No new VLAN or internal network isolation was created. Guests attached to `vmbr0` still shared the lab LAN.

A deliberate denial test against VM 100 was not completed. The pool setup should therefore not be described as a fully tested security boundary for every operation. Pool permissions, Tailscale policy, and guest OS permissions remain separate controls.

## 10. Ubuntu SSH settings before the change

Inside **VM 100**, I inspected the server configuration:

```bash
sudo /usr/sbin/sshd -T
```

| Setting | Observed value before hardening |
| --- | --- |
| `PubkeyAuthentication` | `yes` |
| `PasswordAuthentication` | `yes` |
| `KbdInteractiveAuthentication` | `no` |
| `PermitRootLogin` | `without-password` |
| `X11Forwarding` | `yes` |
| `AllowAgentForwarding` | `yes` |
| `AllowTcpForwarding` | `yes` |

`without-password` did not mean unauthenticated root access. It excluded root password authentication while potentially allowing an appropriate public key. The output contained configuration values and file paths, not private-key contents or passwords.

## 11. Finding the password setting

Inside **Ubuntu**:

```bash
ls -l /etc/ssh/sshd_config.d/
sudo cat /etc/ssh/sshd_config.d/50-cloud-init.conf
```

The directory contained `50-cloud-init.conf`, owned by root with owner-only read/write permissions. Its content was:

```text
PasswordAuthentication yes
```

The later change used a separate local configuration file and checked the effective result. The cloud-init file was not deleted.

## 12. Key passphrase versus account password

The MacBook's passphrase unlocks its private key locally. It is different from the Ubuntu account password, and it is not sent to Ubuntu as a login password. The public key had already been installed in Phase 8; the private key stayed on the MacBook.

Before changing server authentication, I tested a connection that selected that key and disabled password fallback on the client. Successful key login was a prerequisite for the server change, not evidence that the server had already stopped accepting passwords.

## 13. Scope of the PHP review

Discussion of **CVE-2026-109618** and **CVE-2026-109619** prompted a check for installed PHP and running PHP-related services. The systems checked separately were Ubuntu VM 100, the Kali Pi, and the Proxmox host.

This was a package/process/listener review, not an exploit test or malware investigation. It did not inspect VM 101's guest, comprehensively enumerate containers, or rule out custom installations outside the package manager.

## 14. Ubuntu package, process, and listener checks

The package search was:

```bash
dpkg -l | grep -E '^ii[[:space:]]+(php|libapache2-mod-php)'
```

It returned no matches. The initial process search used `ps` and `grep`; its only result was the search command itself, which matched the `php` substring in the supplied `frankenphp` pattern. It did not establish a running PHP service.

TCP listeners were inspected with:

```bash
sudo ss -lntp
```

| Listener | Observed process and scope |
| --- | --- |
| TCP 22 | `sshd`, IPv4 and IPv6 |
| TCP 53 | `systemd-resolved` on local loopback addresses |
| TCP 58237 / 52029 | `tailscaled`, bound to the Tailscale addresses |

No PHP web service appeared in that output.

## 15. Initial findings on the Kali Pi

The same package search found PHP `8.4.24-1`, including the Apache PHP module. The installed components included common files, CLI, MySQL support, OPcache, readline, and the `php8.4` metapackage.

```bash
systemctl is-active apache2
pgrep -a -f 'php|apache2|nginx|frankenphp|litespeed'
sudo ss -lntp
```

Apache reported `inactive`. After correcting the spelling of `frankenphp`, the process search returned no matches. The TCP listener list showed SSH and Tailscale only, including Tailscale listeners on 54129 and 60967.

No active PHP service was observed, but installed packages could still be used later. The package update was handled separately from whether Apache was currently running.

## 16. Checking the available PHP update

Inside **Kali on the Pi**:

```bash
sudo apt update
apt-cache policy php8.4-common libapache2-mod-php8.4
```

The refresh reported 537 upgradable packages. It updated package indexes rather than installing those upgrades.

| Package | Installed | Candidate |
| --- | --- | --- |
| `php8.4-common` | `8.4.24-1` | `8.4.27-1` |
| `libapache2-mod-php8.4` | `8.4.24-1` | `8.4.27-1` |

The candidate packages came from `kali-rolling/main` for ARM64. These are historical package observations, not a recommendation to pin future installations to this version.

## 17. Simulating the targeted upgrade

```bash
apt -s install --only-upgrade php8.4-common \
  libapache2-mod-php8.4 php8.4-cli php8.4-mysql \
  php8.4-opcache php8.4-readline php8.4
```

The simulation reported:

```text
Upgrading: 7, Installing: 0, Removing: 0, Not Upgrading: 530
```

All seven PHP packages were scheduled to move from `8.4.24-1` to `8.4.27-1`, without additional packages or removals. This was a limited package update, not a full Kali upgrade.

## 18. Applying and checking the PHP update

After reviewing the simulation, I ran:

```bash
sudo apt install --only-upgrade php8.4-common \
  libapache2-mod-php8.4 php8.4-cli php8.4-mysql \
  php8.4-opcache php8.4-readline php8.4
```

The installation completed and returned to the prompt. The follow-up checks were:

```bash
php -v
systemctl is-active apache2
```

I confirmed the expected results: PHP `8.4.27` and Apache still `inactive`. A fresh `dpkg` listing for all seven packages was not captured after installation. No reboot was recorded for this update, and the other 530 upgrades in the plan were outside its scope.

## 19. Checking the Proxmox host separately

Inside the **Proxmox node's Shell**, as the host administrator:

```bash
dpkg -l | grep -E '^ii[[:space:]]+(php|libapache2-mod-php)'
pgrep -a -f 'php|apache2|nginx|frankenphp|litespeed'
ss -lntp
```

The package and process searches returned no matches. A host check does not inspect the operating systems inside its VMs.

## 20. Interpreting the Proxmox TCP listeners

| Port | Process | Observed purpose or binding |
| --- | --- | --- |
| 8006 | `pveproxy` | Proxmox HTTPS management |
| 3128 | `spiceproxy` | SPICE console proxy |
| 22 | `sshd` | Host SSH listener, IPv4 and IPv6 |
| 111 | `rpcbind` | RPC service mapping |
| 85 | `pvedaemon` | Internal API listener on `127.0.0.1` |
| 25 | `master` | Mail service, loopback only |
| 58986 / 46926 | `tailscaled` | Tailscale-address listeners |

The listing showed no PHP service. A listener on `0.0.0.0` or `*` does not by itself prove internet reachability; routing and access controls still apply. No services were disabled because of this inventory, and the Tailscale policy was not expanded to allow host SSH.

## 21. Reviewing the collaborator's task history

I used **Pool View** to inspect assigned resources, then **Server View → proxmox → Tasks** to review management activity. The history showed these October 22 operations on VM 101 under the collaborator's account:

| Displayed time | Recorded activity |
| --- | --- |
| 19:22 | Create completed with `OK` |
| 19:23 onward | Start and console access succeeded |
| 19:42–19:44 | Shutdown attempts failed; a lock error and an already-running Start result also appeared |
| 19:44 onward | Stop succeeded, followed by another Start at 19:46 |
| 20:05 | Shutdown completed with `OK` |

Tasks records Proxmox management operations, not the full command history inside a guest. The error messages were truncated in the screenshot and their detailed logs were not opened, so a definite cause was not established.

## 22. VM 101 hardware and observed state

| Setting | Observation on October 28 |
| --- | --- |
| VM ID / name | `101`; default display name |
| State | `stopped`; 0 B RAM in use |
| Allocated memory | 2 GiB |
| CPU | 1 socket, 1 core; `x86-64-v2-AES` |
| Machine / BIOS | i440fx / SeaBIOS |
| Disk | 32 GiB, `scsi0`, `local-lvm`; IO Thread enabled |
| SCSI controller | VirtIO SCSI single |
| Network | VirtIO on `vmbr0`; `firewall=1` |
| Installation media | `ubuntu-24.04.5-live-server-amd64.iso` still attached |
| Guest Agent | `No Guest Agent configured` |

The guest was not started for this inspection. An attached ISO does not prove either a completed or incomplete installation. The virtual NIC's firewall flag is also not proof of an effective filtering policy. Ubuntu installation, guest identity, updates, and remote access inside VM 101 remain to be checked.

## 23. Testing the Ubuntu key before hardening

From the **MacBook's local Terminal**, with Tailscale connected:

```bash
ssh -i ~/.ssh/id_ed25519_lab -o IdentitiesOnly=yes \
  -o PasswordAuthentication=no \
  -o KbdInteractiveAuthentication=no vm-admin@100.101.10.40
```

The login succeeded. The Ubuntu banner reported `24.04.5 LTS`, kernel `6.8.0-147-generic`, LAN address `192.168.88.249`, and nine available updates. Those updates and the offered distribution upgrade were not applied during the SSH change.

## 24. Creating the local SSH configuration

Inside **Ubuntu VM 100**:

```bash
sudo nano /etc/ssh/sshd_config.d/00-lab-hardening.conf
```

I saved these three lines in the same file:

```text
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
```

The local `00-` file was used ahead of the existing `50-cloud-init.conf`; the effective values were checked afterward. In nano, **Ctrl + O**, then **Enter** saved the file, and **Ctrl + X** exited. The Linux account password was not removed or changed.

## 25. Validating the configuration before reloading

The existing working session stayed open while I checked syntax:

```bash
sudo /usr/sbin/sshd -t
```

The command returned to the prompt without an error. At this point the file had been saved and checked; that alone did not load it into the running service.

## 26. Checking effective values and reloading SSH

```bash
sudo /usr/sbin/sshd -T | grep -E \
  '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin) '
```

The output was:

```text
permitrootlogin no
passwordauthentication no
kbdinteractiveauthentication no
```

I then reloaded the service:

```bash
sudo systemctl reload ssh
```

The command returned without an error. The change did not modify Tailscale policy or forwarding settings, and it did not apply to the Pi or the Proxmox host.

## 27. Positive and negative login tests

In a new **MacBook** tab, I repeated the explicit key-only command from section 23. The new connection successfully opened Ubuntu after the reload.

A separate test disabled public-key authentication and requested a password-only attempt:

```bash
ssh -o PubkeyAuthentication=no \
  -o PreferredAuthentications=password \
  -o NumberOfPasswordPrompts=1 vm-admin@100.101.10.40
```

I reported `Permission denied`. Combined with the effective configuration and successful new key connection, this verified the intended outcome for my account from the MacBook.

| Check | Result |
| --- | --- |
| `sshd -t` | No configuration error reported |
| Effective settings | All three reviewed settings were `no` |
| Service reload | No error reported |
| New connection using the key | Succeeded after reload |
| Password-only attempt | Rejected |
| Direct root login | Prohibited in configuration; no separate root login test performed |

This was not a test of every possible user or `Match` condition. The Linux password remains relevant to `sudo` and console access; a new console-login test was not part of this change. X11, agent, and TCP forwarding settings were not changed.

## 28. Command-entry and environment lessons

| Issue | Explanation and correction |
| --- | --- |
| `ssh -I` used with the private-key file | Uppercase `-I` selects a PKCS#11 provider. Lowercase `-i` selects an identity file. The provider error did not mean the key needed to be recreated. |
| Key path missing under `/home/.../.ssh/` | The client command was entered inside Ubuntu instead of on the MacBook. That attempt was cancelled and retried on the machine holding the private key. |
| `quote>` prompt | An unmatched quote left the command incomplete; it was cancelled with Ctrl + C and retyped. |
| Mistyped paths/options | `/ect`, `.comf`, and `NumberOfPasswordPromts` were corrected to `/etc`, `.conf`, and `NumberOfPasswordPrompts`. |
| Command returned without text | Meaning depends on the command: no search matches, successful syntax validation, or a reload without an error. |

We also discussed inspecting commands before pasting and the purpose of `sudo`. A privileged read-only inspection does not become a configuration change merely because it uses `sudo`. No private key was copied to a server.

After the new-login tests, SSH sessions could be ended with `exit`. Disconnecting a session does not shut down its VM or host.

## 29. End-of-phase state

- [x] Create the collaborator pool and assign scoped Proxmox roles.
- [x] Confirm Create VM availability; later observe actual VM 101 creation and operation in Tasks.
- [x] Inspect SSH configuration and identify the existing password-authentication setting.
- [x] Review PHP packages, processes, and TCP listeners on Ubuntu, the Pi, and Proxmox.
- [x] Simulate and apply the seven-package PHP update on the Pi; confirm the CLI version and inactive Apache state.
- [x] Inspect VM 101's hardware and stopped state without starting or changing it.
- [x] Configure Ubuntu VM 100's SSH authentication, validate it, and reload the service.
- [x] Verify a fresh key login and rejection of a password-only attempt.

No VM backup or restore test was completed. No equivalent SSH change was applied to the Pi, Proxmox host, or VM 101. The shared LAN and previously documented Tailscale grants remained unchanged. These observations are not a malware-clearance certificate or a comprehensive vulnerability assessment.

## 30. Next work

1. Create a backup of VM 100 on a suitable destination and verify restoration. Keep a secure backup of the MacBook key and important configurations; the key passphrase alone cannot recreate a lost private key.
2. Continue with the collaborator to establish VM 101's actual guest installation state, hostname, updates, Guest Agent, and intended remote access. Creating a VM does not automatically enroll it in Tailscale.
3. Plan the remaining Kali updates and the nine Ubuntu updates reported in this session, with dependency and resource checks before a larger change.
4. Validate that the collaborator's permissions exclude unintended resources, including VM 100. Design a suitable network boundary before introducing intentionally vulnerable guests.
5. Carry forward the unresolved switch review, post-change MikroTik export, and previous-owner data-backup tasks from earlier phases.

## Address and access reference

| Endpoint | Public example or identifier | Access recorded |
| --- | --- | --- |
| Ubuntu | VM 100 / `100.101.10.40` | `vm-admin`, SSH with the MacBook key |
| Kali Pi | `100.101.10.10` | Existing OpenSSH access; no key-only change in this phase |
| Proxmox | `https://100.101.10.30:8006/` | HTTPS through Tailscale and a permitted Proxmox account |
| Collaborator guest | VM 101 | Proxmox management; guest address and remote access not established |
| Collaborator pool | `collaborator-lab` | Logical resource grouping in Pool View |

## Technical references

- [PHP release and security announcements](https://www.php.net/)
- [Ubuntu Server OpenSSH documentation](https://ubuntu.com/server/docs/how-to/security/openssh-server/)

The commands and outcomes above come from the recorded lab sessions. Package versions and PHP findings describe that limited review; they are not a claim that every vulnerability or execution path was tested. Raw screenshots, authentication material, and private configuration exports are not included in this public chapter.
