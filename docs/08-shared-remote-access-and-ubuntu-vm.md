# Phase 8 — Shared Remote Access and the First Ubuntu VM

**Result:** My collaborator connected to the Raspberry Pi from their own computer after we corrected the Tailscale identity in the access policy. I added remote access to Proxmox, verified its web interface from a phone hotspot, and created the first Ubuntu Server VM. The guest was updated, rebooted, and checked. A fresh SSH login using my MacBook's key succeeded, and `sudo` was verified inside that session.

> **Historical scope:** October 15–20, 2026, following Phase 7. This chapter ends after successful key authentication to Ubuntu. Password-based SSH authentication has **not** yet been disabled. Version numbers and resource figures are observations from these sessions, not recommendations or a live inventory.

Previous phase: [MacBook Hardening and Switch Management](07-macbook-hardening-and-switch-management.md). Project overview: [README](../README.md).

> **Publication convention:** LAN addresses are retained. Tailscale addresses are illustrative: Pi `100.101.10.10`, MacBook `100.101.10.20`, Proxmox `100.101.10.30`, and Ubuntu `100.101.10.40`. `admin-mac`, `lab-user`, and `vm-admin` normalize policy aliases and personal usernames. `collaborator@github` is a placeholder for the verified login identity. Fingerprints, personal tailnet names, email addresses, passwords, and authentication URLs are omitted. Substitute actual values before using examples; this is a historical record, not a deployment script.

## 1. Objective and starting point

The router, switch, Pi, and Proxmox host were already working. The next goal was to turn that infrastructure into a usable shared lab: finish collaborator access, manage Proxmox remotely, and bring up a small Linux guest without changing the existing network design.

## 2. Scope carried forward from Phase 7

The lab remained `192.168.88.0/24`, with the MikroTik at `192.168.88.1` providing routing and DHCP. The switch retained VLAN 1 and PVID 1. No new VLAN, switch routing configuration, or replacement for the lab bridge was introduced.

The switch backup and MacBook baseline from Phase 7 remained in place. Outstanding backup, firmware, and management-service checks were not marked complete simply because VM work had begun.

## 3. Physical organization and documentation

The equipment was arranged on a small stand, and photos were taken from two angles. We also worked on a network diagram and a draft LinkedIn post describing the project as collaborative work. Publication of that post was not confirmed. A final cable-labeling and full connectivity audit remained separate from arranging the equipment.

## 4. Ideas considered without changing the project

We discussed larger homelabs, NAS systems, clusters, and virtualization. No Kubernetes cluster, additional storage system, or pooled-memory configuration was deployed. Two computers with 8 GB each do not automatically provide a single VM with 16 GB. The immediate work stayed focused on the existing mini PC and its first guest.

## 5. Identifying the collaborator's Tailscale account

The collaborator had joined the correct tailnet but was authenticated through GitHub. The login identity reported by Tailscale differed from the email address originally supplied for the access rule.

The distinction mattered: the policy needed to match the actual authenticated identity, not an assumed email address. The Linux account on the Pi was a separate identity again.

## 6. Diagnosing and fixing collaborator access

My MacBook could reach the Pi, while the collaborator received `no matching peer` from `tailscale ping` and an SSH timeout. One SSH attempt also used the invalid option `ConnectionTimeout`; the valid spelling is `ConnectTimeout`:

```bash
ssh -v -o ConnectTimeout=10 lab-user@100.101.10.10
```

Correcting the command syntax did not resolve the policy mismatch. After the source identity in the policy was corrected and saved, the collaborator successfully reached the Kali shell **from their own computer**. This completed the external collaborator test that remained open in Phase 6.

## 7. Machine names and Linux permissions

The Pi's MagicDNS FQDN was retrieved and shared privately. A DNS name provides a convenient destination; it does not grant access or replace authentication.

The collaborator's Pi account remained separate, with the previously observed UID/GID 1004 and no `sudo` group membership. No general administrator or package-installation privilege was granted. Their laptop needed an SSH client, not an SSH server, to initiate this connection.

## 8. Remote access matrix at the end of this phase

| Source | Destination | Allowed connection |
| --- | --- | --- |
| My specified MacBook | Pi | TCP 22 / OpenSSH |
| Collaborator's verified Tailscale identity | Pi | TCP 22 / OpenSSH |
| My specified MacBook and collaborator identity | Proxmox host | TCP 8006 / HTTPS interface |
| My specified MacBook | Ubuntu VM | TCP 22 / OpenSSH |

No host SSH grant was added for Proxmox. The collaborator was not granted SSH access to this Ubuntu VM. Access to the Proxmox web interface still requires a valid Proxmox account and permissions.

## 9. Policy structure and test assertions

The normalized grants at the end of this phase were:

```json
{
  "hosts": {
    "admin-mac": "100.101.10.20",
    "lab-pi": "100.101.10.10",
    "lab-proxmox": "100.101.10.30",
    "lab-ubuntu": "100.101.10.40"
  },
  "grants": [
    {"src": ["admin-mac"], "dst": ["lab-pi"], "ip": ["tcp:22"]},
    {"src": ["collaborator@github"], "dst": ["lab-pi"], "ip": ["tcp:22"]},
    {"src": ["admin-mac", "collaborator@github"], "dst": ["lab-proxmox"], "ip": ["tcp:8006"]},
    {"src": ["admin-mac"], "dst": ["lab-ubuntu"], "ip": ["tcp:22"]}
  ],
  "ssh": [],
  "tests": [
    {"src": "admin-mac", "proto": "tcp", "accept": ["lab-pi:22", "lab-proxmox:8006", "lab-ubuntu:22"], "deny": ["lab-pi:80", "lab-pi:443", "lab-proxmox:22", "lab-ubuntu:80", "lab-ubuntu:443"]},
    {"src": "collaborator@github", "proto": "tcp", "accept": ["lab-pi:22", "lab-proxmox:8006"], "deny": ["lab-pi:80", "lab-pi:443", "admin-mac:22", "admin-mac:445", "admin-mac:5900", "lab-proxmox:22", "lab-ubuntu:22"]},
    {"src": "lab-pi", "proto": "tcp", "deny": ["admin-mac:22", "admin-mac:445", "admin-mac:5900", "lab-proxmox:22", "lab-proxmox:8006"]},
    {"src": "lab-proxmox", "proto": "tcp", "deny": ["admin-mac:22", "admin-mac:445", "admin-mac:5900", "lab-pi:22"]},
    {"src": "lab-ubuntu", "proto": "tcp", "deny": ["admin-mac:22", "admin-mac:445", "admin-mac:5900", "lab-pi:22", "lab-proxmox:22", "lab-proxmox:8006"]}
  ]
}
```

This example requires real identities and addresses before use. The `tests` entries express expected policy outcomes; their `deny` values are **not separate firewall rules**. They are also not evidence that every prohibited connection was tested with live traffic. The empty `ssh` array accompanies ordinary OpenSSH over Tailscale, rather than Tailscale SSH.

## 10. Limits of the isolation

Tailscale policy governs the overlay path. It does not automatically filter direct traffic between devices on the shared lab LAN. The MikroTik's routed household-subnet restriction likewise does not isolate every pair of devices on VLAN 1.

No public port forwarding, exit node, or advertised lab-subnet route was added. Each remotely managed endpoint received its own Tailscale client. Replies to an allowed connection do not require a grant allowing a new connection in the opposite direction.

## 11. Creating a dedicated MacBook SSH key

An ED25519 key pair was created with a passphrase:

| File or item | Purpose |
| --- | --- |
| `~/.ssh/id_ed25519_lab` | Private key, retained on the MacBook |
| `~/.ssh/id_ed25519_lab.pub` | Public key, suitable for adding to the destination account |
| Passphrase | Unlocks the private key locally; different from the server account password |
| Randomart | Visual representation of the key fingerprint, not a password |

The passphrase alone cannot recreate a lost key file. Installation and successful use of this key on the Pi were not confirmed; its verified deployment later in this phase was to Ubuntu.

## 12. Checking virtualization support on the Pi

| Observation | Recorded result |
| --- | --- |
| Architecture / processor | `aarch64` / Cortex-A76 |
| KVM device | `/dev/kvm` present, owned by `root:kvm` |
| RAM | 7.9 GiB total; 7.5 GiB available at that moment |
| Swap | 0 B |

The device's presence was useful evidence, but no VM was started on the Pi and no per-user KVM access test was completed. We continued with the x86 mini PC as the first VM host.

## 13. Inspecting Proxmox before remote access

The host initially appeared unreachable because it was powered off. After startup, the following state was recorded:

| Item | Recorded value |
| --- | --- |
| Hostname | `proxmox` |
| Proxmox Manager | `9.2.21` |
| Host kernel | `7.0.14-20-pve` |
| Base OS | Debian 13 / trixie, reported as 13.7 |
| LAN address | `192.168.88.2/24` on `vmbr0` |
| Physical interfaces | `nic0` up; `wlp2s0` down |
| Default gateway | `192.168.88.1` through `vmbr0` |

## 14. Connecting Proxmox to Tailscale

Tailscale 1.104.1 was observed on the host. Enrollment used:

```bash
tailscale up --accept-dns=false
tailscale status
```

The DNS option prevented Tailscale from taking over the host's DNS configuration. After the TCP 8006 grant was added, I logged into the interface using its Tailscale address. With Ethernet disconnected and the MacBook using a phone hotspot, refreshing the page still worked.

The HTTPS certificate trust warning was not resolved by installing a new certificate. Remote access was verified to the host itself, not to every LAN device.

## 15. Creating a separate Proxmox account

A separate collaborator account was created in Proxmox, represented here as `lab-user@pve`.

| Account | Login realm | Purpose |
| --- | --- | --- |
| `root@pam` | Linux PAM standard authentication | Existing host administration |
| `lab-user@pve` | Proxmox VE authentication server | Separate collaborator account |

The Proxmox account and the same person's Kali account are independent. A network grant to TCP 8006 does not create either account or assign an administrator role.

## 16. Checking the new login

A new browser window initially reused the existing root session. Logging out and selecting the correct realm allowed the separate account to be checked from my MacBook. Full host-administrator and unrestricted VM-creation access were not granted.

The collaborator's own Proxmox login and a VM-specific role assignment remained unverified. The plan was to provide a dedicated VM with scoped permissions, rather than automatically extending host administration.

## 17. Choosing the first VM

The first guest was a small Ubuntu Server installation without a desktop environment. This gave us a separate Linux system for administration practice while leaving room on the 8 GB host for further work. No second collaborator VM was created in this phase.

## 18. VM hardware configuration

| Setting | Recorded configuration |
| --- | --- |
| VM ID / name | `100` / `ubuntu-lab-01` |
| CPU | 1 socket, 2 cores, type `host` |
| Memory | 2048 MiB |
| Disk | 32 GiB, `scsi0`, `local-lvm`, raw |
| Disk options | Discard and IO Thread enabled |
| SCSI controller | VirtIO SCSI single |
| Machine / BIOS | i440fx / SeaBIOS |
| Network adapter | VirtIO, connected to `vmbr0`, no VLAN tag |
| QEMU Guest Agent | Enabled in VM options; guest package installed later |

## 19. Virtualization and networking boundaries

Proxmox manages the VM; QEMU provides the virtual hardware, and KVM supplies hardware-assisted execution. The QEMU Guest Agent is a separate service inside Ubuntu used for host–guest management communication.

Connecting the guest to `vmbr0` placed it on the existing lab LAN. The virtual NIC's Firewall option was checked, but that checkbox alone did not establish that an effective Proxmox firewall policy was active. No isolated attack-lab network was built.

## 20. Installing Ubuntu Server

| Item | Recorded result |
| --- | --- |
| OS / architecture | Ubuntu 24.04.5 LTS / x86_64 |
| Guest kernel | `6.8.0-147-generic` |
| Hostname | `ubuntu-lab-01` |
| Administrator account | `vm-admin` in this public record |
| Storage | LVM; root filesystem approximately 29.36 GB |
| OpenSSH | Installed; password login available |
| Optional services | Ubuntu Pro skipped; no Featured Snaps selected |

The guest account password, Proxmox password, and SSH-key passphrase serve different purposes. Their values are not included here.

## 21. Booting from the installed disk

After `Installation Complete`, the guest was rebooted and its virtual CD/DVD drive was set to **Do not use any media**. Ubuntu reached its login prompt and then the shell, confirming a boot from the installed system.

## 22. Moving from the console to SSH

The browser console was useful during installation, but copying commands was inconvenient. We prepared SSH access from the MacBook. Commands for the guest belong at its Ubuntu prompt, not at `root@proxmox` on the host.

The login banner offered a newer Ubuntu release, but `do-release-upgrade` was not run. This phase kept the guest on 24.04 LTS and applied package updates within that release.

## 23. Enrolling the Ubuntu guest

Inside Ubuntu:

```bash
sudo tailscale up --accept-dns=false
```

| Connection | Recorded value |
| --- | --- |
| Guest LAN interface | `ens18` |
| DHCP address observed | `192.168.88.249/24` |
| Gateway | `192.168.88.1` |
| Tailscale address in this document | `100.101.10.40` — example only |
| Policy alias | `lab-ubuntu` |

Only my MacBook received the guest's TCP 22 grant. The Pi had previously used the same `.249` LAN address in another DHCP snapshot. These observations were made at different times; they do not prove an address collision or establish the Pi's current LAN address.

## 24. Verifying the SSH host key

The server fingerprint was checked from the Ubuntu console before accepting the key in the MacBook's SSH prompt:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

The fingerprint itself is omitted from this public record. One connection closed after accepting the host key; a subsequent login succeeded. The initial closure's cause was not established.

```bash
ssh vm-admin@100.101.10.40
```

## 25. Guest services and security updates

The `qemu-guest-agent` package was installed and its service became active. The guest's address also appeared in the Proxmox summary.

On October 20, the package update upgraded `python3-jwt` from `2.7.0-1ubuntu0.3` to `2.7.0-1ubuntu0.4`. The output reported one standard LTS security update, no new packages, and no removals. It reported the kernel as up to date, with `dbus`, `systemd-logind`, and some user processes awaiting restart.

Ubuntu was rebooted. SSH disconnected with a `Broken pipe` message, and a fresh connection to the same Tailscale address succeeded afterward.

## 26. Post-reboot validation

Inside Ubuntu:

```bash
systemctl is-active ssh tailscaled qemu-guest-agent
systemctl --failed --no-pager
```

Observed results:

```text
active
active
active
0 loaded units listed.
```

The last line referred to the filtered list of **failed** units. The login banner reported zero available updates, 16.2% root-filesystem usage, and 9% memory usage at that moment. These were not sustained-load measurements.

The update tool's message about outdated hypervisor binaries was emitted inside the guest; it was not an independent health check of the Proxmox host.

## 27. Installing and testing the MacBook's public key

From the MacBook, the existing public key was copied into the guest account:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_lab.pub vm-admin@100.101.10.40
```

```text
Number of key(s) added: 1
```

A new connection explicitly selected public-key authentication:

```bash
ssh -i ~/.ssh/id_ed25519_lab \
  -o IdentitiesOnly=yes \
  -o PreferredAuthentications=publickey \
  vm-admin@100.101.10.40
```

The login succeeded. Only the public key was installed on Ubuntu; the private key remained on the MacBook. A local passphrase prompt unlocks that private key and is not the guest's account-password prompt.

## 28. Verifying administration and recording the stopping point

Inside the new SSH session:

```bash
sudo -v
```

The command returned to the prompt after authentication, confirming the account's `sudo` access.

| Check | End-of-session state |
| --- | --- |
| Public key installed | One key added |
| Fresh public-key login | Successful |
| `sudo` in new session | Verified |
| `PasswordAuthentication no` | Not yet configured |
| Keyboard-interactive authentication disabled | Not yet configured |
| Password-login rejection test | Not yet performed |

Key authentication works, but the guest must not yet be described as accepting keys only. Disabling SSH password authentication later does not require removing the Linux password used for `sudo` and console access.

## 29. What this phase completed

- [x] Correct the collaborator's Tailscale identity and verify Pi SSH from their computer.
- [x] Generate a dedicated MacBook key pair.
- [x] Check the Pi's KVM device and resource snapshot without deploying a VM there.
- [x] Enroll Proxmox in Tailscale and verify its interface from an external hotspot.
- [x] Create and check a separate Proxmox account from my MacBook.
- [x] Create VM 100 and install Ubuntu Server.
- [x] Enroll Ubuntu in Tailscale and verify SSH.
- [x] Update and reboot the guest; verify three active services and no failed units.
- [x] Install the public key, test a fresh key login, and verify `sudo`.

## 30. Next work

1. Inspect Ubuntu's effective SSH configuration, then configure key-only remote authentication. Keep a working session open, validate the configuration, and test a new key login and password rejection before closing the recovery path.
2. Create a suitable VM backup and verify restoration. A snapshot, if added, is not a replacement for a separate backup.
3. Verify the collaborator's own Proxmox login, create their dedicated VM, and assign permissions scoped to that VM.
4. Plan network isolation before introducing intentionally vulnerable guests. Tailscale restrictions do not isolate the shared LAN.
5. Carry forward Phase 7's open tasks: previous-owner data backup confirmation, switch time/firmware/management-service review, and a post-change MikroTik export.

## Address and access reference

| Endpoint | Address used in this public record | Access |
| --- | --- | --- |
| MikroTik | `192.168.88.1` | Local lab management |
| SG3210 | `192.168.88.252` | Local HTTPS; DHCP reservation |
| Proxmox LAN | `192.168.88.2` | HTTPS TCP 8006 |
| Proxmox overlay | `100.101.10.30` — example | HTTPS TCP 8006 |
| Pi overlay | `100.101.10.10` — example | SSH TCP 22 |
| Ubuntu overlay | `100.101.10.40` — example | SSH TCP 22 |
| Ubuntu LAN | `192.168.88.249` | Dynamic snapshot, not a reservation |

Private credentials, actual overlay identifiers, recovery material, and raw configuration exports remain outside the repository.
