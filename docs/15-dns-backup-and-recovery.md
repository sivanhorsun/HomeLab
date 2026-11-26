# Phase 15 — Lab DNS, Remote Router Access, and Recovery Testing

**Result:** BIND9 now runs in Debian container 102 at `192.168.88.237`. MikroTik DHCP supplies that resolver and the `home.arpa` search domain. VM 100 resolved the short lab names and public internet names successfully. Remote MikroTik management works through Proxmox using SSH and WinBox. Container 102 was added to the daily backup job, restored into an isolated test container, and copied to the MacBook with matching SHA-256 checksums.

**Scope:** Follow-up work after [Phase 14](14-almalinux-ansible-and-remote-management.md), covering the recorded cleanup, access, and DNS sessions through November 26, 2026. Historical resource measurements below describe their recorded checkpoints.

Project overview: [README](../README.md).

## 1. Storage cleanup and VM resources

Both existing VMs were retained. Two unused AlmaLinux installer ISOs were deleted after the installer image was detached from VM 101's virtual CD drive:

```text
AlmaLinux-10.2-x86_64-minimal.iso
AlmaLinux-10-latest-x86_64_v2-minimal.iso
```

The Proxmox APT archive cache had occupied approximately 632 MiB and was cleaned. The approximately 41 MiB of logs and existing VM backup archives were retained. Reported `local` storage use fell from 20,277,772 to 16,307,064 KiB, freeing approximately **3.79 GiB**.

| Measurement after cleanup | Used | Available |
| --- | ---: | ---: |
| `local` | Approximately 15.6 GiB | Approximately 48.7 GiB |
| `local-lvm` | Approximately 6.8 GiB | Approximately 134.4 GiB |

Subsequent container creation, updates, and backups changed storage use further. These values are the cleanup checkpoint rather than current capacity.

VM 101 was shut down gracefully and confirmed stopped before its configured memory was reduced from **4096 to 2048 MiB**. It was then started again. Its four CPU cores and 60 GiB virtual disk remained in place. Afterward, Proxmox reported 7.5 GiB total RAM, 4.8 GiB used, 2.7 GiB available, and approximately 878 MiB of swap in use.

## 2. OpenBao after the VM restart

The AlmaLinux guest contained `openbao.service`, launched with configuration under `/etc/openbao.d`. The observed OpenBao version was `2.6.5-1.el10_2.alma_altarch`, with file storage and HA disabled.

OpenBao used a Shamir seal with five key shares and a threshold of three. It was sealed after the restart. Three distinct shares were entered interactively, and the status changed to **`Sealed: false`**. Reusing the same share did not advance unseal progress. No key material is included in this documentation.

Read-only status check inside VM 101:

```bash
bao status -address=http://127.0.0.1:8200
```

The explicit HTTP address matches the local service. The CLI's default HTTPS address previously produced an HTTP/HTTPS mismatch. A future restart can require unsealing again. The collaborator reported that the application was empty; the session did not independently inventory stored secrets or migrate OpenBao.

## 3. Remote management through Proxmox

The MacBook reaches the existing `proxmox` SSH alias over Tailscale. Proxmox then provides access to the router's lab address. Raw Tailscale addresses, personal workstation paths, and credentials are omitted from this public chapter.

### MikroTik SSH

Initial attempts produced `Bad packet length`, a message authentication code error, authentication timeouts, and password rejection. Explicitly selecting `aes256-ctr` and `hmac-sha2-256` allowed the SSH handshake to complete. The router's `admin` account was enabled, belonged to the `full` group with SSH permission, and had no source-address restriction. Its imported SSH-key list was empty.

A connection initiated by the MacBook's SSH client with Proxmox as the jump host succeeded:

```bash
ssh -c aes256-ctr -m hmac-sha2-256 -J proxmox admin@192.168.88.1
```

The following host entry was added to the MacBook's existing `~/.ssh/config` and tested successfully. This is a configuration block, not a shell command:

```sshconfig
Host mikrotik
    HostName 192.168.88.1
    User admin
    ProxyJump proxmox
    Ciphers aes256-ctr
    MACs hmac-sha2-256
```

The saved shortcut is:

```bash
ssh mikrotik
```

The exact cause of the earlier nested SSH authentication failures was not established. Router authentication policy and the admin password were not reset during this work.

### WinBox

Checks from Proxmox found router ports 80 and 443 refusing connections. This was consistent with the earlier decision to disable WebFig. TCP 8291 was open, and WinBox login succeeded through a local SSH forward.

Run in a MacBook Terminal tab and keep that tab open:

```bash
ssh -N -L 127.0.0.1:18291:192.168.88.1:8291 proxmox
```

| WinBox field | Tested value |
| --- | --- |
| Connect to | `127.0.0.1:18291` |
| Login | `admin` |
| Password | Enter the router password locally |

WinBox 4.5 was used on macOS. RouterOS was still shown as **7.23.8**; an available-update banner did not establish that an update had been installed.

### Proxmox and container access

For remote Proxmox browser access, run this in the MacBook Terminal and leave it open:

```bash
ssh -N -L 127.0.0.1:8006:127.0.0.1:8006 proxmox
```

Open `https://127.0.0.1:8006` in the browser. The recorded login uses `root` with the Linux PAM realm. A later session initially timed out, then browser access resumed; no underlying fault was isolated.

To enter the active DNS container from another MacBook Terminal tab:

```bash
ssh -t proxmox 'pct enter 102'
```

This produces the container prompt `root@dns-lab-01:~#`. BIND configuration commands run inside that container; `pct` and `qm` commands belong on the Proxmox host.

## 4. Debian DNS container

| Setting | Confirmed value |
| --- | --- |
| ID / hostname | CT 102 / `dns-lab-01` |
| Template | `debian-12-standard_12.12-1_amd64.tar.zst` |
| CPU | 1 core |
| Memory / swap | 512 MiB / 256 MiB |
| Root disk | 4 GiB on `local-lvm` |
| Container privilege | Unprivileged |
| Nesting | Enabled after the systemd 252 warning |
| Network | `eth0`, bridge `vmbr0`, IPv4 DHCP |
| Address | `192.168.88.237/24`, reserved on MikroTik |
| Start at boot | Enabled; behavior after a full host reboot remains untested |

The template download and checksum verification completed successfully. Debian packages were updated, then `bind9` and `bind9-dnsutils` were installed. The observed BIND version was `9.18.49-1~deb12u3`.

CT 102 remains a DHCP client. Its matching MikroTik lease was converted to a static lease with **Make Static**, reserving the current address for that client.

## 5. BIND configuration and the initial resolution failure

The initial local query for `example.com` returned `SERVFAIL`. A direct query to `1.1.1.1` succeeded, and the named journal showed unsuccessful IPv6 resolution attempts with `network unreachable`.

The startup options in `/etc/default/named` were changed to:

```text
OPTIONS="-u bind -4"
```

A `.bak` copy of the original file was created. After restarting named, the localhost query succeeded with `NOERROR` and the `ad` flag. The change and restart preceded recovery; the initial failure's precise cause was not independently isolated.

### Resolver access and listeners

The previous `/etc/bind/named.conf.options` was saved as `named.conf.options.before-lab`. The resulting configuration is:

```conf
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-query { localhost; 192.168.88.0/24; };
    allow-recursion { localhost; 192.168.88.0/24; };
    allow-query-cache { localhost; 192.168.88.0/24; };
    allow-transfer { none; };

    dnssec-validation auto;

    listen-on { 127.0.0.1; 192.168.88.237; };
    listen-on-v6 { none; };
};
```

The server accepts queries and recursion from localhost and the lab IPv4 subnet. Zone transfers are disabled. No forwarders were configured in this file.

### Local zone

The original `/etc/bind/named.conf.local` was backed up. Below its existing comments, this declaration was added:

```conf
zone "home.arpa" {
    type primary;
    file "/etc/bind/db.home.arpa";
};
```

The zone file `/etc/bind/db.home.arpa` contains:

```dns
$TTL 300
@ IN SOA dns.home.arpa. hostmaster.home.arpa. (
    2026112501 ; Serial
    3600       ; Refresh
    900        ; Retry
    604800     ; Expire
    300        ; Negative cache TTL
)

@       IN NS dns.home.arpa.

dns     IN A 192.168.88.237
proxmox IN A 192.168.88.2
router  IN A 192.168.88.1
```

The names remain short: `dns`, `proxmox`, and `router`. Fully qualified names end in `home.arpa`. Future record edits require a serial increment and configuration checks before reload.

Read-only zone validation inside CT 102:

```bash
named-checkzone home.arpa /etc/bind/db.home.arpa
```

Read-only configuration and zone-load validation inside CT 102:

```bash
named-checkconf -z
```

Both checks succeeded, and the zone loaded with serial `2026112501`. The validated configuration was then applied with a service reload, which changes the running service configuration:

```bash
systemctl reload named
```

## 6. DHCP distribution and client verification

MikroTik continued to provide DHCP and the default gateway. Its DHCP network entry was changed to advertise the new resolver and search domain. The settings change was performed in RouterOS:

```text
/ip dhcp-server network set 0 dns-server=192.168.88.237 domain=home.arpa
```

The subsequent printout confirmed:

| DHCP setting | Verified value |
| --- | --- |
| Network | `192.168.88.0/24` |
| Gateway | `192.168.88.1` |
| DNS server | `192.168.88.237` |
| Domain | `home.arpa` |

| Test | Confirmed result |
| --- | --- |
| CT 102: `dig @127.0.0.1 example.com` | `NOERROR`, public IPv4 answers, `ad` flag |
| CT 102: `dig @127.0.0.1 proxmox.home.arpa` | `NOERROR`, authoritative `aa`, `192.168.88.2` |
| Proxmox: explicit UDP query to `192.168.88.237` | Local name answered correctly with `aa` |
| Proxmox: `example.com` query to `192.168.88.237` over TCP | `NOERROR`, public answers, `ad` flag |
| VM 100: `resolvectl status --no-pager` | `ens18` used DNS `192.168.88.237` and domain `home.arpa` |
| VM 100: `resolvectl query proxmox` | Expanded to `proxmox.home.arpa`, answered `192.168.88.2` |
| VM 100: `resolvectl query router dns example.com` | Router `.1`, DNS `.237`, and public A/AAAA answers |

The tests establish both explicit resolver reachability and automatic client use through DHCP. Proxmox itself still used the router as its configured default resolver during the explicit tests, so this does not demonstrate that every statically configured system switched to CT 102.

The public replies carrying `ad` show validation at BIND. VM 100's systemd-resolved reported local DNSSEC support as `no/unsupported`; the private `home.arpa` zone was unsigned. Receiving AAAA records does not establish working IPv6 connectivity.

## 7. Daily backups now include CT 102

On November 26, **Datacenter → Backup → Edit** was used to add CT 102 while retaining VMs 100 and 101. The updated selection was verified in the job list.

| Setting | Verified value |
| --- | --- |
| Guests | `100,101,102` |
| Schedule | Daily at 02:00 in the Proxmox host's time zone |
| Storage | `local` |
| Mode | Snapshot |
| Compression | ZSTD |
| Retention | `keep-last=3`, per guest |

**Run now** was then used to test the complete job. The task ran from **13:19:13 to 13:20:18** in the displayed host time and ended with **OK**. The CT 102 archive appeared in the backup list:

```text
vzdump-lxc-102-2026_11_26-13_20_10.tar.zst
```

The interface reported **203.18 MB**, also displayed as **193.77 MiB** in the storage view. The next scheduled run including CT 102 had not yet occurred at the end of this session; the manual job verified the updated selection.

## 8. Isolated restore into CT 103

The first restore dialog was opened from CT 102's own Backup tab. Its title read **Overwrite Restore: CT 102**. That dialog was cancelled before any restore was submitted.

The restore was instead initiated from **local (proxmox) → Backups**, allowing a new target ID:

| Restore setting | Selected value |
| --- | --- |
| Source | CT 102 archive from November 26 at 13:20:10 |
| Target ID | CT 103 |
| Storage | `local-lvm` |
| Unique | Enabled; a different MAC address was confirmed |
| Start after restore | Unchecked |
| Privilege level | From Backup |

The restore created `vm-103-disk-0`, extracted the archive, and ended with **TASK OK**. Before starting CT 103, its `eth0` adapter was edited under **Network → Advanced** and **Disconnect** was enabled. This kept the recovery test isolated from the active DNS server.

CT 103 started successfully. Root console login worked using the credentials retained in the backup. The original hostname `dns-lab-01` was also retained, so the Proxmox ID was used to distinguish the test copy from production.

Read-only validation in **CT 103's console**:

```bash
named-checkconf -z
```

The restored `home.arpa` zone and the default zones loaded successfully. The local authoritative response was then tested:

```bash
dig @127.0.0.1 proxmox.home.arpa
```

| Restored-service evidence | Observed value |
| --- | --- |
| Query time | November 26, 2026 at 19:42:45 UTC |
| DNS status | `NOERROR` |
| Flags | `qr aa rd ra` |
| Answer | `proxmox.home.arpa. 300 IN A 192.168.88.2` |
| Zone serial from validation | `2026112501` |

This demonstrated recovery of the container, BIND configuration, zone data, and a working localhost DNS service. External recursive resolution was not tested in CT 103 because its network remained disconnected.

After the successful test, CT 103 was shut down; the shutdown task returned **OK**. Its **Start at boot** setting was changed to **No**. It remains retained, stopped, and disconnected. CT 102 remains the active DNS server.

## 9. Verified backup copy on the MacBook

The CT 102 archive was copied to the MacBook from a separate local Terminal tab. This command writes the destination file and would replace an existing file with the same name; the Proxmox source remains in place:

```bash
scp proxmox:/var/lib/vz/dump/vzdump-lxc-102-2026_11_26-13_20_10.tar.zst ~/Downloads/
```

SCP reached **100%** and reported approximately **194 MB** transferred in **14 seconds**. The source checksum was then read through SSH:

```bash
ssh proxmox 'sha256sum /var/lib/vz/dump/vzdump-lxc-102-2026_11_26-13_20_10.tar.zst'
```

The downloaded copy was checked locally on macOS:

```bash
shasum -a 256 ~/Downloads/vzdump-lxc-102-2026_11_26-13_20_10.tar.zst
```

Both commands returned exactly:

```text
fc590ba755f98bd67b8027a96c0decd7fe689e0772e3521f271f8bbc5d85705e
```

The copy is therefore verified against the source. Its last confirmed location is **`~/Downloads/`**. Moving it into a `HOMELAB/Backups` folder was suggested but not confirmed. This was a one-time transfer; automatic off-host copying was not configured.

## 10. End-of-phase state and remaining work

| Area | Confirmed state | Still open |
| --- | --- | --- |
| Active DNS | CT 102 serves the lab and public queries; DHCP distribution works on VM 100 | Verify startup behavior after a planned Proxmox host reboot |
| Router access | MacBook SSH shortcut and WinBox tunnel both work through Proxmox | Exact cause of the earlier SSH failures remains unresolved |
| Scheduled backup | Job selection includes 100, 101, and 102; manual run ended OK | Observe a subsequent scheduled run including CT 102 |
| DNS recovery | Isolated CT 103 passed zone-load and local authoritative-query checks | CT 103 is retained off and disconnected; removal has not been performed |
| Off-host DNS backup | Archive downloaded to MacBook and SHA-256 verified | Choose a durable destination and automate future copies if desired |
| Other guests | VM 100 and VM 101 remain in the backup job | A current off-host copy and restore test for VM 101 remain unverified |
| Earlier Authentik and Ansible work | Previous milestones remain documented in Phase 14 | Authentik/PostgreSQL recovery and Ansible against another host remain open |

The active lab should remain powered while clients depend on CT 102 for DNS. Closing the MacBook's management windows ends the remote sessions and tunnels; it does not shut down the guests. Reconnect Tailscale and reopen the required tunnel when returning.

The completed checkpoint is a working lab DNS service with a tested local restore and a checksum-verified off-host backup copy. The next operational follow-up is to inspect the first scheduled run after CT 102 was added and decide how future off-host copies should be maintained.
