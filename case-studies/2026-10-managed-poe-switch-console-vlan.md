# Case study: Is this switch managed? Console diagnostics and VLAN setup on a 24-port PoE switch

> **Privacy checklist before publishing** - done: no employer or site names, no IP
> addresses, no host names, no passwords,
> no logs from the production network. Only vendor CLI syntax is shown.

| Field | Value |
|---|---|
| Date | 2026-10 |
| Area | Networking |
| Technologies | 24-port Gigabit PoE L2 switch (4 SFP), USB-to-RS232 console cable (PL2303), Windows, PowerShell `System.IO.Ports.SerialPort` |
| Status | Done |

## Context

A small lab needed separate networks for IP phones, lab workstations and an uplink to a
VoIP gateway and a router. The only device available was a 24-port PoE switch with a
"Console" port, and nobody knew whether it was really manageable or the port was just
decorative. I had a Windows laptop with a USB console cable and remote shell access to it.

## Problem / Symptoms

- Unknown if the switch supports CLI and 802.1Q VLANs at all.
- No documentation at hand, no known credentials.
- The task description assumed Linux tooling (`/dev/ttyUSB*`, `screen`, `picocom`), but the
  laptop runs Windows.

## Scope and Constraints

- Diagnostics only until the owner confirmed changes; no reset or configuration without approval.
- Do not touch the laptop's VPN/overlay interface or default routes.
- Do not brute-force passwords (risk of lockout).

## Diagnostic Process

1. **Find the console adapter (Windows equivalent of `ls /dev/ttyUSB*` + `dmesg`).**
   `[System.IO.Ports.SerialPort]::GetPortNames()` and `Get-PnpDevice -Class Ports` showed
   a Prolific PL2303 adapter (`VID_067B&PID_2303`) on `COM4` with a working driver.
2. **Probe baud rates, sending only Enter** (8N1, no flow control, 3 s wait):

   | Baud | Result |
   |---|---|
   | 115200 | silence |
   | 9600 | `???` (garbage - wrong speed) |
   | 38400 | `Username:` / `Password:` prompt |

3. **Check credentials once.** Factory default `admin`/`admin` was rejected - the password
   had been changed earlier. Stopped there instead of guessing.
4. **Vendor documentation** confirmed: console 38400 8N1, default `admin`/`admin`, default
   management IP in VLAN 1, web + telnet + Cisco-like CLI, access/trunk/hybrid ports.
5. **Owner performed a factory reset**, after which login worked and read-only commands
   were collected: `show version`, `show ip interface brief`, `show vlan`,
   `show interface brief`, `show running-config`.
6. **Learned the exact syntax with `?`** in configuration mode before applying anything
   (`vlan database`, `vlan-name`, `switchport mode/access/trunk`, `username ... password`).

## Hypotheses

| Hypothesis | How it was checked | Result |
|---|---|---|
| Console port is decorative (unmanaged switch) | Baud probe | Rejected - login prompt at 38400 |
| Default credentials still valid | One login attempt | Rejected - changed earlier |
| Switch supports 802.1Q VLANs from CLI | `show vlan`, `?` in config mode | Confirmed |

## Root Cause

Not a fault - an unknown device. It is a managed L2 switch with CLI; the "silence" people
expected is just a non-standard console speed (38400 instead of 9600/115200), and the
admin password had been changed by a previous owner.

## Solution

After approval, configured via console (out-of-band, so a VLAN mistake cannot cut off access):

| VLAN | Name | Ports | Mode |
|---|---|---|---|
| 1 | default | port 1 (management laptop) + remaining ports | access |
| 10 | voice | ports 11-12 (IP phones) | access |
| 20 | LAB | ports 2-10 (lab PCs) | access |
| 30 | UPLINK | reserved | - |
| 10, 20 | - | port 28 (SFP, VoIP gateway / router) | trunk |

Example syntax (this firmware family):

```
configure terminal
vlan database
 vlan 10,20,30
 vlan-name 10 voice
 vlan-name 20 LAB
 vlan-name 30 UPLINK
exit
interface ge1/11
 switchport mode access
 switchport access vlan 10
exit
interface ge1/28
 switchport mode trunk
 switchport trunk allowed vlan add 10
 switchport trunk allowed vlan add 20
exit
end
write
```

Baseline hardening in the same change: new strong admin password (stored outside the repo),
removed the default read-only SNMP community `public`, time zone set to local, hostname set.

## Verification

- Logged in again with the new password before saving (the switch drops the session right
  after the password change - a useful sign that it took effect).
- `show vlan`: VLAN 10 `voice` = ports 11-12 untagged + port 28 tagged; VLAN 20 `LAB` =
  ports 2-10 untagged + port 28 tagged; VLAN 30 `UPLINK` created; port 1 stays in VLAN 1.
- `show interface brief`: ports 11-12 `Access(10)`, ports 2-10 `Access(20)`, port 28 `Trunk`,
  links that were up before the change stayed up.
- `show running-config`: SNMP `public` removed, time zone and hostname set.
- Only then `write`; the switch confirmed "Building and writing configuration".
- One command was rejected: explicit `switchport access vlan 1` - this firmware does not accept
  VLAN 1 there, and it is the default anyway.

## Security and Privacy Considerations

- Factory defaults (`admin`/`admin`, SNMP `public`) are a real exposure on any switch that has
  an IP in a user VLAN; both were changed.
- Management stays in VLAN 1 on a dedicated port plus the console; user VLANs have no
  management IP.
- Anonymised: site, addresses, host names and the actual password are not published.

## Lessons Learned

- "Console is silent" is often a wrong baud rate. Probe 9600, 38400 and 115200 before
  concluding a port is dead.
- On Windows, PowerShell `SerialPort` is enough for scripted console work; no PuTTY needed.
- PowerShell gotcha: a helper function named `Rd` is shadowed by the built-in alias `rd`
  (`Remove-Item`) - aliases win over functions. Use unambiguous names.
- Apply to running-config first, verify (including logging in with the new password), and
  only then save - a reboot is the rollback.

## What I Would Improve

- Move management to a dedicated management VLAN instead of VLAN 1.
- Add SNMPv3 or syslog to a monitoring server instead of plain SNMP v2c.
- Keep an exported config backup and a port map next to the rack.
