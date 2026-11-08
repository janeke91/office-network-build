# 08. Active Directory (corp.lab)

A small Windows domain for practicing the identity side of an IT support / junior sysadmin job: users, groups, Group Policy, delegation and file permissions.

## Design

| Item | Value |
|---|---|
| Domain controller | `dc-01`, Windows Server 2022 (evaluation), VLAN 20 INFRA, `10.10.20.5` |
| Domain / NetBIOS | `corp.lab` / `CORP` |
| DNS | AD-integrated DNS on `dc-01`; forwarder to the pfSense resolver `10.10.20.1` |
| Client | Windows 11 Enterprise (evaluation), joined to the domain, DHCP from pfSense with DNS `10.10.20.5` |
| Remote access | RDP to `dc-01` only from `ADMIN_HOSTS` (rule on the MGMT interface, see [03](03-firewall-rules.md)) |

The domain name `corp.lab` is used for the lab only. A production design would use a subdomain of a name the company owns (for example `corp.example.com`) and never a made-up suffix or `.local`.

## Structure

OUs hold objects and are the target of Group Policy and delegation. Groups hold permissions.

| OU | Contents |
|---|---|
| `IT` | IT staff accounts |
| `Staff` | regular employee accounts |
| `Workstations` | domain-joined client computers |
| `Groups` | security groups |

| Group | Type / scope | Purpose |
|---|---|---|
| `IT` | Security / Global | helpdesk and administrators |
| `Accounting` | Security / Global | accounting department users |
| `FS_Accounting_RW` | Security / Domain local | Modify rights on the Accounting share (resource group) |

## Group Policy

| GPO | Linked to | Settings |
|---|---|---|
| Default Domain Policy | domain root | password history 10, maximum age 60 days, minimum age 1 day, minimum length 10, complexity on; lockout after 5 failed attempts, 15 minutes lockout and reset |
| `Staff - Restrict Control Panel` | OU `Staff` | Control Panel and Settings prohibited, desktop wallpaper locked, software installation and removal blocked |
| `Workstations - IT admins` | OU `Workstations` | Group Policy Preferences: `CORP\IT` added to local `Administrators` and `Remote Desktop Users` on every workstation |

Password and lockout rules are only effective for domain accounts when set in the policy linked at the **domain root** (Default Domain Policy). The same settings in Default Domain Controllers Policy have no effect on user accounts.

## Delegation

The `IT` group can reset passwords for users in OU `Staff` only (Delegation of Control: *Reset user passwords and force password change at next logon*). Verified in the ACL of the OU: `ExtendedRight` Reset Password and `WriteProperty` on `pwdLastSet`, both inherited by descendant user objects. Result: IT can reset a Staff password but not the password of `Administrator`.

## File share (AGDLP)

Path `C:\Shares\Accounting`, shared as `\\dc-01\Accounting`.

- Share permissions: Everyone, Full Control (the share is not the control point).
- NTFS permissions: inheritance disabled; `SYSTEM`, `Administrators` and `Administrator` Full Control; `FS_Accounting_RW` Modify.
- Accounts go into `Accounting` (global group), which is a member of `FS_Accounting_RW` (domain local group), which holds the permission. Adding a new accountant means one change: group membership.

## Verification

| Test | Result |
|---|---|
| Client resolves `corp.lab` and `dc-01` through `10.10.20.5` | OK |
| `dc-01` resolves external names through the pfSense forwarder | OK |
| `gpresult /r` on the client lists the Staff and Workstations GPOs | OK |
| Staff user opens Control Panel: "Operation has been cancelled due to restrictions" | OK (policy applied) |
| Accounting user reads and writes `\\dc-01\Accounting` | OK |
| Non-Accounting user opens `\\dc-01\Accounting` | Access is denied (expected) |
| IT member resets a Staff user's password in Active Directory Users and Computers | OK |

## Lessons learned

- **RDP timeout to `dc-01`:** the firewall rule was created on the INFRA tab. Rules are evaluated on the interface where traffic enters, so a rule for traffic coming from the admin laptop belongs on MGMT.
- **`net user <name> * /domain` returns "System error 5" for a delegated helpdesk user** although the delegation is correct. Remote SAM calls are restricted to administrators by default on current Windows. Resetting through Active Directory Users and Computers (LDAP) works; this is also how helpdesk teams do it.
- **`dc-01` could not resolve external names ("Server failed" from the pfSense resolver)** on the first queries after the forwarder was added; after clearing the DNS cache on `dc-01` resolution worked. Check order: pfSense DNS Lookup, then the forwarder, then the cache.
- **Edited the wrong policy for passwords:** account policy lives in Default Domain Policy, not Default Domain Controllers Policy.
- **A GPO setting did not apply** until the user logged off and on (User Configuration) or the computer restarted (Computer Configuration). `gpupdate /force` and `gpresult /r` are the first checks.
- **Moving an OU failed with "Access is denied":** OUs are protected from accidental deletion by default. The computer object had to be moved, not the OU.

Screenshots: [screenshots/active-directory](../screenshots/active-directory)
