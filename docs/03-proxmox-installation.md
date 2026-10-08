# Phase 3 — Proxmox VE Installation on the Intel N100

This phase covers replacing the mini PC's Windows installation with Proxmox VE, assigning management networking, completing the first local login, and initiating an orderly shutdown.

**Outcome:** Proxmox booted from the internal SSD with management address `192.168.88.2/24`. Local login as `root` succeeded. The session ended by running `poweroff`; browser access and post-installation checks remained pending.

> **Historical scope:** This record is based on installation notes compiled on October 4, 2026. A successful local console login is documented. A successful web-interface login, full hostname verification, system updates, and creation of the first VM are not yet documented.

Previous phase: [MikroTik Wireless WAN and Lab Network Setup](02-network-setup.md). Project overview: [README](../README.md).

## 1. Objective and starting state

I selected Proxmox VE as the virtualization host for this home lab, with an Ubuntu Server VM, additional Linux systems, and containers planned for later stages. Proxmox would run directly on the N100 hardware, replacing the seller-provided Windows installation.

The MikroTik network had already been configured in Phase 2. The installation USB was ready, and the mini PC was connected by Ethernet to a MikroTik LAN port. `ether1`, which retained its WAN role, was not used for the server connection.

| Item | Recorded starting state |
| --- | --- |
| Processor | Intel N100, 4 cores |
| Memory | 8 GB RAM |
| Storage | 256 GB SSD |
| Hardware virtualization | Enabled |
| Existing OS | Windows 11 Pro |
| Windows activation | Digital license linked to a Microsoft account |
| Lab router | MikroTik hAP ax³, `192.168.88.1` |
| Lab subnet | `192.168.88.0/24` |
| DHCP pool | `192.168.88.10`–`192.168.88.254` |
| Management computer | MacBook on the MikroTik LAN |

Windows activation was checked before erasing the disk. The decision to proceed included the expectation that Windows could generally be reinstalled and reactivated on the same hardware later; that restoration was not performed or tested during this project phase.

Phase 2 proposed `ether3` for the server. The installation record confirms use of a LAN port, but does not independently confirm its exact port number.

## 2. Installation media and the Etcher workaround

The installation notes identify the downloaded image as:

```text
Proxmox VE 9.2 ISO Installer
proxmox-ve_9.2-1.iso
```

These are the version label and filename recorded for this installation, not a recommendation to use a particular current release.

The original attempt to open the ISO in balenaEtcher on the MacBook failed with:

```text
Error opening source
h.requestMetadata is not a function
```

balenaEtcher was removed, and the workflow switched to **Raspberry Pi Imager**:

1. Opened **Choose OS → Use Custom**.
2. Selected `proxmox-ve_9.2-1.iso`.
3. Selected **USB SanDisk 3.2Gen1 Media** as the storage target.
4. Wrote the ISO successfully to the USB drive.

This completes the error record summarized in Phase 2. The exact software cause of the Etcher error was not established; changing the writing tool allowed preparation to proceed.

## 3. Network and router checks before installation

The N100 and MacBook were connected to the lab LAN. The MikroTik used `wifi1` as its 5 GHz wireless uplink to the household AT&T network.

During the pre-installation check, **IP → UPnP** showed that UPnP had been enabled accidentally. It was turned off. This is the latest recorded state, following the earlier Phase 2 record of UPnP being off; the timing of the accidental change was not established.

| Setting | State recorded before installation |
| --- | --- |
| UPnP | Off after correction |
| MikroTik firewall | Remained enabled |
| Proxmox port forwarding | Not configured |

The intended policy was to manage Proxmox from the lab network without directly exposing the management interface to the internet. This record does not add a verified lab-to-household isolation test to those documented in Phase 2.

## 4. Booting the USB installer

Attempts to open the boot menu using **F12, F11, F7, and Esc** during startup were unsuccessful; Windows started before the menu could be reached. These attempts did not establish the correct firmware boot-menu key.

The successful route was through Windows:

1. Opened **Start → Power**.
2. Held **Shift** while selecting **Restart**.
3. Entered the blue Windows recovery screen.
4. Selected **UEFI USB PARTITION 2**, the SanDisk installation media entry.

The Proxmox boot menu displayed:

```text
Install Proxmox VE (Graphical)
Install Proxmox VE (Terminal UI)
Advanced Options
```

I selected **Install Proxmox VE (Graphical)** for the interactive installation.

## 5. Selecting the internal SSD

The **Target Harddisk** screen identified the installation target as:

```text
/dev/sda (238.47 GiB, PELADN 256GB)
```

| Field | Value |
| --- | --- |
| Disk path in this installation | `/dev/sda` |
| Displayed capacity | `238.47 GiB` |
| Device identifier | `PELADN 256GB` |
| Selected filesystem | `ext4` |

I checked that the selected device was the internal SSD rather than the SanDisk USB. Installing to this disk would erase its existing partitions, including Windows. `/dev/sda` is the path observed on this machine, not a universal target to assume on another system.

## 6. Regional settings and administration account

| Installer setting | Selection |
| --- | --- |
| Country | United States |
| Time Zone | America/Chicago |
| Keyboard Layout | English (US) |
| Keymap shown in the summary | `en-us` |

An **Administration Password** was set for `root`, and an email address was selected for system notifications. Neither the password nor the email address is included in the repository. The preparation guidance was to retain the password securely in a password manager.

## 7. Choosing the management interface

The **Management Network Configuration** screen showed two interfaces:

| Interface | Driver shown | Identification and selection |
| --- | --- | --- |
| `nic0` | `r8169` | Wired Ethernet; green link indicator; selected for management |
| `nic1` | `rtw88_8822ce` | Internal Wi-Fi interface; not selected for management |

The selected management interface was **`nic0`**. The N100 used its Ethernet connection to the MikroTik; the router, rather than the server's internal Wi-Fi interface, supplied the wireless uplink to the household network.

**Pin network interface names** remained checked. This was intended to keep interface names stable and was separate from the server's hostname setting.

## 8. Assigning the static management address

The installer initially displayed `192.168.100.2` with gateway `192.168.100.1`. Those values did not match the lab network already configured on the MikroTik.

I selected **`192.168.88.2/24`** for Proxmox: an address on the lab subnet and outside the router's DHCP pool, which begins at `192.168.88.10`.

| Management field | Entered value |
| --- | --- |
| Interface | `nic0` |
| Hostname / FQDN | `proxmox.lab.local` |
| IP address | `192.168.88.2` |
| CIDR | `/24` |
| Gateway | `192.168.88.1` |
| DNS server | `192.168.88.1` |

This gave the router, server, and DHCP clients distinct address roles:

| Role | Address or range |
| --- | --- |
| MikroTik LAN gateway | `192.168.88.1` |
| Proxmox host | `192.168.88.2` |
| Dynamically assigned clients | `192.168.88.10`–`192.168.88.254` |

## 9. Hostname discrepancy and final review

Although `proxmox.lab.local` was entered, the installer **Summary** displayed **Hostname: proxmox**. Returning to the previous screen and entering the full name again produced the same summary display.

The working explanation was that the summary showed the short hostname rather than the full FQDN. This remained a hypothesis. Installation continued with a plan to verify the full hostname from the installed system afterward.

The final review captured:

| Summary field | Displayed value |
| --- | --- |
| Filesystem | `ext4` |
| Disk | `/dev/sda` |
| Country | United States |
| Timezone | America/Chicago |
| Keymap | `en-us` |
| Management Interface | `nic0` |
| Hostname | `proxmox` |
| IP CIDR | `192.168.88.2/24` |
| Gateway | `192.168.88.1` |
| DNS | `192.168.88.1` |

**Automatically reboot after successful installation** remained checked. The plan was to remove the installation USB after completion or during the reboot so the machine would start from the internal SSD.

## 10. Installation result and first local login

Installation completed successfully, and the N100 booted Proxmox from its internal SSD. The previous Windows installation had been replaced.

The local console displayed:

```text
Welcome to the Proxmox Virtual Environment.

Please use your web browser to configure this server - connect to:

https://192.168.88.2:8006/
```

This showed that the configured address had been retained. Displaying a URL on the console was not a test of browser reachability.

At the prompt:

```text
proxmox login:
```

I entered `root` and the administration password chosen during installation. There was initially concern that special characters were preventing password entry. The console behavior was clarified: Linux did not display letters, asterisks, or dots while the password was being typed.

The login succeeded and produced:

```text
root@proxmox:~#
```

The screen also identified the running system as `Linux proxmox` and `x86_64`. The record does not establish a password-character defect; the confirmed outcome was successful local authentication.

## 11. Browser management — planned next check

The planned management connection was from the MacBook on the MikroTik LAN to:

```text
https://192.168.88.2:8006
```

| Login field | Planned value |
| --- | --- |
| Username | `root` |
| Password | Administration Password set during installation |
| Realm | Linux PAM standard authentication |

A certificate warning was expected from the initial self-signed certificate. The recorded guidance concerned access to this specific local server, not bypassing unrelated browser warnings. The address and intended server would need to be checked during that first connection.

**A successful browser login was not recorded in this phase.** The login that succeeded was at the local console.

For future access from outside the home, WireGuard or Tailscale was discussed as a VPN-based option. Neither was configured here. No port-forwarding rule for the Proxmox management port, TCP `8006`, was created.

## 12. First orderly shutdown

At the authenticated root prompt, I ran:

```bash
poweroff
```

The console showed:

```text
root@proxmox:~# poweroff
```

Pressing Enter started the orderly shutdown process. The alternative command discussed was:

```bash
shutdown -h now
```

The alternative was provided for reference; `poweroff` was the command actually entered. The documented session ends with shutdown being initiated, rather than with a subsequent restart test.

## 13. Problems, decisions, and evidence

| Issue | Response | Recorded result / remaining uncertainty |
| --- | --- | --- |
| Etcher could not open the ISO | Switched to Raspberry Pi Imager / Use Custom | USB image written successfully; Etcher's root cause not established |
| Startup keys did not open the boot menu | Used Windows Shift + Restart recovery path | Booted UEFI USB PARTITION 2 and reached the installer |
| Installer address did not match the lab subnet | Changed management addressing to `192.168.88.2/24`, gateway/DNS `.1` | Address appeared in the installed system's console URL |
| Two possible management interfaces | Selected wired `nic0` using its Ethernet driver and active link | Installer used `nic0`; internal Wi-Fi was not selected |
| Full hostname entered, short name displayed | Re-entered FQDN, then proceeded with follow-up verification planned | `proxmox` displayed; full FQDN verification still pending |
| Password entry appeared invisible | Clarified console password behavior and completed login | Root shell obtained; no password-character fault confirmed |
| UPnP found enabled during pre-install checks | Turned it off | Latest recorded state: Off |

## 14. End-of-phase status and next steps

### Completed and observed

- Proxmox installed on the internal PELADN SSD, replacing Windows.
- Installation filesystem selected as `ext4` on `/dev/sda`.
- Wired interface `nic0` selected for management.
- Static address `192.168.88.2/24` configured, with gateway and DNS `192.168.88.1`.
- Proxmox booted from the installed system and displayed the expected management URL.
- Local `root` login succeeded.
- `poweroff` initiated an orderly shutdown.

### Next-session work recorded in the notes

1. Power on the MikroTik and N100.
2. Connect from the MacBook to the web interface and verify login.
3. Inspect the Proxmox node and storage.
4. Review update repositories and carry out initial updates.
5. Verify the configured FQDN after the installer summary discrepancy.
6. Upload a Linux ISO and create the first VM, with Ubuntu Server the intended starting point.

Lab-to-household firewall isolation testing remained open from the previous phases. No VM, container, remote-access VPN, or successful browser session is claimed as completed here.

## 15. Lessons learned

- A known lab subnet made the installer's initial addressing mismatch easier to recognize.
- Selecting a static server address outside the DHCP pool kept it separate from automatic client allocation.
- Disk model and capacity were important checks before replacing the existing OS.
- Boot-menu troubleshooting had a workable alternative through Windows recovery.
- An interface's role, a server's short hostname, and its FQDN are separate settings that need separate verification.
- A local root shell confirms local access; browser connectivity and remote management require their own tests.
- Recording the exact error, successful workaround, and unresolved question is more useful than treating every suspected cause as proven.

## Quick reference

| Item | Phase 3 value |
| --- | --- |
| Host CPU / RAM | Intel N100 / 4 cores / 8 GB |
| Disk | PELADN 256GB, shown as 238.47 GiB |
| Installation target / filesystem | `/dev/sda` / `ext4` |
| Management NIC | `nic0` / `r8169` |
| Unused wireless NIC | `nic1` / `rtw88_8822ce` |
| Management address | `192.168.88.2/24` |
| Gateway / DNS | `192.168.88.1` |
| Displayed hostname | `proxmox` |
| Entered FQDN | `proxmox.lab.local`; verification pending |
| Web interface | `https://192.168.88.2:8006` |
| User / planned web realm | `root` / Linux PAM standard authentication |
| Lab DHCP pool | `192.168.88.10`–`192.168.88.254` |
| Upstream path | MikroTik `wifi1`, 5 GHz station, to the household network |
| Latest UPnP state | Off |
| Proxmox port forwarding | Not configured |
| Recorded ISO | `proxmox-ve_9.2-1.iso` |
| USB writer / target | Raspberry Pi Imager / USB SanDisk 3.2Gen1 Media |
| Boot entry used | UEFI USB PARTITION 2 |
| Region / keyboard | United States / America/Chicago / English (US) |
| Shutdown command used | `poweroff` |
| Alternative discussed | `shutdown -h now` |

*Source: Phase 3 Proxmox installation notes. This repository record preserves the observed settings and outcomes; passwords and the notification email are omitted.*
