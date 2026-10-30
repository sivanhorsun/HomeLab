# [PROPOSAL] Introduce Centralized IT Management Tools

**Ticket ID:** 002  
**Type:** Infrastructure Improvement / Technical Proposal  
**Priority:** Normal  
**Status:** Proposed  
**Assigned Team:** Systems Team

## Objective

Introduce centralized, self-hosted tools to improve infrastructure management, security, collaboration, and operational organization across our homelab projects.

The objective is to adopt practices commonly used in enterprise IT environments while utilizing free and open-source software.

## Proposed Tools

### 1. Identity and Access Management (IAM)

**Proposed solution:** Authentik

Purpose:
- Centralized user and group management.
- Individual user identities.
- Role-based access control.
- Single Sign-On (SSO).
- Multi-factor authentication (MFA).
- LDAP / OIDC integration.

Proposed departments/groups:
- Systems
- Network
- Security
- Development

### 2. Secrets Management

**Proposed solution:** OpenBao

Purpose:
- Centralized storage of passwords, tokens, and secrets.
- Secure access to credentials through APIs.
- Secret lifecycle management.
- Credential rotation where supported.
- Integration with automation scripts.

Future objective:

Develop automation that can provision Linux users and configure their credentials using centrally managed secrets.

### 3. IT Service Management (ITSM)

**Proposed solution:** GLPI

Purpose:
- Centralized ticket management.
- Department-based ticket assignment.
- Incident and service request tracking.
- Change management.
- Infrastructure asset inventory.
- Operational documentation.

Example ticket categories:
- Network Requests
- System Administration
- Security Incidents
- Hardware Requests
- Infrastructure Changes

### 4. Infrastructure Automation

**Proposed solution:** Ansible

Purpose:
- Automated Linux user provisioning.
- User and group management.
- Server configuration.
- Package installation.
- Configuration management.
- Repeatable infrastructure deployments.

## Proposed Architecture

The proposed environment will consist of:

| Component | Responsibility |
|---|---|
| Authentik | Identity and access management |
| OpenBao | Secrets management |
| GLPI | ITSM and ticket management |
| Ansible | Infrastructure automation |
| Linux servers / VMs | Managed infrastructure |

## Implementation Plan

- [ ] Evaluate the proposed tools.
- [ ] Determine hardware and resource requirements.
- [ ] Deploy GLPI.
- [ ] Establish departments and ticket categories.
- [ ] Deploy Authentik.
- [ ] Configure users, groups, and permissions.
- [ ] Deploy OpenBao.
- [ ] Define secrets management policies.
- [ ] Introduce Ansible automation.
- [ ] Test automated Linux user provisioning.
- [ ] Document the architecture and operational procedures.

## Expected Benefits

- Improved accountability and traceability.
- Consistent user and access management.
- Reduced manual administration.
- Centralized ticket tracking.
- Better infrastructure documentation.
- Reproducible deployments.
- Improved security practices.

## Additional Considerations

All proposed solutions should be evaluated for licensing, maintenance requirements, security, resource consumption, and compatibility before deployment.

Individual user accounts should remain the standard. Shared departmental credentials should be avoided where possible.

Credential rotation policies should be based on security requirements and the capabilities of the systems involved rather than relying exclusively on arbitrary monthly password changes.

## Expected Outcome

Establish a structured, maintainable, and secure IT management environment that can grow alongside our homelab infrastructure and projects.
