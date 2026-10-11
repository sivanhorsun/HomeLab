# Phase 6 — Raspberry Pi, Tailscale, and Remote Access

**Result:** I identified the headless Raspberry Pi, verified its Kali installation, moved its existing Tailscale enrollment into my own account, and connected over SSH from a phone hotspot. I replaced the broad tailnet policy with a rule allowing my MacBook to reach the Pi on TCP 22, changed the administrator password, prepared a separate collaborator account, and completed a large Kali upgrade. After reboot, SSH and Tailscale were active and no failed systemd units were reported.

> **Historical scope:** Work performed on October 10, 2026. Remote access was verified to the Pi itself; no subnet router or remote route to the entire lab was configured. The collaborator's Linux account was tested from my MacBook, but the collaborator had not yet received Tailscale access.

Previous phase: [Lab Isolation and Firewall Troubleshooting](05-lab-isolation-and-firewall.md). Project overview: [README](../README.md).

> **Publication convention:** Private LAN addresses are retained. Tailscale addresses below are illustrative replacements: `100.101.10.10` represents the Pi and `100.101.10.20` represents the MacBook. `admin-mac` is a normalized policy alias, and `lab-user` replaces the collaborator's actual Linux username. Account emails, MAC addresses, and host-key fingerprints are omitted. Substitute real values before using examples; these are historical records, not a deployment script.

## 1. Objective and starting point

The next milestone after the MikroTik isolation work was remote administration. The original plan was to find the Pi on the lab network, connect with SSH, and install Tailscale. The checks showed that both Kali and Tailscale were already installed by my collaborator, so the work changed to verifying the installation, transferring the enrollment, and restricting access.

The existing lab stayed on one `192.168.88.0/24` LAN. The MikroTik remained the gateway, DHCP server, and internet boundary. No RouterOS firewall rule, switch VLAN, or bridge setting was changed during this phase.

| Component | Starting role / connection |
| --- | --- |
| MikroTik hAP ax³ | Lab gateway at `192.168.88.1`; existing wireless upstream |
| TP-Link TL-SG3210 | Managed switch; Port 1 connected to MikroTik `ether5` |
| N100 / Proxmox | Existing server at `192.168.88.2`, on Switch Port 3; not reconfigured |
| Raspberry Pi | Headless Kali client, connected to **Switch Port 8** |
| MacBook | Wired to the lab initially; later disconnected for external-access testing |

Port 8 was the Pi connection confirmed in this session. The earlier proposal to use Port 2 was not the actual mapping used here.

## 2. Discovering the Pi through DHCP

From the **MikroTik terminal**:

```routeros
/ip dhcp-server lease print detail without-paging
```

The lease table was enough to identify the Pi, so no port scan or Nmap discovery was necessary. This was a read-only check and did not require Safe Mode.

| Device | Observed LAN address | Lease evidence |
| --- | --- | --- |
| TP-Link switch | `192.168.88.252` | Hostname `SG3210`; class ID `SG3210_3.20` |
| MacBook | `192.168.88.251` | Management laptop's Ethernet lease |
| Raspberry Pi | `192.168.88.249` | Hostname `kali-raspberrypi` |

All three entries were dynamic leases with `status=bound` from DHCP server `defconf`. The active-address and active-MAC fields identified the current lease associations. These addresses were not converted into DHCP reservations and may change.

## 3. First SSH attempt and retry

From the **MacBook's local Terminal**:

```bash
ssh kali@192.168.88.249
```

The first connection displayed an unknown-host prompt. After I entered `yes`, the host key was saved in `known_hosts`, but the server closed the connection before a password prompt appeared.

I retried with diagnostic output:

```bash
ssh -v kali@192.168.88.249
```

This time the connection succeeded. The output showed:

- TCP port 22 was reachable.
- The server's ED25519 key matched the key just saved.
- The server offered `publickey,password` authentication.
- Password authentication succeeded and opened a shell as `kali`.

The original early closure was not conclusively diagnosed. Enabling verbose output provided evidence; it did not itself repair a configuration. No firewall or SSH service change was needed for the successful retry.

An independent verification of the first host-key fingerprint against a local console or the collaborator's record was not documented. Later, the Tailscale-address connection presented the same key as the LAN connection, providing continuity between those two addresses.

## 4. OS, interface, and route checks

Inside the **Pi's SSH session**:

```bash
cat /etc/os-release
date
ip -br addr
ip route
```

| Check | Recorded result |
| --- | --- |
| Distribution | Kali GNU/Linux Rolling, `VERSION_ID=2026.2`, `kali-rolling` |
| Architecture | `aarch64` / ARM64 |
| Running kernel | `6.12.34+rpt-rpi-2712` |
| Clock output | October 10, 2026, `21:03:34 UTC` |
| Ethernet | `eth0` UP, `192.168.88.249/24` |
| Wi-Fi | `wlan0` DOWN; no active address shown |
| Default route | Via `192.168.88.1`, on `eth0`, supplied by DHCP |
| VPN interface | `tailscale0` already existed with an assigned address |

These commands inspected state without changing it. An old `Last login` entry from a household address described a previous session, not proof of a current direct household-network connection.

## 5. Discovering and transferring the existing Tailscale enrollment

```bash
tailscale status
```

The Pi was already enrolled in my collaborator's tailnet. Another laptop was listed as offline, last seen 17 days earlier. That second device was not changed.

The decision was to manage the lab endpoint through my own Tailscale account. While connected to the Pi over its **LAN address**, I ran:

```bash
sudo tailscale logout
sudo tailscale up --reset
```

I opened the generated authentication link and completed sign-in with my own account. The success page and a subsequent status check confirmed that `kali-raspberrypi` belonged to the new tailnet. It received a new Tailscale address; its observed LAN address stayed `192.168.88.249`.

At one point I accidentally entered `kali` as a command. The shell offered to install a package with that name. That suggestion was unrelated to Tailscale setup, and no such installation was recorded.

The one-time authentication URL and account emails are not included here. The logout/reset commands document the migration; they do not need to be repeated to resume an already working connection.

## 6. MacBook enrollment and external-access test

I installed the macOS Tailscale client. The installer displayed version **1.104.1**. I approved the network extension and VPN configuration, then authenticated the MacBook into the same tailnet as the Pi.

From the **MacBook's local Terminal**, using the Pi's actual Tailscale address in the live session:

```bash
# Illustrative address; replace with the Pi's current Tailscale IP.
ssh kali@100.101.10.10
```

The host-key prompt identified the key as already known under `192.168.88.249`. Its ED25519 fingerprint matched the LAN connection. After approval and password entry, SSH opened a shell on the Pi.

I then closed the old SSH sessions with `exit`, unplugged the MacBook's Ethernet cable, joined my phone's hotspot, and opened a **new** SSH connection to the Pi's Tailscale address. It succeeded. A later test from the household Wi-Fi also succeeded.

| Test | Observed result |
| --- | --- |
| LAN SSH to `192.168.88.249` | Successful |
| SSH using the Pi's Tailscale address | Successful |
| Fresh SSH session from a phone hotspot, with Ethernet disconnected | Successful |
| Fresh SSH session from household Wi-Fi | Successful |

This verified remote access to the Pi without opening a public management port. No port-forward rule was created on either router. No route to `192.168.88.0/24` was advertised, and no subnet router was configured. Direct versus relayed Tailscale transport was not measured.

Separate Terminal tabs and colors helped distinguish the local MacBook shell from the remote Pi shell. `Command + T` opens another local tab; `exit` closes the current SSH shell and does not power off the Pi.

## 7. Checking SSH forwarding and listening services

Before granting another person access, I checked what the Pi could reach and what services were exposed.

### MacBook SSH client settings

From the **local macOS Terminal**:

```bash
# Illustrative address; use the actual Pi address for the real check.
ssh -G kali@100.101.10.10 |
  grep -E '^(forwardagent|forwardx11|localforward|remoteforward|dynamicforward) '
```

Recorded output:

```text
forwardx11 no
forwardagent no
```

No local, remote, or dynamic forwarding entries appeared in that filtered output. The inspected configuration did not forward my SSH agent or X11 session to the Pi. This was not a full audit of the MacBook or proof that every possible network path from the Pi was blocked.

### Pi listeners

Inside the **Pi session**:

```bash
sudo ss -tulpn
```

| Process | Observed listener | Interpretation |
| --- | --- | --- |
| `sshd` | TCP 22 on all IPv4 and IPv6 addresses | Ordinary OpenSSH server |
| `tailscaled` | UDP 41641 on IPv4 and IPv6 | Tailscale networking |
| `tailscaled` | TCP 54129 and 60967 on Tailscale addresses | Listeners owned by the Tailscale process at that time |
| `dhcpcd` | UDP 68 on the LAN address; UDP 546 on an IPv6 link-local address | DHCP configuration |

No HTTP or file-sharing listener was shown in the supplied snapshot. Dynamic Tailscale TCP ports were observations, not ports to configure manually. The scan covered listening sockets, not every process or outbound connection.

An encrypted SSH session alone does not determine whether the Pi can initiate a separate connection to the MacBook. The original tailnet policy allowed broad traffic, so access policy needed its own review.

## 8. Services and scheduled tasks

The active-service output contained 23 units:

| Group | Units observed; `.service` suffix omitted |
| --- | --- |
| Network and remote access | `NetworkManager`, `systemd-networkd`, `wpa_supplicant`, `ModemManager`, `ssh`, `tailscaled` |
| User and desktop environment | `accounts-daemon`, `lightdm`, `polkit`, `rtkit-daemon`, `bluetooth` |
| System infrastructure and logging | `cron`, `dbus`, `rsyslog`, `systemd-journald`, `systemd-logind`, `systemd-timesyncd`, `systemd-udevd`, `systemd-userdbd` |
| Consoles and user managers | `getty@tty1`, `serial-getty@ttyAMA10`, `user@1000`, `user@116` |

Seeing both NetworkManager and systemd-networkd did not establish a conflict. Interface ownership was not investigated and neither service was disabled on that basis.

Nine timers appeared: `phpsessionclean`, `dpkg-db-backup`, `logrotate`, `plocate-updatedb`, `e2scrub_all`, `man-db`, `systemd-tmpfiles-clean`, `fstrim`, and `ntpsec-rotate-stats`. The last had no scheduled time displayed.

The user-crontab checks returned:

```text
no crontab for kali
no crontab for root
```

That result concerned those users' personal crontabs. It did not rule out system-wide cron files or other persistence mechanisms. No unfamiliar entry was identified as malicious from these outputs, but this review was not a malware clearance or forensic examination.

## 9. Accounts, SSH keys, and administrator password

Before creating a collaborator account, I inspected root and ordinary user IDs:

```bash
getent passwd | awk -F: '
  $3 == 0 || ($3 >= 1000 && $3 < 65534) {
    print "user=" $1, "uid=" $3, "home=" $6, "shell=" $7
  }'
getent group sudo
```

| Account | UID | Home | Shell / group observation |
| --- | --- | --- | --- |
| `root` | `0` | `/root` | `/bin/bash` |
| `kali` | `1000` | `/home/kali` | `/usr/bin/zsh`; member of `sudo` |

The filter was not a list of every service account or a complete sudoers audit.

```bash
sudo ls -la /home/kali/.ssh /root/.ssh
```

Both directories had mode `700`. Each contained a zero-byte `authorized_keys` file with mode `600`; no `authorized_keys2` file appeared. Empty files alone did not cover every possible SSH key source, so I inspected the effective server configuration for `kali` and the MacBook's source address:

```bash
# The source address below is a publication example.
sudo /usr/sbin/sshd -T -C \
  user=kali,addr=100.101.10.20,host=100.101.10.20 |
  grep -E -e '^authorizedkeys(file|command) ' \
    -e '^(trustedusercakeys|passwordauthentication) ' \
    -e '^(pubkeyauthentication|permitrootlogin) '
```

Recorded effective settings:

```text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication yes
trustedusercakeys none
authorizedkeyscommand /usr/bin/userdbctl ssh-authorized-keys %u
authorizedkeysfile .ssh/authorized_keys .ssh/authorized_keys2
```

The configured external key lookup was also checked:

```bash
sudo /usr/bin/userdbctl ssh-authorized-keys kali
```

It returned no keys on two attempts. A root-specific effective-configuration check returned `permitrootlogin no`. Other possible `Match` contexts were not exhaustively reviewed.

I changed the `kali` password and verified it with a new login. Password authentication remained enabled; a key-only configuration was not implemented.

## 10. Replacing the broad tailnet policy

The original policy contained a grant permitting all sources, destinations, and IP traffic. I backed it up in Notes, then saved a plain-text copy as `tailscale-policy-backup.txt` before replacing the policy in Access Controls.

The saved policy allowed **one MacBook to initiate TCP 22 connections to the Pi**. Replies to that permitted connection remained allowed. A fresh SSH connection succeeded after saving.

The following preserves the policy structure, with normalized aliases and example IPs:

```json
{
  "hosts": {
    "admin-mac": "100.101.10.20",
    "lab-pi": "100.101.10.10"
  },
  "grants": [
    {
      "src": ["admin-mac"],
      "dst": ["lab-pi"],
      "ip": ["tcp:22"]
    }
  ],
  "ssh": [],
  "tests": [
    {
      "src": "admin-mac",
      "proto": "tcp",
      "accept": ["lab-pi:22"],
      "deny": ["lab-pi:80", "lab-pi:443"]
    },
    {
      "src": "lab-pi",
      "proto": "tcp",
      "deny": ["admin-mac:22", "admin-mac:445", "admin-mac:5900"]
    }
  ]
}
```

The editor displayed `Saved tailnet policy file`. The `tests` entries are assertions checked against policy, not separate firewall rules or recorded live connection attempts. Access not granted by this policy was denied within its tailnet scope. The `hosts` entries were policy aliases, not hostname changes or DHCP reservations.

The empty `ssh` list did not enable Tailscale SSH. The working method was **ordinary OpenSSH over Tailscale**, with Linux username/password authentication. Network permission to reach TCP 22 and permission to log into a Linux account remained separate decisions.

This policy did not replace the MikroTik firewall, block direct traffic on the shared LAN, or restrict the Pi's ordinary internet egress. No new local Pi firewall was configured. The policy tests were not a live reverse-connection test from the Pi to the MacBook.

## 11. Preparing a separate collaborator account

On the Pi, a separate Linux account was created. The commands below use the normalized name `lab-user`:

```bash
sudo adduser lab-user
```

Initial password entries were empty or mismatched. `passwd` reported `Authentication token manipulation error` and the password was not set successfully at that stage. `adduser` continued to the optional personal-information fields. I left those fields empty and confirmed the information.

The **Full Name** field was optional account metadata; it did not change the login name. After returning to the shell, I set the account password successfully:

```bash
sudo passwd lab-user
```

From my authorized MacBook, I opened a fresh SSH session to that account and checked its identity:

```bash
# Example account name and Tailscale address.
ssh lab-user@100.101.10.10
```

Inside that session:

```bash
whoami
id
```

The observed numeric identity was **UID 1004 / GID 1004**, with the user's own group and `users` (GID 100). The `sudo` group was absent and the account was not added to it. No full sudoers audit was performed.

The successful test proved that the separate account and password worked **from my already authorized MacBook**. It did not grant my collaborator access from another device. Their Tailscale login identity was still unavailable, so no invitation, device sharing, or additional network grant was completed.

A separate Linux account has its own credentials and home directory; it is not a separate VM or an isolated network. A shell on the Pi may still reach other lab resources subject to the host's permissions and network policy.

## 12. Reviewing the Kali upgrade before installation

From the Pi's administrator session:

```bash
sudo apt update
df -h /
apt -s full-upgrade
```

Metadata refresh fetched approximately **73.6 MB** and reported **1462 packages available for upgrade**. The listed sources included Kali `kali-rolling` and the existing Tailscale stable Debian `bullseye` repository for ARM64. No repository file was changed during the session.

The simulation and subsequent confirmation screen showed:

| Item | Planned value |
| --- | --- |
| Upgrade | `1461` packages |
| Install | `80` packages |
| Remove | `2` packages |
| Not upgrade | `1` package: `openssh-client-gssapi` |
| Download | `2,436 MB` |
| Additional space | `594 MB` |
| Available space reported by apt | `106 GB` |
| Root filesystem from `df -h /` | `/dev/mmcblk0p2`; `118G` total, `14G` used, `99G` available, `13%` used |

The planned removals were `network-manager-gnome` and `python3-pkg-resources`. Removing the graphical NetworkManager frontend was not a reported removal of the NetworkManager service itself. Why `openssh-client-gssapi` remained unupgraded was not investigated.

These were the approved **plan counts**. A complete final list of installed versions and a fresh post-upgrade list of remaining updates were not captured.

## 13. Running the upgrade in tmux

```bash
tmux -V
tmux new -s lab-update
sudo apt full-upgrade
```

`tmux 3.6b` was available. I started the `lab-update` session, reviewed the confirmation summary, and approved the upgrade with `Y`. The status bar beneath the prompt belonged to tmux and was not an additional package-manager question.

The session kept the upgrade attached to a persistent terminal if SSH disconnected. It would not survive reboot or power loss.

### Package and configuration dialogs

| Prompt | Recorded choice | Purpose |
| --- | --- | --- |
| libc6: restart services without asking | `No` | Retain explicit control over service-restart prompts |
| libc6: services to restart | Keep `ssh cron`; confirm `OK` | Restart the selected services after library updates |
| `/etc/lightdm/lightdm.conf` | `N` | Keep the locally installed configuration |
| `/etc/ssh/sshd_config` | Keep the local version currently installed | Preserve the working SSH server configuration |

Keeping a configuration file did not cancel the software upgrade. A locally modified-file notice did not identify who changed it or prove a security issue. No configuration diff was reviewed during these prompts; comparison with the distributor's updated files remained possible follow-up work.

### Checks after installation

```bash
sudo dpkg --audit
sudo apt-get check
systemctl is-active ssh tailscaled
```

`dpkg --audit` returned no findings, the dependency check completed without errors, and the services returned:

```text
active
active
```

## 14. Reboot, password retries, and recovery

From the Pi session:

```bash
sudo reboot
```

The SSH connection ended with `Operation timed out` and `Broken pipe`. A dropped connection was expected after reboot; there was no need to manually power-cycle the device.

On reconnect, the server offered a password prompt but rejected several entries. I later confirmed that I had mistyped the password, then successfully logged in. No Tailscale, firewall, or account reset was required.

Post-reboot checks:

```bash
uname -r
systemctl is-active ssh tailscaled
systemctl --failed --no-pager
```

Recorded output:

```text
6.12.34+rpt-rpi-2712
active
active
  UNIT LOAD ACTIVE SUB DESCRIPTION

0 loaded units listed.
```

The Pi was accessible again through Tailscale, both services were active, and no failed systemd units were listed. The running kernel was **unchanged** from the initial check; this was not evidence that userspace packages had failed to update. The pre-reboot tmux session did not persist across reboot.

## 15. End-of-phase state and limits

| Area | Final documented state |
| --- | --- |
| Pi hardware connection | Switch Port 8; `eth0`; headless operation |
| OS | Kali Rolling 2026.2, ARM64 |
| Remote administration | Ordinary OpenSSH over Tailscale; external-network login verified |
| Tailnet ownership | Pi and MacBook enrolled under my own account |
| Tailnet access policy | My specified MacBook → Pi TCP 22; saved and fresh login tested |
| Administrator account | `kali`; password changed and new login verified |
| Collaborator account | Separate account created; UID/GID 1004; login tested from my MacBook |
| Collaborator's Tailscale access | Pending; identity not yet supplied |
| Updates | Full upgrade performed; package checks clean before reboot |
| Post-reboot services | `ssh` and `tailscaled` active; zero failed units |
| Entire-lab remote routing | Not configured; no subnet route advertised |
| Router and switch configuration | Unchanged in this phase |

The service, socket, account, and key checks were a baseline review. They did not prove the absence of malware, audit all persistence mechanisms, or demonstrate complete isolation from every household and lab device. The prior MikroTik forwarding policy and the new Tailscale policy apply at different boundaries. Same-LAN traffic does not become isolated merely because remote SSH is restricted.

## 16. Next work

1. Obtain my collaborator's Tailscale login identity.
2. Configure scoped access to the Pi while retaining my working administrator path.
3. Verify a fresh connection from the collaborator's own device to their Linux account; grant only the additional permissions needed.
4. Consider individual SSH keys and an appropriate backup of host configuration and tailnet policy.
5. Review retained configuration-file differences and the remaining unupgraded package if needed.
6. Decide separately whether broader lab access, Proxmox access, or subnet routing is required.

Switch-management inspection, shared-VLAN documentation, the first VM, and the remaining Phase 5 isolation tests are still open. Establishing remote access to the Pi did not complete those tasks.

## 17. Lessons learned

- A DHCP lease table can identify a known device without scanning the subnet.
- Inspect installed software and account ownership before starting a new installation.
- Use the local LAN management path when logging a remote-access service out of its old account.
- A fresh connection from a phone hotspot tests remote access more convincingly than an existing local session.
- Ordinary OpenSSH over Tailscale and the Tailscale SSH feature are different authentication arrangements.
- Network permission, Linux account authentication, and administrator privileges are separate layers.
- Policy assertions and live network tests provide different evidence; record which was actually performed.
- tmux protects a long-running terminal session against SSH loss, not against reboot.
- Confirm package consistency, service availability, and a new login after a large update.

## Quick reference

### Addressing and identity

| Item | Phase 6 value |
| --- | --- |
| Lab network / gateway | `192.168.88.0/24` / `192.168.88.1` |
| Pi LAN address | `192.168.88.249`; dynamic DHCP snapshot |
| Pi hostname / interface | `kali-raspberrypi` / `eth0` |
| Pi switch connection | Port `8` |
| Switch LAN address | `192.168.88.252`; dynamic DHCP snapshot |
| MacBook Ethernet address | `192.168.88.251`; dynamic DHCP snapshot |
| Pi Tailscale address in this chapter | `100.101.10.10` — example only |
| MacBook Tailscale address in this chapter | `100.101.10.20` — example only |
| Administrator | `kali`; UID 1000; member of `sudo` |
| Collaborator account in this chapter | `lab-user` — normalized name; observed UID/GID 1004 |
| Authentication | Ordinary OpenSSH with passwords over Tailscale |
| Kernel after reboot | `6.12.34+rpt-rpi-2712` |
| Original policy backup | `tailscale-policy-backup.txt` |
| Upgrade tmux session | `lab-update`; existed before reboot only |
| Immediate next step | Configure and test collaborator access after receiving their Tailscale identity |

### Read-only checks — Pi SSH session

```bash
whoami
id
ip -br addr
ip route
tailscale status
systemctl is-active ssh tailscaled
systemctl --failed --no-pager
uname -r
sudo dpkg --audit
sudo apt-get check
```

For a future update session that still exists after an SSH disconnection, reconnect as the same Linux user and run `tmux attach -t lab-update`. After a reboot, do not assume that the old session exists. To end an SSH session normally, use `exit`.

## References

- [Tailscale — Tailnet policy file syntax](https://tailscale.com/docs/reference/syntax/policy-file)
- [Kali Linux — Updating Kali](https://www.kali.org/docs/general-use/updating-kali/)
- [OpenSSH — sshd_config manual](https://man.openbsd.org/sshd_config.5)

*Source: Hebrew Raspberry Pi and Tailscale report and recorded session outputs from October 10, 2026. Commands that change enrollment, accounts, or packages document completed work and are not intended to be replayed as a single script. Publication substitutions are identified at the start of this chapter.*
