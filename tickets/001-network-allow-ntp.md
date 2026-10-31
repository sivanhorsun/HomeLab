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
