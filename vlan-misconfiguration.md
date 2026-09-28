# VLAN Misconfiguration Troubleshooting

## Scenario

`ADMIN1` is connected to `SW1` interface `Fa0/1` and normally belongs to VLAN 10 (`ADMIN`).

To simulate a common Layer 2 network configuration problem, `Fa0/1` was intentionally reassigned from VLAN 10 to VLAN 20 (`SALES`).

### Expected Configuration

| Device | Interface | VLAN | Network |
|---|---|---|---|
| ADMIN1 | FastEthernet0 | VLAN 10 (ADMIN) | 192.168.10.0/24 |
| SW1 | Fa0/1 | VLAN 10 (ADMIN) | 192.168.10.0/24 |

The fault was introduced on `SW1` with:

```text
configure terminal
interface fa0/1
 switchport access vlan 20
end
```

This placed `ADMIN1` in the SALES VLAN at Layer 2 while the workstation retained an IP configuration belonging to the ADMIN VLAN.

---

## Symptoms

`ADMIN1` retained its existing DHCP configuration:

```text
IPv4 Address:    192.168.10.101
Subnet Mask:     255.255.255.0
Default Gateway: 192.168.10.1
```

![ADMIN1 IP configuration](../screenshots/vlan-admin1-ipconfig.png)

Although the workstation's IP configuration appeared correct, it could no longer communicate with its default gateway:

```text
ping 192.168.10.1
```

The result was 100% packet loss.

![ADMIN1 failed gateway ping](../screenshots/vlan-admin1-ping-failed.png)

This indicated that the problem was not necessarily an incorrect IP address, subnet mask, or default gateway. Troubleshooting therefore moved toward Layer 2 connectivity and switch configuration.

---

## Troubleshooting Process

### 1. Verify Client IP Configuration

The first step was to inspect the workstation configuration:

```text
ipconfig
```

`ADMIN1` still had a valid `192.168.10.0/24` address and the correct default gateway of `192.168.10.1`.

This reduced the likelihood of a client-side addressing problem.

### 2. Verify Physical Interface Status

On `SW1`, interface status was checked with:

```text
show interfaces status
```

`Fa0/1` showed:

```text
Fa0/1    connected    20
```

The interface was physically connected, but it was associated with VLAN 20.

This indicated that the issue was not a disconnected cable or administratively disabled port.

### 3. Verify VLAN Membership

The VLAN table was then inspected:

```text
show vlan brief
```

The output showed:

```text
10  ADMIN   Fa0/2
20  SALES   Fa0/1, Fa0/3, Fa0/4
```

`Fa0/1`, which connects to `ADMIN1`, was incorrectly assigned to VLAN 20.

![Incorrect VLAN assignment](../screenshots/vlan-misconfiguration-found.png)

The expected configuration was:

```text
10  ADMIN   Fa0/1, Fa0/2
20  SALES   Fa0/3, Fa0/4
```

---

## Root Cause

`SW1 Fa0/1` was configured as an access port in VLAN 20 instead of VLAN 10.

Although `ADMIN1` still had the valid address:

```text
192.168.10.101/24
```

its Ethernet frames were now being placed into the VLAN 20 broadcast domain.

The workstation was therefore unable to communicate with the VLAN 10 SVI/default gateway:

```text
192.168.10.1
```

The failure occurred at Layer 2 rather than because of incorrect Layer 3 addressing or routing.

---

## Resolution

`Fa0/1` was reassigned to the correct VLAN:

```text
configure terminal
interface fa0/1
 switchport access vlan 10
end
```

The correction was verified using:

```text
show vlan brief
```

The output now showed:

```text
10  ADMIN   Fa0/1, Fa0/2
20  SALES   Fa0/3, Fa0/4
```

![Corrected VLAN assignment](../screenshots/vlan-misconfiguration-corrected.png)

---

## Verification

After restoring `Fa0/1` to VLAN 10, connectivity to the default gateway was tested again:

```text
ping 192.168.10.1
```

The result was successful:

```text
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
```

![ADMIN1 connectivity restored](../screenshots/vlan-admin1-ping-restored.png)

This confirmed that the incorrect VLAN assignment was the cause of the connectivity failure and that normal communication had been restored.

---

## Commands Used

### Client

```text
ipconfig
ping 192.168.10.1
```

### SW1

```text
show interfaces status
show vlan brief

configure terminal
interface fa0/1
 switchport access vlan 10
end

show vlan brief
```

---

## Troubleshooting Summary

| Stage | Finding |
|---|---|
| Client addressing | Correct |
| Physical link | Connected |
| Default gateway reachability | Failed |
| Switchport VLAN | Incorrect — VLAN 20 |
| Root cause | ADMIN1 access port assigned to SALES VLAN |
| Corrective action | Reassigned `Fa0/1` to VLAN 10 |
| Final verification | Gateway ping successful |

## Key Takeaway

A workstation can retain a valid IP address, subnet mask, and default gateway while still losing connectivity if the switch access port is assigned to the wrong VLAN.

Checking the problem in layers helped isolate the fault:

1. Verify endpoint addressing.
2. Verify physical interface status.
3. Verify VLAN membership.
4. Correct the switchport configuration.
5. Retest connectivity.

The `show interfaces status` and `show vlan brief` commands were sufficient to identify the Layer 2 misconfiguration.
