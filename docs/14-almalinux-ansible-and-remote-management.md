# Phase 14 — AlmaLinux, Ansible, and Remote Lab Management

**Result:** Personal Authentik sign-in with TOTP succeeded. AlmaLinux VM 101 was reached over SSH from the MacBook and backed up manually. The existing nightly Proxmox backup job was updated to include both VMs. A first Ansible playbook ran locally on VM 101 and made no further changes on its second run. DNS deployment and remote browser access to MikroTik remained open.

> **Historical scope:** Work after the Phase 13 Authentik deployment through November 13, 2026, reconstructed from the available conversation excerpts and command results. The final troubleshooting record is partial.

Previous phase: Phase 13 — Authentik deployment on Raspberry Pi. Project overview: [README](../README.md).

> **Publication convention:** Commands use saved SSH aliases. Account paths and raw Tailscale addresses are omitted. The MikroTik lab LAN address and the VM backup filename are retained to identify the tested destination and backup artifact.

## 1. Starting point and work sequence

At the end of Phase 13, Authentik and PostgreSQL were running in Docker on the Raspberry Pi. The collaborator's Authentik access over HTTPS had been verified. Personal TOTP and the Authentik recovery checks were still open. Proxmox already had a daily backup job for VM 100.

| Stage | Recorded progress |
| --- | --- |
| AlmaLinux | VM 101 was running AlmaLinux 10.2. `ansible-core` was installed and its version checked on November 10. |
| Authentik | On November 13, a fresh sign-in with the personal account prompted for a one-time code and accepted it. |
| Proxmox backups | A manual backup of VM 101 completed successfully. The nightly job was updated to include VM 100 and VM 101. |
| SSH and Ansible | SSH from the MacBook reached VM 101. The local Ansible ping succeeded. A practice playbook passed its syntax check, created a file, and produced `changed=0` on rerun. |
| DNS, DHCP, and router access | A DNS server on a VM was discussed. The conversation then clarified the current DHCP role and moved to remote MikroTik management over Tailscale. |

## 2. AlmaLinux and SSH access

The Proxmox console identified **VM 101** as `almalinux`. Its release was **AlmaLinux 10.2 (Lavender Lion)**. `ansible --version` reported **Ansible Core 2.16.16** after the package installation. The VM appeared **Connected** in Tailscale as `lab-alma101`. The November 13 SSH session opened a shell on the VM under the regular administrator's Linux account.

| Saved alias | Destination |
| --- | --- |
| `alma101` | AlmaLinux VM 101 |
| `pi` | Raspberry Pi |
| `proxmox` | Proxmox host |

The SSH login demonstrated remote access from the MacBook to VM 101. The exercise that followed ran on that VM. Ansible access to a separate managed host remained open.

## 3. Personal Authentik sign-in with TOTP

The personal Authentik account was used for a new sign-in. Authentik requested a one-time passcode, accepted the code, and completed sign-in. This verified the interactive TOTP flow for that account.

The user's Groups view was empty in the captured state. Membership in the intended admin group and the account's resulting administrative permissions still needed verification. Authentik and PostgreSQL backup and recovery work remained open from Phase 13.

## 4. Manual backup and nightly job update

The manual backup of VM 101 used Proxmox `local` storage, **Snapshot** mode, and **ZSTD** compression. Its task log ended with `Backup job finished successfully` and `TASK OK`. The archive was reported at **1.40 GB**.

```text
vzdump-qemu-101-2026_11_13-11_26_56.vma.zst
```

The existing job under **Datacenter → Backup** was expanded to include VM 101 alongside VM 100.

| Setting | Recorded value |
| --- | --- |
| Selection | VM 100 and VM 101 |
| Schedule | Daily at 02:00 on the Proxmox host |
| Storage | `local` |
| Mode / compression | Snapshot / ZSTD |
| Retention | `keep-last=3` |

Phase 12 had already verified a scheduled run and isolated restore for VM 100. At this phase's cutoff, a scheduled run of the expanded job, an off-host copy of the VM 101 backup, and a VM 101 restore remained to be verified.

## 5. First Ansible playbook

On VM 101, Ansible's ping module reported `SUCCESS`, `ping: pong`, and `changed: false` for `localhost`. The playbook `~/ansible-first.yml` then passed its syntax check. The exercise targeted `localhost` using Ansible's inline inventory syntax.

```bash
ansible-playbook -i localhost, ~/ansible-first.yml
```

The task was **Create a practice file**. The first run made the change. The second run found the file in the desired state.

| Playbook run | `ok` | `changed` | `unreachable` | `failed` |
| --- | ---: | ---: | ---: | ---: |
| First | 1 | 1 | 0 | 0 |
| Second | 1 | 0 | 0 | 0 |

The repeat run demonstrated idempotence for this local task. The next Ansible milestone would be testing a separate managed host.

## 6. DNS and current DHCP

The discussion first concerned running a DNS server on one of the VMs. VM selection and deployment remained open. The later DHCP question concerned address assignment for the entire lab.

| Setting | Documented baseline |
| --- | --- |
| Lab LAN | `192.168.88.0/24` |
| DHCP server and default gateway | MikroTik, `192.168.88.1` |
| Lab DHCP pool | `192.168.88.10`–`192.168.88.254` |

The existing DHCP role provided context for the proposed DNS server. The retrieved session contains no new DNS deployment or DHCP changes.

## 7. MikroTik management from outside the lab Wi-Fi

The final retrieved request was to open the MikroTik management interface from the MacBook over Tailscale while disconnected from the lab Wi-Fi. The discussion proposed using the reachable Proxmox host as an SSH forwarding hop. It also raised a route for the router's management address as a longer-term option.

| Path or option | Recorded status |
| --- | --- |
| MacBook → Proxmox over Tailscale and SSH | Already available for host administration. |
| Proxmox (`192.168.88.2`) → `192.168.88.1:80` | The separate tunnel troubleshooting session reported `Connection refused`. |
| Proxmox → `192.168.88.1:443` | An HTTPS check was started in that session. The available output stops before a result. Forwarding MacBook port `8443` to router port `443` was also proposed. |
| Tailscale route for `192.168.88.1/32` | Discussed as an option. Configuration and access validation remain open. |

The documented MikroTik baseline kept SSH and Winbox enabled and WebFig HTTP and HTTPS disabled. The port 80 result is consistent with that baseline. WebFig login from outside the lab Wi-Fi remained unverified at the cutoff.

## 8. End-of-phase status

| Area | Confirmed result | Open at the cutoff |
| --- | --- | --- |
| Authentik | Personal sign-in succeeded with TOTP. | Admin-group membership and permissions. Authentik and PostgreSQL backup and recovery. |
| AlmaLinux | VM 101 was reachable by SSH from the MacBook. | Use the VM to test management of a separate host. |
| Proxmox backup | VM 101 manual backup succeeded. The nightly job selection includes both VMs. | Observe a scheduled run of the expanded job and test VM 101 recovery. |
| Ansible | Local playbook created the practice file and was idempotent on rerun. | Remote inventory and managed-host testing. |
| DNS and DHCP | The MikroTik DHCP role is documented. | Select a VM for DNS, deploy the service, and validate queries. |
| Remote MikroTik management | The Proxmox SSH hop was available. A port 80 check was refused. | Confirm the intended management service and verify access from outside the lab Wi-Fi. |

The immediate continuation was to establish a working remote path to the MikroTik management interface from the MacBook and verify it off the lab Wi-Fi.
