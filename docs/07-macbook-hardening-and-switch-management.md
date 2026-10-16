# Phase 7 — MacBook Hardening and Switch Management

**Result:** I enabled the MacBook's firewall, reviewed sharing and app permissions, and changed its login password. I then restored Chrome access to the TP-Link switch, verified the hardware and existing VLAN configuration, reserved its management address in MikroTik DHCP, and labeled the three main switch ports. The switch configuration was saved and an updated backup was downloaded.

> **Historical scope:** Work performed from October 10–15, 2026, after the Raspberry Pi and Tailscale work in Phase 6. This chapter ends with the MacBook and switch-management tasks. Later collaborator onboarding and SSH-key work belong to a separate phase. No new VM was created here.

Previous phase: [Raspberry Pi, Tailscale, and Remote Access](06-raspberry-pi-and-tailscale.md). Project overview: [README](../README.md).

> **Publication convention:** Private LAN addresses are retained. Personal account names, email addresses, MAC addresses, passwords, and recovery keys are omitted. The Pi's example Tailscale address remains `100.101.10.10`, matching Phase 6. `<SWITCH_MAC>` is a placeholder, not a value to paste unchanged. Configuration exports are private backups and are not included in this repository.

## 1. Objective and starting point

With remote SSH to the Pi working, I focused on the computer used to administer the lab and the switch connecting the equipment. The goals were to reduce unnecessary access on the MacBook, make switch management predictable, and document the working configuration.

The lab stayed on **one LAN, `192.168.88.0/24`, and one shared switch VLAN**. The MikroTik remained the gateway, DHCP server, and internet boundary. This phase did not change its forwarding rules or remove the lab bridge.

| Component | Role at the start of this phase |
| --- | --- |
| MacBook Pro | Local Ethernet administration and remote SSH to the Pi through Tailscale |
| MikroTik hAP ax³ | Lab gateway and DHCP server at `192.168.88.1` |
| TP-Link SG3210 | Switch; management address `192.168.88.252` initially assigned dynamically |
| N100 / Proxmox | Existing host at `192.168.88.2`, connected to Switch Port 3 |
| Raspberry Pi / Kali | Existing client on Switch Port 8; previously verified LAN address `192.168.88.249` |

## 2. Inspecting the MacBook before cleanup

The MacBook had previously belonged to another person. Before removing anything, I checked the system, storage, management enrollment, and installed applications. I first left the Pi's SSH session with `exit`, so these commands ran on the **MacBook itself**:

```bash
sw_vers
system_profiler SPHardwareDataType | grep -E 'Model Name|Model Identifier|Chip|Processor Name|Memory'
df -h /
profiles status -type enrollment
ls -1 /Applications
```

| Check | Recorded result |
| --- | --- |
| macOS | `26.7.1`, build `25G241` |
| Hardware | MacBook Pro, `MacBookPro16,1` |
| Processor | 8-core Intel Core i9 |
| Memory | 16 GB |
| Storage reported by `df` | 932 GiB filesystem size; 840 GiB available |
| Enrollment | `Enrolled via DEP: No`; `MDM enrollment: No` |
| `/Applications` listing | Google Chrome, Microsoft Word, Safari, Tailscale, Zoom, and Utilities |

These are session observations, not claims about the latest software versions. The `12 GiB` used value shown for `/` was not an inventory of all personal data across APFS volumes. The available-space result did not indicate a storage shortage, but it also did not prove the computer was empty.

No factory reset, clean installation, or confirmed application removal was performed. Zoom remained installed because I still use it occasionally. Personal-file cleanup was deferred until the previous owner's backup could be confirmed.

## 3. Enabling the MacBook firewall

The initial checks were:

```bash
fdesetup status
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
```

```text
FileVault is On.
Firewall is disabled. (State = 0)
```

FileVault was already enabled. I enabled the local firewall and checked its state again:

```bash
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
```

```text
Firewall is enabled. (State = 1)
```

I confirmed that SSH to the Pi still worked after the change. The services reviewed under **Sharing** were reported as off; file sharing, screen sharing, and remote login to the MacBook were not enabled for this work.

The macOS firewall controls incoming connections to applications and services. It is separate from Tailscale access policy and the MikroTik forwarding rules. Enabling it was not a malware scan or a complete security audit.

A typing mistake later left an unmatched quote and displayed the shell prompt `quote>`. The recovery instruction was **Control + C** to cancel that unfinished input. This was a shell-input issue, not a firewall failure.

## 4. Reviewing login, privacy, and background permissions

I changed the MacBook login password and confirmed that automatic login was off. The account remained an administrator and Guest access was off. The account displayed my name, while the existing Unix username and home-directory path remained unchanged; no new macOS account was created.

Screen-lock settings were reviewed. A five-minute interval was discussed, along with requiring a password immediately after sleep or locking. The exact final combination of those settings was not independently recorded.

| Area | Recorded action or observation |
| --- | --- |
| Microphone | Chrome and Zoom permissions turned off |
| Camera | Reviewed permissions confirmed off |
| Screen recording | Zoom permission turned off |
| Full Disk Access | No applications shown in the reviewed list |
| Accessibility / Input Monitoring | Zero permissions shown |
| Automation / App Management | Zero permissions shown |
| Open at Login | List was empty |
| Background activity | Zoom background activity disabled; application retained |
| Tailscale and browser updates | Required Tailscale components retained; Google Update components not removed |

These changes reduced app access while keeping the tools needed for lab administration. Camera or microphone permission may need to be enabled again when intentionally using Zoom.

Chrome's **Local Network** permission was enabled later, on October 13, for switch management. That permission is separate from camera access, microphone access, and the incoming-connections firewall.

## 5. Browser, performance, and recovery checks

I reported that macOS was up to date and later confirmed updating Chrome. The browser review recorded:

| Setting | Observed state |
| --- | --- |
| Safe Browsing | Standard protection |
| Secure connections | Preference for secure connections enabled |
| Secure DNS | Using the system provider setting |
| Site notifications | Allow/block lists empty; sites could still request permission |
| Browser extension mentioned | Google Docs Offline; removal not confirmed |
| Apple Account | Sign In displayed; no signed-in account shown on that screen |

Activity Monitor showed green memory pressure, 16 GB physical memory, about 11.73 GB in use, and 278.5 MB swap. The CPU snapshot showed 95.54% idle, 1.55% system, and 2.91% user. Those snapshots did not suggest a reason to terminate processes indiscriminately. They were not a sustained performance test.

No cleanup utility was installed, and no system files or caches were deleted as a performance measure. An unsigned-in Apple Account did not establish that the previous owner's files had been removed. I confirmed possession of a FileVault recovery key, but an actual recovery test and confirmation of off-device access remained open.

## 6. Planning and equipment handling

Between the MacBook review and switch configuration, I discussed how the lab would develop:

- **Documentation:** Phase 6 and its README update had been prepared. Repository visibility, a concise description, and linking the project from LinkedIn were discussed. This phase did not verify a visibility change, inspect the repository's full history, or confirm publication on LinkedIn.
- **Virtual machines:** Proxmox would provide separate environments for learning operating systems, services, and networking. No Ubuntu Server VM was created here. A VM's network attachment and access rules determine its reachability; creating it alone does not isolate it from the LAN.
- **Physical organization:** A simple wire shelf or small stand was considered for the equipment. Completion of the final rack arrangement and a full post-move connectivity test were not recorded.
- **Continuous operation:** Power consumption and CPU workload were discussed as different measurements. No wall-power measurement or measured monthly electricity cost was recorded.

### Shutdown and startup guidance discussed

This was an operating procedure, not evidence that every device completed a shutdown during the session.

| Device | Procedure discussed |
| --- | --- |
| Raspberry Pi | Run `sudo shutdown -h now` inside the Pi, then allow the shutdown to finish before removing power |
| Proxmox | Shut down any active guests first, then shut down the host node |
| MikroTik | Use `/system shutdown` in RouterOS before removing power |
| SG3210 | Save the configuration, then disconnect its power supply |

`exit` closes an SSH session; it does not shut down the remote computer. Losing an SSH connection alone was not proof that a shutdown completed, particularly when a password error had also occurred.

For startup, the discussed order was network equipment first, followed by the computers once networking was ready. A MikroTik halted through its terminal needed a power cycle to start again; the Reset button was not used as a power button. Access to the router and switch was subsequently restored, but this phase did not record a complete restart test of every lab device.

## 7. Finding and testing the switch

On October 13, I connected the MacBook by Ethernet to the lab and inspected the **MikroTik DHCP leases**:

```routeros
/ip dhcp-server lease print detail without-paging
```

| Device | Address | Evidence |
| --- | --- | --- |
| Switch | `192.168.88.252` | Hostname `SG3210`; class ID `SG3210_3.20`; dynamic and `bound` |
| MacBook | `192.168.88.251` | Management laptop's Ethernet lease; dynamic and `bound` |

The Pi was not present in that particular supplied lease listing. Its absence alone did not diagnose a Pi fault or establish its current address.

From the **MacBook's local Terminal**:

```bash
ping -c 3 192.168.88.252
nc -vz -w 3 192.168.88.252 80
nc -vz -w 3 192.168.88.252 443
```

| Test | Observed result |
| --- | --- |
| ICMP | 3 sent, 3 received, 0% packet loss |
| Round-trip times | Minimum 3.213 ms; average 4.484 ms; maximum 6.953 ms |
| TCP 80 | Connection succeeded |
| TCP 443 | Connection succeeded |

The initial `v/ip` typo failed in RouterOS. The Mac commands also failed when entered into the router's terminal: RouterOS does not use macOS `ping -c` syntax or provide that `nc` command. Running the checks in the correct terminal resolved the command issue.

The successful TCP checks established service reachability. They did not yet verify web login, certificate trust, or application content.

## 8. Fixing Chrome access and signing in

Despite the successful Terminal checks, Chrome reported `ERR_ADDRESS_UNREACHABLE` for:

```text
https://192.168.88.252
```

Chrome's macOS Local Network permission was off. I enabled it under:

```text
System Settings > Privacy & Security > Local Network > Google Chrome
```

After refreshing, Chrome reached the switch and displayed `NET::ERR_CERT_AUTHORITY_INVALID`. Reaching the HTTPS certificate check showed that the local-network access problem had been resolved. No VLAN change or removal of MikroTik isolation rules was needed.

The certificate warning and later Not Secure indicator meant Chrome did not trust the certificate presented by the switch. HTTPS can still encrypt the connection without a browser-trusted certificate establishing the server's identity. I proceeded for the specifically identified local switch; no globally disabled certificate checks, replacement certificate, or trusted DNS-name setup was recorded.

I signed in and set a new management password. Data collection was discussed with the intention to leave it off, but the final checkbox state was not separately confirmed.

An initial configuration backup was downloaded through **System Tools → Backup Config** before the later port-description changes. The interface showed **UNIT 1**; Export and All Units Export were reviewed. The backup was kept privately, separate from the GitHub documentation.

This result applies to the switch access tested here. The earlier Proxmox Chrome error from Phase 4 was not separately retested during this phase.

## 9. Verifying the switch and shared VLAN

| Item | Recorded value |
| --- | --- |
| Model | TP-Link SG3210 / TL-SG3210 JetStream, managed L2+ switch |
| Hardware revision | `SG3210 3.20` |
| Physical ports | 8 Gigabit copper ports and 2 SFP slots |
| Firmware in System Summary | `3.20.0 Build 20230818 Rel.72032` |
| Management address | `192.168.88.252` |
| System Name | `SG3210` |

Default-looking location and time-zone values, including Hong Kong and UTC+13, were observed but not corrected. The Boot Config view contained separate image information; no firmware update was performed.

| VLAN field | Observed configuration |
| --- | --- |
| VLAN ID / name | `1` / `System-VLAN` |
| Member ports | `1/0/1` through `1/0/10` |
| Untagged membership | All ten ports |
| Tagged membership | None shown |
| PVID | `1` on every port |
| Ingress Checking | Enabled |
| Acceptable Frame Types | Admit All |

PVID identifies the VLAN assigned to an untagged frame arriving on a port. In this configuration, ordinary untagged client traffic enters VLAN 1; the untagged egress membership lets the connected devices operate without adding VLAN tags themselves.

The existing configuration matched the goal of one shared lab LAN. **No new VLAN was created, no PVID was changed, and no inter-VLAN routing was enabled.** The ten listed interfaces include the two SFP slots; they are not ten copper sockets.

SNMP and HTTP were enabled in the summary, while Spanning Tree, IGMP Snooping, and Jumbo were shown disabled. These were recorded observations, not completed hardening changes. No link aggregation or new trunk was configured.

## 10. Reserving the switch management address

On October 15, I converted the switch's existing dynamic lease to a **static DHCP lease** on the MikroTik. The address stayed `192.168.88.252` and the switch remained a DHCP client; I did not enter a manual static IP on the switch.

The recorded command selected the switch by MAC address. The public template below omits the real identifier. Replace `<SWITCH_MAC>` with the verified address before using it in an equivalent setup:

```routeros
/ip dhcp-server lease make-static [find where mac-address=<SWITCH_MAC>]
/ip dhcp-server lease print detail without-paging where mac-address=<SWITCH_MAC>
```

The change returned without a success message, so I checked the resulting lease rather than treating silence as failure.

| Field | Verified result |
| --- | --- |
| Address | `192.168.88.252` |
| Hostname | `SG3210` |
| DHCP server | `defconf` |
| Status | `bound` |
| Dynamic flag beside the entry | `D` present before the change, absent afterward |

The Flags legend still included `D` because it explains possible flags. The relevant evidence was the flag next to the actual lease entry. The active lease could still show an expiry time: reserving its assignment on the server does not stop normal DHCP renewal.

This reservation belongs to the **MikroTik configuration**. A switch export does not contain it. A new post-change router export was not recorded in this phase.

## 11. Labeling ports, saving, and exporting

I added descriptions under:

```text
L2 FEATURES > Switching > Port > Port Config
```

| Switch interface | Description | Physical connection |
| --- | --- | --- |
| `1/0/1` | `MikroTik-ether5` | Router `ether5` uplink |
| `1/0/3` | `Proxmox-N100` | N100 Proxmox server |
| `1/0/8` | `RaspberryPi-Kali` | Raspberry Pi Ethernet |

Each change selected one port, entered its description, and used the **Apply button for the port table**. The nearby Jumbo Apply button was not the control for port descriptions. The previous port was deselected before editing another.

No intentional speed, duplex, or flow-control changes were made. Copper-port speed and duplex were shown as Auto, and the ports remained enabled. Descriptions made the interface easier to read without changing addressing or VLAN membership.

| Action | Purpose in the workflow |
| --- | --- |
| Apply | Apply the edit on the current configuration screen |
| Save | Save the switch configuration for subsequent startup |
| Export | Download a separate backup copy |

After editing the descriptions, I confirmed using **Save**. I then returned to:

```text
SYSTEM > System Tools > Backup Config
```

An updated export was downloaded and saved. The menu was named System Tools, not a separate top-level Tools menu. Together with the earlier export, this provided before-and-after switch backups. A deliberate reboot-persistence test and a restore test were not performed.

## 12. End-of-phase state

| Area | State supported by the session record |
| --- | --- |
| MacBook | FileVault on; firewall enabled; own login password set; automatic login off |
| App permissions | Reviewed permissions reduced; Zoom retained for occasional use |
| Existing user data | Preserved pending confirmation of the previous owner's backup |
| Switch login | HTTPS management reached at `192.168.88.252`; new password set |
| Switch addressing | Same address reserved by a static DHCP lease on MikroTik |
| Internal network | Existing `192.168.88.0/24` LAN, VLAN 1, PVID 1 retained |
| Port labels | Router on 1, Proxmox on 3, Pi on 8 documented in the switch |
| Persistence and backup | Save confirmed; initial and updated switch exports downloaded |
| New virtual machines | None created |

The basic switch-management work was complete enough to disconnect the MacBook's lab Ethernet cable when finished. Pi access through Tailscale still depended on internet connectivity and the Tailscale client, with the Pi and its network equipment powered and connected. No subnet router was configured to make every lab management address remotely reachable.

The local firewall, Tailscale policy, and MikroTik forwarding policy operate at different boundaries. This work did not establish full switch-service hardening, remove the certificate warning, or isolate devices from each other inside the shared VLAN.

## 13. Next work

1. Confirm the previous owner's data is backed up before deleting personal files from the MacBook.
2. Confirm that the FileVault recovery key can be retrieved when the MacBook is unavailable.
3. Create and securely store an updated MikroTik export containing the switch DHCP reservation.
4. Review switch time settings, device description, firmware suitability, and management services separately.
5. Plan a maintenance window for switch restart-persistence and backup-restore validation.
6. Retest Chrome access to Proxmox if needed; the switch fix did not itself prove that separate test.
7. Complete the remaining isolation checks from Phase 5 and document the final physical layout.
8. Create the first lightweight VM once its purpose, resources, and network attachment are selected.

Later collaborator access and individual SSH-key work will be documented in their own phase, with configuration changes distinguished from successful login tests.

## 14. Lessons learned

- Inventory and backup decisions should come before deleting data on a previously used computer.
- A local firewall, an app privacy permission, and a VPN access rule are different controls.
- Successful ping and TCP checks can narrow a browser problem to a later layer.
- A certificate warning is a different failure from an unreachable address.
- Record the hardware revision before making model-specific switch changes.
- Retaining an existing VLAN can be the right decision when it already matches the design.
- A DHCP reservation changes the server's allocation, while the client can remain on DHCP.
- Inspect flags on the actual record rather than reading the legend as device state.
- Apply, Save, and Export serve different purposes; backups belong to the device whose configuration they contain.
- Record completed checks separately from operational guidance and future plans.

## Quick reference

| Item | Documented address or connection |
| --- | --- |
| Lab subnet / gateway | `192.168.88.0/24` / `192.168.88.1` |
| Switch | `https://192.168.88.252`; DHCP reservation |
| Proxmox | `https://192.168.88.2:8006`; existing configuration |
| Pi LAN address | `192.168.88.249`; last verified in Phase 6, dynamic |
| MacBook Ethernet address | `192.168.88.251`; observed DHCP lease |
| Switch mapping | Port 1 → MikroTik `ether5`; Port 3 → Proxmox; Port 8 → Pi |
| VLAN / PVID | VLAN 1 / PVID 1 on all listed ports |

### MacBook local Terminal — checks and connection

```bash
ping -c 3 192.168.88.252
nc -vz -w 3 192.168.88.252 443
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
fdesetup status
```

For the existing Pi connection, substitute its real Tailscale address:

```bash
# Publication example only.
ssh kali@100.101.10.10
```

### MikroTik terminal — read-only checks

```routeros
/ip dhcp-server lease print detail without-paging
/ping 192.168.88.252 count=3
```

Normal access checks do not require repeating `make-static` or changing VLAN membership. The shutdown instructions above are separate operational actions.

## References

- [Apple — Block connections to your Mac with a firewall](https://support.apple.com/guide/mac-help/mh34041/mac)
- [MikroTik — DHCP server](https://manual.mikrotik.com/docs/network-management/dhcp/server/)

*Source: Hebrew MacBook and switch report covering October 10–15, 2026. This chapter records that period's outputs, screenshots, and user confirmations. Software versions and unreserved DHCP addresses are historical snapshots. Commands that change configuration describe the recorded workflow, not a script to replay in full.*
