# [NETWORK] Allow NTP Traffic (UDP/123)

**Ticket ID:** 001  
**Type:** Network Access Request  
**Priority:** Normal  
**Status:** Open  
**Assigned Team:** Network Team  
**Requested By:** Systems Team

## Description

Request firewall configuration changes to allow Network Time Protocol (NTP) communication between the lab infrastructure and approved NTP servers.

## Technical Requirements

| Parameter | Value |
|---|---|
| Protocol | UDP |
| Destination Port | 123 |
| Direction | Outbound |
| Source | Lab network / server IP addresses |
| Destination | Approved NTP server IP addresses |
| Service | NTP |

## Business Justification

Accurate time synchronization is essential for reliable infrastructure operations.

NTP synchronization helps ensure:

- Consistent timestamps across servers.
- Accurate system and application logs.
- Easier troubleshooting and event correlation.
- Reliable authentication and security operations.

## Requested Action

1. Review the requested network access.
2. Verify the source and destination IP ranges.
3. Implement the required firewall rules.
4. Confirm once the changes have been implemented.

## Validation

The Systems Team will verify NTP connectivity and synchronization after implementation.

Example Linux commands:

```bash
timedatectl status
timedatectl timesync-status
```

Additional network testing may be performed using appropriate NTP diagnostic tools.

## Acceptance Criteria

- [ ] Firewall rules reviewed and approved.
- [ ] UDP port 123 permitted to approved NTP destinations.
- [ ] NTP connectivity verified.
- [ ] System time synchronization confirmed.
- [ ] Network Team confirms completion.

## Notes

Actual source subnets and destination NTP server addresses must be confirmed before implementation.

## Validation Update — 2026-11-02

NTP checks completed:

- Raspberry Pi: NTP service active; clock synchronized.
  NTP server response details confirmed with `timedatectl timesync-status`.
- Proxmox host: NTP service active; clock synchronized.
  Synchronization details confirmed with `chronyc tracking`.
- Ubuntu VM 100: NTP service active; clock synchronized,
  as reported by `timedatectl status`.

No firewall changes were required for these checks.

### Final Update — 2026-11-03

Gavriel confirmed that NTP synchronization also works on his VM.

All checked machines report working time synchronization.
No firewall changes were required.

The request is closed. Restricting NTP traffic to an approved
server list, if desired, will be handled as a separate change.
