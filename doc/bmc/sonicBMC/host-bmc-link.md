# USB0/BMC0 interface: Switch-Host ↔ Switch-BMC #

## 1. Revision

| Rev | Date | Author | Change Description |
|:---:|:----:|:------:|:-------------------|
| 0.1 | 05/2026 | | Initial version |

### 2. Scope  

This document covers **SONiC on the host (Switch-Host)** and **SONiC on the BMC (Switch-BMC)**.  
It describes how the **USB Ethernet link** (logical interface name `usb0` or `bmc0`) is configured so both ends have  
static addresses, bidirectional ping works, and configuration is re-applied after host-only or BMC-only restart.

### 3. Definitions/Abbreviations 

| Term | Meaning |
|------|---------|
| **Switch-Host** | SONiC on the switch CPU; `platform_env.conf` sets `switch_host=1`. |
| **Switch-BMC** | SONiC on the BMC; `platform_env.conf` sets `switch_bmc=1`. |
| **`bmc.json`** | JSON with `bmc_if_name`, `bmc_if_addr`, `bmc_addr`, `bmc_net_mask` — same schema on both sides; values must agree on the link. |
| **`interfaces.j2`** | Jinja2 template rendered to `/etc/network/interfaces`. |
| **`interfaces-config.sh`** | Regenerates `/etc/network/interfaces` and restarts `networking.service`. |
| **ifupdown2** | Debian `networking.service` backend; `ifup` applies `auto <iface>` stanzas. |

### 4. Requirements

1. **Two-way connectivity:** Switch-Host and Switch-BMC each configure the link interface with consistent bmc.json.  
2. **Reachability:** From Switch-Host, ping `bmc_addr`. From Switch-BMC, ping `bmc_if_addr` (Switch-Host side of the link).  
3. **Recovery after partial restart:** After host restart (BMC up) or BMC restart (host up), the link must come back.

### 5. Overview 

The link is configured from `bmc.json` (located in the platform folder - host platform or BMC platform):

```json
{
    "bmc_if_name": "usb0",
    "bmc_if_addr": "169.254.0.2",
    "bmc_addr": "169.254.0.1",
    "bmc_net_mask": "255.255.255.252"
}
```

**Switch-Host** renders `bmc_if_addr` into `/etc/network/interfaces`:

```text
auto usb0
iface usb0 inet static
    address 169.254.0.2
    netmask 255.255.255.252
```

**Switch-BMC** renders `bmc_addr` into `/etc/network/interfaces`:

```text
auto usb0
iface usb0 inet static
    address 169.254.0.1
    netmask 255.255.255.252
```

SONiC turns the template into `/etc/network/interfaces` using `sonic-cfggen` (from `interfaces-config.sh`).  
`networking.service` then reads `/etc/network/interfaces` and brings up `auto usb0`.

**Relevant changes are part of the following PRs:**  
Generic changes and new py-common API's for BMC based platforms  
https://github.com/sonic-net/sonic-buildimage/pull/26544  
Add NVIDIA AST2700 BMC platform support  
https://github.com/sonic-net/sonic-buildimage/pull/27038

### 6. High-Level Design 

#### 6.1 Host side (SONiC)

**After host restart:**  
`interfaces-config.service` runs `interfaces-config.sh` → `sonic-cfggen` + `interfaces.j2` writes `/etc/network/interfaces`,  
then `systemctl restart networking` → networking.service → ifupdown2 runs `ifup` for each `auto` stanza (e.g. `auto usb0`).

**After BMC restart:**  
`interfaces-config` does not run again by itself. When the USB device is (re)added, the HW MGMT UDEV rules detect the event and run `ifup usb0`.  
https://github.com/Mellanox/hw-mgmt/blob/master/usr/lib/udev/rules.d/70-hw-management-bmc.rules  
https://github.com/Mellanox/hw-mgmt/blob/master/usr/usr/bin/hw-management-ifupdown.sh

#### 6.2 BMC side (SONiC BMC)

**`systemd-networkd` `.network` file:** HW MGMT still ships a template such as `00-hw-management-bmc-usb0.network` (OpenBMC-style).  
On Sonic BMC, `systemd-networkd` is masked so that path is not used.  
See https://github.com/sonic-net/sonic-buildimage/pull/25223.

**After BMC restart:**  
* `interfaces-config.service` runs `interfaces-config.sh` → `sonic-cfggen` + `interfaces.j2` writes `/etc/network/interfaces`,  
then `systemctl restart networking` → networking.service → ifupdown2 runs `ifup` for each `auto` stanza (e.g. `auto usb0`).  
* https://github.com/Mellanox/hw-mgmt/blob/master/bmc/usr/usr/bin/hw-management-bmc-ready-common.sh  
loads `g_ether` if needed, brings `usb0` up and sets the address.

**After host restart:**  
On the BMC, `usb0` often stays; carrier may drop while the host reboots.  
BMC-side UDEV (e.g. `/usr/lib/udev/rules.d/5-hw-management-bmc-events.rules`, `99-hw-management-bmc-mctp.rules`)  
can react to link/device events.   
sources live under the `bmc/` tree in https://github.com/Mellanox/hw-mgmt/tree/master/bmc.

### 7. Testing Requirements/Design  

Validate **Switch-Host restart** (BMC stays up): Ping `bmc_addr` from the host. Ping `bmc_if_addr` from the BMC.  
Validate **Switch-BMC restart** (host stays up): Ping `bmc_addr` from the host. Ping `bmc_if_addr` from the BMC.
