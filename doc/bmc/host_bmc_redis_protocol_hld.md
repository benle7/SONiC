# Host to BMC communication over Redis

## Table of Content

- [1. Revision](#1-revision)
- [2. Scope](#2-scope)
- [3. Definitions/Abbreviations](#3-definitionsabbreviations)
- [4. Overview](#4-overview)
- [5. Requirements](#5-requirements)
- [6. Architecture Design](#6-architecture-design)
- [7. High-Level Design](#7-high-level-design)
- [8. SAI API](#8-sai-api)
- [9. Configuration and management](#9-configuration-and-management)
- [10. Warmboot and Fastboot Design Impact](#10-warmboot-and-fastboot-design-impact)
- [11. Memory Consumption](#11-memory-consumption)
- [12. Restrictions/Limitations](#12-restrictionslimitations)
- [13. Testing Requirements/Design](#13-testing-requirementsdesign)
- [14. Open/Action items](#14-openaction-items)

### 1. Revision

| Rev | Date | Author | Change Description |
|---|---|---|---|
| 0.1 | 2026-08-10 | NVIDIA | Initial version |

### 2. Scope

This document covers adding Redis as a second transport for host-to-BMC
inventory reads, alongside the existing Redfish transport, on platforms whose BMC
runs SONiC. It specifies how the transport is selected, how `BMCBase.get_eeprom()`
is served from the BMC's `STATE_DB`, how the BMC's TLV-keyed `EEPROM_INFO` schema
is translated into the dictionary shape callers already consume, and how the
operations that have no Redis equivalent behave once the Redis transport is
selected.

It does not cover the BMC side. The BMC's own `EEPROM_INFO` population is
existing behaviour, implemented for the NVIDIA SONiC BMC platform in
`platform/aspeed/sonic-platform-modules-nvidia-bmc/ast2700/sonic_platform/eeprom.py:36-42`
and written by `syseepromd` through
`src/sonic-platform-common/sonic_platform_base/sonic_eeprom/eeprom_tlvinfo.py:746-776`.
No change is proposed to either.

Three further areas are deliberately out of scope. Firmware update, BMC reset,
root-password reset, session management and debug-log collection stay on Redfish;
this design specifies their behaviour under the Redis transport but does not
provide Redis equivalents. Thermal data is unaffected: it already crosses this
link, but in the opposite direction — `thermalctld` on the switch host mirrors
`TEMPERATURE_INFO` *into* the BMC's `STATE_DB`
(`src/sonic-platform-daemons/sonic-thermalctld/scripts/thermalctld:961`, `:988`),
and the BMC then reads that table from its own local database
(`thermalctld:1309-1321`). And Redfish is not being removed or deprecated — it
remains the transport for every platform whose BMC runs OpenBMC.

### 3. Definitions/Abbreviations

| Term | Definition |
|---|---|
| BMC | Board Management Controller — a service processor on the switch board, managed out-of-band from the switch host |
| OpenBMC | Linux distribution for BMC hardware that exposes a full Redfish service |
| SONiC BMC | A BMC running SONiC rather than OpenBMC, which publishes inventory into its own Redis databases |
| Switch host | The main switch CPU complex, as distinct from the BMC; the side this design's code runs on |
| Redfish | DMTF RESTful hardware-management API, used today as the sole host-to-BMC transport |
| FRU | Field Replaceable Unit — the IPMI inventory record format the BMC reads from its EEPROM |
| TLV | Type-Length-Value — the ONIE EEPROM encoding whose type codes key the BMC's `EEPROM_INFO` table |

### 4. Overview

Every host-to-BMC operation in SONiC today goes through Redfish. The switch host
builds a `RedfishClient`
(`src/sonic-platform-common/sonic_platform_base/redfish_client.py`) which shells
out to `curl` for each request, and `BMCBase` creates one unconditionally in its
constructor (`src/sonic-platform-common/sonic_platform_base/bmc_base.py:72`).
Reading the BMC's EEPROM therefore costs a Redfish login, a GET against the
chassis inventory resource, and a logout — three HTTPS requests, each a separate
`curl` process.

That design assumes the BMC runs a full Redfish service, which holds for OpenBMC.
On a BMC running SONiC it is a poor fit: the BMC already publishes its inventory
into its own `STATE_DB` as any SONiC platform daemon would, and a TCP path
between the two sides already exists and is already used. Requiring a full
Redfish stack on the BMC purely so the host can read three EEPROM fields it could
read directly is avoidable work on both sides.

This design adds Redis as an alternative transport for that read. Which transport
is used becomes a configured property of the platform, held in `CONFIG_DB`, so a
single image supports both kinds of BMC. When the Redis transport is selected,
`get_eeprom()` connects to the BMC's `STATE_DB`, reads the `EEPROM_INFO` table,
and translates its TLV-code-keyed entries into the same dictionary keys the
Redfish path already returns. Callers above `get_eeprom()` — including
`get_model()`, `get_serial()` and both `show platform bmc` subcommands — are
unchanged and unaware of which transport served them.

The operations with no Redis equivalent are not silently degraded. The existing
session-management decorator becomes the single point that recognises "this
operation requires Redfish" and raises, so a caller on a SONiC BMC gets an
explicit failure rather than a confusing empty result.

### 5. Requirements

Each requirement below is a behaviour that can be verified, and each maps to at
least one case in section 13.

1. The host-to-BMC transport is selectable through `CONFIG_DB`, so one image
   serves platforms whose BMC runs OpenBMC and platforms whose BMC runs SONiC.
2. When the Redis transport is selected, `get_eeprom()` returns the BMC EEPROM
   contents read from the BMC's `STATE_DB`.
3. `get_eeprom()` returns the same dictionary key names under both transports, so
   that `get_model()` and `get_serial()` require no transport-specific code.
4. `get_model()` and `get_serial()` return the BMC's model and serial number
   under both transports.
5. The operations that require Redfish — session open and close, root-password
   reset, firmware update, BMC reset, debug-log dump trigger and retrieval, and
   the Redfish-readiness wait — raise `NotImplementedError` when the Redis
   transport is selected, and that exception reaches the caller rather than being
   converted into an error return value.
6. `get_version()` returns `'N/A'` when the Redis transport is selected, in
   keeping with its existing documented contract, so that callers which read it
   alongside EEPROM fields continue to function.
7. The connection to the BMC's Redis is constructed with a finite timeout, so a
   read against an unreachable BMC fails in bounded time rather than blocking.
8. Under the Redis transport, a read that fails, or that cannot supply both
   `Model` and `SerialNumber`, returns an empty dictionary — consistent with
   `get_eeprom()`'s existing empty-on-failure contract. The completeness rule
   itself is new and applies to the Redis accessor only; the Redfish path's
   behaviour is unchanged.
9. The `DeviceBase` APIs — `get_name()`, `get_presence()`, `get_revision()`,
   `get_status()` and `is_replaceable()` — behave identically under both
   transports.
10. `show platform bmc eeprom` and `show platform bmc summary` produce useful
    output under both transports, marking unavailable fields `N/A` rather than
    failing.
11. The transport is configurable from the CLI, modelled in YANG, and the
    configured value survives a warm reboot and a fast reboot.

#### 5.1. Exemptions

The following are deliberately not supported. They are design decisions rather
than oversights, and they are not testable behaviours, so section 13 does not
cover them.

- **No DB migrator step.** An upgraded device with no configured value resolves
  to the SONiC BMC transport. This is a behaviour change for existing OpenBMC
  platforms, detailed in section 7.5.
- **The Redis transport is unauthenticated**, consistent with the existing
  host-to-BMC Redis usage in `thermalctld`.
- **`Manufacturer` and `PowerState` are not available** under the Redis
  transport, because the BMC's FRU parsing does not produce them. See section 7.5.
- **No Redfish operation gains a Redis equivalent** in this design.

### 6. Architecture Design

The SONiC architecture does not change. No new daemon, service, container or
inter-process interface is introduced, and no existing one changes shape.

The change is confined to the inside of the existing platform-API BMC layer. The
host-side call path today is `Platform().get_chassis().get_bmc()`, returning a
vendor `BMC` subclass of `BMCBase` which owns a `RedfishClient` and reaches the
BMC over HTTPS. This design adds a second, alternative read path inside
`BMCBase` — a direct Redis connection to the BMC's `STATE_DB` — and a branch that
selects between them. The class hierarchy, the vendor subclass contract and every
caller above `get_bmc()` stay as they are.

The transport mechanism is not new to SONiC, though its direction here is. The
switch host already opens the BMC's `STATE_DB` over TCP through
`daemon_base.db_connect_remote()`, using the BMC address from `bmc.json`:
`thermalctld` does so to mirror `TEMPERATURE_INFO` into the BMC
(`thermalctld:961`, `:988`). So the TCP path, the address source and the absence
of authentication are all established practice. What this design adds is the
first host-side *read* of a BMC-owned table, which is a new data-flow direction
over an existing link rather than a new link or a new trust boundary.

No diagram is committed with this revision; the call path is a single branch
inside one class and is given concretely in section 7.6.

### 7. High-Level Design

#### 7.1. Feature type

A built-in SONiC feature. It is not an Application Extension.

#### 7.2. Modules and repositories changed

Three repositories are affected. Because some SONiC code lives in submodules that
are separate repositories while other code is in-tree, this list — not the count
of directories touched — determines the set of pull requests the implementation
produces.

| Repository | Path | Change |
|---|---|---|
| `sonic-buildimage` | `src/sonic-yang-models` | New YANG leaf for the transport selector |
| | `src/sonic-py-common/sonic_py_common/device_info.py` | Accessor returning the configured BMC OS |
| | `platform/mellanox/mlnx-platform-api/sonic_platform/bmc.py` | Make the Redfish credential requirement conditional on transport |
| `sonic-platform-common` | `sonic_platform_base/bmc_base.py` | Transport branch, Redis read, TLV translation, Redfish-required gate |
| `sonic-utilities` | `config/bmc.py`, `doc/Command-Reference.md` | CLI to set the transport, and its documentation |

`src/sonic-py-common` and `platform/mellanox/mlnx-platform-api` are in-tree, not
submodules, so their changes land in the `sonic-buildimage` pull request, while
`sonic-platform-common` and `sonic-utilities` are submodules and get their own.

**The change has an ordering dependency across that boundary.** `bmc_base.py`
lives in the `sonic-platform-common` submodule but calls the new accessor in
`sonic_py_common`, and the submodule is built against whatever `sonic_py_common`
is installed rather than necessarily the copy in the same tree. The
`sonic-buildimage` change must therefore be in the image before the
`sonic-platform-common` change is functional. Section 7.3 specifies how
`BMCBase` behaves when it is not, because that case must not break the OpenBMC
path this design otherwise leaves untouched.

The Mellanox platform module needs a change, contrary to what the
transport-neutral callbacks suggest: `BMC.get_instance()` returns `None` unless a
Redfish NOS account is found, and `chassis.get_bmc()` returning `None` makes both
consumers print "BMC is not available on this platform"
(`src/sonic-utilities/show/platform.py:94-96`, `:143-145`). Without it, a platform
would have to ship Redfish credentials it never uses in order to read EEPROM over
Redis. Three sites in
`platform/mellanox/mlnx-platform-api/sonic_platform/bmc.py` gate on those
credentials and all three must become conditional on the transport, because a
SONiC-BMC platform is likely to ship no `bmc_config.json` at all:

- the early return when the build configuration is absent entirely (`:67-70`),
- the early return when the file exists but carries no NOS account name
  (`:71-74`) — both of which discard the `bmc_addr` already resolved above them,
- and the `get_instance()` gate itself, which tests both the address and the
  account name (`:99`).

Under the SONiC transport, the two early returns must yield `bmc_addr` with the
credential fields unset, and the gate must test the address alone. Nothing else in
the class needs changing: the constructor performs no I/O, and the password
callback and the vendor Redfish import are reached only from `RedfishClient.login()`,
which the transport gate prevents.

#### 7.3. Interfaces and dependencies

**One accessor, one resolution point.** The configured BMC OS is read through a
single new `sonic_py_common.device_info` accessor — placing it there rather than in
`BMCBase` keeps `CONFIG_DB` access out of the platform API and matches where
`get_bmc_data()` and `get_bmc_address()` already live
(`src/sonic-py-common/sonic_py_common/device_info.py:1137-1170`). Two callers
consult it: the vendor credential gate in `get_instance()` (section 7.2) and
`BMCBase` itself. The vendor gate necessarily runs first, before any `BMCBase`
instance exists, so the transport cannot be something the base constructor
discovers on its own.

The accessor therefore memoises its result for the life of the process. That gives
**one `CONFIG_DB` read per process** regardless of how many times it is consulted
or from where, and one answer that cannot change mid-command — which matters
because `get_model()` and `get_serial()` each call `get_eeprom()`
(`bmc_base.py:294`, `:306`).

Per-process is the right granularity, and holding the value that long is safe:
no long-lived caller exists. No daemon constructs a BMC object at all — nothing
under `src/sonic-platform-daemons` calls `get_bmc()` — so `pmon` cannot hold one
across a configuration change, and every constructor in the tree belongs to a
short-lived CLI process. On Mellanox the vendor class is a singleton and the
chassis guards re-entry, so within one process the transport is fixed at first
construction in any case. A transport change therefore takes effect on the next
command, with no restart required.

**Behaviour when the accessor is unavailable.** Because of the ordering dependency
in section 7.2, `BMCBase` may run against a `sonic_py_common` that predates the
accessor, in which case calling it raises `AttributeError`. That must not break
BMC object construction, since doing so would take out `show platform bmc` and the
firmware tooling on OpenBMC platforms. `BMCBase` therefore treats an unavailable
accessor the same way it treats any other inability to determine the transport,
per the rule in section 7.15: it resolves to `openbmc`, preserving today's
behaviour exactly, and logs.

The Redis connection is constructed directly rather than through
`daemon_base.db_connect_remote()`, for the reason given in section 7.6. This
leaves `sonic_py_common` with a single new symbol and keeps the timeout decision
inside this design.

Nothing depends on `BMCBase` differently as a result of any of this. The public
method signatures and return contracts are unchanged; only the behaviour of a
subset of them under one configuration is new. The consumers are the two `show
platform bmc` subcommands (`show/platform.py:98`, `:148`), the BMC firmware and
techsupport tooling, and the vendor `ComponentBMC` object — none of which need
modification.

#### 7.4. SWSS and Syncd changes

N/A. Neither `swss` nor `syncd` is involved in host-to-BMC communication.

#### 7.5. DB and schema changes

**CONFIG_DB — one new field.** The transport selector is a field on the
`DEVICE_METADATA|bmc` entry:

| Table | Key | Field | Type | Values |
|---|---|---|---|---|
| `DEVICE_METADATA` | `bmc` | `os` | string | `openbmc`, `sonic` |

The YANG container for this entry already exists
(`src/sonic-yang-models/yang-models/sonic-device_metadata.yang:400-425`), but no
code in the tree populates the row: `bmc.json` is passed to `sonic-cfggen` as
template variables for `interfaces.j2`
(`files/image_config/interfaces/interfaces-config.sh:87-89`), not written to
`CONFIG_DB`, and every consumer reads the file directly through
`device_info.get_bmc_data()`. The row is therefore created by `config bmc os` on
first use, which is why the upgrade case below is an absent *key*, not merely an
absent field.

**STATE_DB on the BMC — consumed, not defined.** The design reads the BMC's
`EEPROM_INFO` table, which is existing schema written by `syseepromd` and
enumerated in `eeprom_tlvinfo.py:746-776`. It is keyed by TLV code, not by field
name, with the key rendered as `hex(code)` (`eeprom_tlvinfo.py:757`):

| Key | Field read | Meaning |
|---|---|---|
| `EEPROM_INFO\|State` | `Initialized` | `1` once the table has been written |
| `EEPROM_INFO\|Checksum` | `Valid` | `1` if the decoded EEPROM checksum verified |
| `EEPROM_INFO\|0x21` | `Value` | Product name |
| `EEPROM_INFO\|0x22` | `Value` | Part number |
| `EEPROM_INFO\|0x23` | `Value` | Serial number |

Nothing is written to the BMC's databases. The host's own `STATE_DB` is not
involved.

**Translation.** The Redfish path returns every non-`@odata` property of the
chassis inventory resource, plus `State`, `Health` and `HealthRollup` derived from
its `Status` (`redfish_client.py:1474-1487`) — so the key set is whatever the BMC's
Redfish implementation exposes, not a fixed list. To satisfy requirement 3, the
Redis path returns the same key *names* for the values it can supply, through a
single mapping constant:

| Source | Key returned |
|---|---|
| `EEPROM_INFO\|0x21`:`Value` | `Model` |
| `EEPROM_INFO\|0x22`:`Value` | `PartNumber` |
| `EEPROM_INFO\|0x23`:`Value` | `SerialNumber` |
| `EEPROM_INFO\|State`:`Initialized` | `State`, `Enabled` when `1` |
| `EEPROM_INFO\|Checksum`:`Valid` | `Health`, `OK` when `1` |

Product name maps to `Model` and part number to `PartNumber` because Redfish
carries both as distinct keys and the two TLVs are distinct values.

The Redis dictionary is a subset: `Manufacturer`, `PowerState`, `HealthRollup`,
`ChassisType`, `Id`, `Location` and `Name` are not produced. The two that matter
are `Manufacturer` and `PowerState`, because they are the only absent keys any
consumer reads. The BMC's FRU parsing extracts five TLVs — product name, part
number, serial number, base MAC and label revision
(`ast2700/sonic_platform/eeprom.py:36-42`) — with no manufacturer among them, and
power state is a Redfish resource property with no EEPROM equivalent at all. Both
are therefore absent rather than invented, and both consumers read them with
`.get(key, 'N/A')` (`show/platform.py:104-108`, `:155-159`), so they display as
`N/A`. The base MAC (`0x24`) and label revision (`0x27`) are readable but have no
consumer in `BMCBase` — `get_revision()` returns `'N/A'` on both transports — so
they are not mapped until one exists.

A dictionary carrying this subset satisfies
`BMCBase._is_bmc_eeprom_content_valid()`, which checks only that the dictionary is
non-empty and carries no `error` key (`bmc_base.py:249-264`).

**Completeness.** `Initialized` is set to `1` in `visit_end()`
(`eeprom_tlvinfo.py:775-776`) even when the TLV walk aborted part-way, because
`visit_eeprom()` breaks out of the loop on an invalid field and still calls
`visit_end()` (`eeprom_tlvinfo.py:658-670`). A read can therefore find
`Initialized = 1` with `0x21` or `0x23` missing. Since `get_model()` and
`get_serial()` would then return `None` from a dictionary that passed validation,
a read that cannot supply both `Model` and `SerialNumber` is treated as failed and
returns `{}` — requirement 8.

**Upgrade story.** No DB migrator step is added, so a device that upgrades with
existing configuration has no `DEVICE_METADATA|bmc` row at all, and an absent
value resolves to `sonic`. For a platform whose BMC runs OpenBMC this is a
behaviour change: after upgrade, EEPROM reads attempt the Redis path and the
Redfish-only operations raise until `os` is set to `openbmc` explicitly. This is a
deliberate exemption (section 5.1), recorded here so that release notes can carry
it. Adding a migrator step, or defaulting from platform data instead, would remove
the need for manual configuration on upgrade; that trade-off is in section 14.

#### 7.6. Sequence diagram

The Redis read, in order. Steps 3 to 6 replace a Redfish login, GET and logout.

1. A caller invokes `get_eeprom()` on the BMC object obtained from the chassis.
   The transport was resolved when that object was constructed (section 7.3).
2. For `openbmc`, `get_eeprom()` calls the existing decorated Redfish accessor
   and the rest of this sequence does not apply.
3. For `sonic` it calls the new undecorated Redis accessor, which resolves the
   BMC address from `bmc.json` via `device_info.get_bmc_address()`
   (`device_info.py:1157-1170`). If there is no address, it returns `{}`.
4. The accessor connects to `STATE_DB` — database index `6` — on that address at
   the Redis port `6379`, with a finite timeout.
5. It reads `EEPROM_INFO|State`:`Initialized`. If it is not `1`, it returns `{}`.
6. It reads the mapped TLV keys and the checksum validity and translates them per
   section 7.5. A failed checksum is reported as `Health` other than `OK` and does
   **not** suppress the record, matching how the Redfish path surfaces `Status`. A
   missing `Model` or `SerialNumber` returns `{}`, per section 7.5. Both checks
   live in this accessor, not in `get_eeprom()`, so that the Redfish path's
   validation is untouched.
7. `get_eeprom()` validates the result as it does today and returns it. For
   `get_model()` and `get_serial()`, the caller then selects `Model` or
   `SerialNumber` from it, unchanged.

**Why the connection is constructed directly.** The obvious helper,
`db_connect_remote(db_id, host, port)`, accepts no timeout: it passes the module
constant `REDIS_TIMEOUT_MSECS`, which is `0` — an unbounded wait
(`src/sonic-py-common/sonic_py_common/daemon_base.py:15`, `:32-46`). Since
`get_eeprom()` is reachable from the CLI, inheriting that default would let an
unreachable BMC hang `show platform bmc` indefinitely. Rather than add a timeout
argument to a shared helper whose only other caller is `thermalctld`, the Redis
accessor constructs the connector itself with the database index, address, port
and a finite timeout. That keeps the change inside `sonic-platform-common`,
leaves the existing caller untouched by construction rather than by a default
value, and avoids a second cross-repository version dependency of the kind
section 7.2 describes. The concrete timeout value is an open item (section 14).

For a Redfish-only operation under `sonic`, the sequence is shorter, and *where*
the check sits is a correctness constraint rather than a detail. In
`with_session_management` the `try` is the wrapper's first statement
(`bmc_base.py:31-32`) and its `except Exception` converts anything raised inside
into `(ERR_CODE_GENERIC_ERROR, str(e))` (`bmc_base.py:44-48`). The transport check
must therefore run **before** that `try`, or requirement 5 silently does not
hold — the caller would receive an error tuple, which is exactly the obscure
failure the requirement exists to prevent.

Two of the named operations are not reached by the decorator at all:
`open_session` (`bmc_base.py:173-185`) and `wait_until_redfish_ready`
(`bmc_base.py:161-171`) are undecorated, the latter deliberately and with a
comment explaining why (`bmc_base.py:168-170`). Each carries its own explicit
transport guard. They are reachable from `config bmc open-session`
(`src/sonic-utilities/config/bmc.py:42`) and from the BMC firmware update tooling,
so leaving them ungated would surface as a connection error instead of a clear
refusal.

#### 7.7. Linux dependencies and interfaces

A TCP connection from the switch host to the BMC's Redis port on the internal
host-to-BMC link. The interface name, the host address and the BMC address all
come from `/etc/sonic/bmc.json` — fields `bmc_if_name`, `bmc_if_addr` and
`bmc_addr` — read through `device_info.get_bmc_data()`
(`device_info.py:1137-1154`). The interface name is platform data, not a fixed
value, and this design does not assume a particular one. The link and its
addressing are existing behaviour, described in the BMC support document alongside
this one.

No new package, kernel module or system service is required.

#### 7.8. Warm reboot requirements and dependencies

The EEPROM read itself has none. It is an on-demand read that holds no state
between calls, and the connection is per-call rather than long-lived, so there is
nothing to preserve or restore. The BMC is not restarted by a host warm reboot, so
its `EEPROM_INFO` table remains populated and the path is available as soon as a
caller uses it.

The transport selector is different, and is covered in section 10: it is
persisted configuration, and its absence is not neutral.

#### 7.9. Fastboot requirements and dependencies

The same as section 7.8, for the same reasons.

#### 7.10. Scalability and performance

The Redis path is cheaper than the path it replaces, and neither is on a scaling
path. A Redfish EEPROM read costs three HTTPS requests — login, GET, logout — each
executed as a separate `curl` process spawned through `subprocess.Popen`. The
Redis read is one connection and a small number of hash reads on an already
running server, with no process creation.

Every caller of `get_eeprom()` is on demand: `show platform bmc summary`
(`show/platform.py:98`), `show platform bmc eeprom` (`show/platform.py:148`), and
`get_model()`/`get_serial()` (`bmc_base.py:294`, `:306`), which have no in-tree
caller. Nothing polls it. Nothing about the change is per-port, per-interface or
per-neighbour, so there is no scale dimension to quantify.

The OpenBMC path acquires one additional cost: a single `CONFIG_DB` read to
resolve the transport. Because the accessor memoises, that is once per process
rather than per object or per call (section 7.3), and it is negligible beside the
three `curl` invocations that follow it.

#### 7.11. Memory requirements

Negligible and bounded. No daemon is added and no cache is introduced. The Redis
connection is created per call and released with it, and the returned dictionary
holds five keys. The mapping constant is a module-level dictionary of five
entries.

#### 7.12. Docker dependency

None. The code runs in the same contexts as today — `pmon` for the component
object, the host for the CLI — and no container gains or loses a dependency.

#### 7.13. Build dependency

None. No new package, build-time tool or build-system change. The YANG leaf is
added to an existing model that is already built.

#### 7.14. Management interfaces

CLI only. A new `config` command sets the transport, and the existing `show
platform bmc` subcommands are unchanged in syntax and in output format. Details
are in section 9.2.

SNMP, gNMI and the REST API are not affected. No BMC inventory is exposed through
them today, and this design does not add any.

#### 7.15. Serviceability and debug

The failure modes are distinguishable from the logs. Each point at which the read
is abandoned logs at error level through the existing `bmc_base` logger, naming
which one occurred: no BMC address available, connection failure or timeout, table
not initialised, and `Model` or `SerialNumber` missing from an initialised table.
A failed checksum is not an abandonment point — it is reported through `Health`,
per section 7.6 — but is logged. This matters because all of these currently
surface to the CLI user as the same "Failed to retrieve BMC EEPROM information"
line (`show/platform.py:150-152`), which the existing Redfish path produces too.

The transport lookup is itself a failure mode, and a misleading one if left
silent. If a `CONFIG_DB` read that fails outright — database container down, or very
early boot — were treated the same as a read that found no value, it would route an
OpenBMC platform onto the Redis path and make every Redfish-only operation raise,
presenting a configuration problem as a BMC fault.

The accessor therefore distinguishes **not knowing** from **knowing there is no
value**, and resolves them differently:

| Situation | Resolves to |
|---|---|
| A successful read finds no `os` value | `sonic` — the new default (section 7.5) |
| The read fails, or the accessor is unavailable, or the value is not one of the two accepted strings | `openbmc` — today's behaviour, preserved |

Only an explicit, readable configuration selects the Redis transport by accident
of absence; anything the code cannot interpret falls back to the transport that
works everywhere today. Each of the fallback cases logs distinctly from the
others, so a transport that was inferred rather than configured is visible in the
log. The unrecognised-value case matters because the CLI rejects invalid input but
a direct `CONFIG_DB` write does not.

An engineer diagnosing the Redis path in the field has three checks, in order:
that `/etc/sonic/bmc.json` carries a `bmc_addr`; that the address answers a ping,
which is what `get_status()` already does (`bmc_base.py:320-334`); and that the
BMC's `EEPROM_INFO` table is populated, which can be read directly with a Redis
client against that address and database index `6`. The `NotImplementedError`
raised by a Redfish-only operation names the operation, so a failed BMC firmware
update on a SONiC BMC is self-explanatory rather than presenting as a connectivity
problem.

#### 7.16. Platform applicability

Applicable to any platform whose BMC runs SONiC. It is not restricted to a
vendor: the behaviour is gated entirely on the configured `os` field, with no SKU
or platform-string comparison anywhere in the change, so a platform opts in
through its device data.

A platform whose BMC runs OpenBMC needs nothing, other than the explicit
configuration on upgrade described in section 7.5. A platform adopting the SONiC
BMC transport needs three things: its BMC must populate `EEPROM_INFO` in its own
`STATE_DB`, which is standard SONiC platform-daemon behaviour rather than anything
specific to this design; the host-to-BMC link and `bmc.json` already required
today; and a vendor BMC class that constructs without Redfish credentials, which
for Mellanox is the change listed in section 7.2.

Because the field is added to a shared YANG model and a shared platform API, the
community should be told in advance of the schema addition.

### 8. SAI API

No SAI API changes, and no new SAI APIs. This design does not touch the ASIC data
path or any SAI object; host-to-BMC communication is entirely outside SAI's
scope.

### 9. Configuration and management

#### 9.1. Manifest

N/A. The feature is not an Application Extension, so no manifest applies.

#### 9.2. CLI/YANG model enhancements

One new command, under the existing `config` group that already holds BMC
configuration (`src/sonic-utilities/config/bmc.py`):

```
config bmc os <openbmc|sonic>
```

It writes the `os` field described in section 7.5, creating the
`DEVICE_METADATA|bmc` row if absent, and rejects any other value. Like other
`config` commands it writes the running `CONFIG_DB`; `config save` is required to
persist it across a cold boot, which section 10 covers. There are no CLI deletions
and no modifications to existing commands. The `show platform bmc` subcommands are
unchanged.

The corresponding YANG leaf is added to the `DEVICE_METADATA` `bmc` container
(`sonic-device_metadata.yang:400-425`) as an enumeration of the two accepted
values. `sonic-utilities`' `doc/Command-Reference.md` gains the new command in the
platform section.

On downward compatibility: a configuration saved by a previous release restores
without error, because the leaf is optional and its absence is valid. What that
absence *means* changes, which is the behaviour change covered in section 7.5
rather than a restore failure. Restoring a configuration saved by *this* release
onto a previous release is the weaker direction: the earlier YANG model has no
`os` leaf, so a restore path that runs YANG validation would reject the field
rather than ignore it. Which restore paths validate has not been established here,
and confirming it is an open item (section 14).

KLISH is not affected; the BMC commands are CLICK only.

#### 9.3. Config DB enhancements

One field added, none deleted or modified: `DEVICE_METADATA|bmc` gains `os`, as
specified in section 7.5, and the row itself is created on first use. Downward
compatibility is addressed in section 9.2 and the upgrade behaviour in section
7.5.

### 10. Warmboot and Fastboot Design Impact

The read path has no impact and no requirements, for the reasons in sections 7.8
and 7.9. The transport selector does have a boot dependency, and it is the one
part of this design that must survive a reboot.

The selector is persisted configuration whose absence is not neutral: an absent
value resolves to `sonic` (section 7.5). So an OpenBMC platform that is configured
with `config bmc os openbmc` but never `config save`d reverts to the Redis
transport on the next cold boot, and every Redfish-only operation then raises.
This is standard `CONFIG_DB` behaviour rather than anything new, but it has a
consequence specific to this design and is called out for that reason. Warm and
fast reboot preserve `CONFIG_DB` and so preserve the selector; requirement 11
states that, and section 13.2 verifies it directly rather than inferring it from
the EEPROM read continuing to work.

Existing warm and fast boot behaviour is otherwise unaffected. The change adds no
service to the boot sequence and no ordering constraint.

#### 10.1. Warmboot and Fastboot Performance Impact

No control-plane or data-plane downtime is added.

The change adds no stall, sleep or IO to the boot critical chain, because nothing
in it executes during boot — both transports are invoked only when a caller asks
for BMC inventory, and no caller does so from the boot path. This holds whether
the feature is configured or not.

It adds no CPU-heavy processing to the boot path, and no Jinja rendering. The YANG
leaf is parsed as part of an existing model with no measurable cost.

No third-party dependency is added or updated. No service or container needs
delaying. Since no boot-time cost is introduced, there is no cost to optimise and
no degradation to expect.

### 11. Memory Consumption

There is no memory consumption when the feature is unused, and none that grows
when it is disabled by configuration.

Under `openbmc`, the code paths added by this design are never entered; the only
residue is the module-level mapping constant of five entries, which exists whether
or not it is used and is not a per-instance cost.

Under `sonic`, consumption is bounded by construction rather than by discipline:
the Redis connection is created inside the accessor and released when it returns,
so no connection accumulates across calls, and the returned dictionary is fixed at
five keys regardless of how many times it is read. No cache, no history and no
queue is introduced, so there is nothing whose size grows with uptime or with call
count.

### 12. Restrictions/Limitations

- `Manufacturer` and `PowerState` are unavailable under the Redis transport and
  display as `N/A`, per section 7.5.
- `get_version()` returns `'N/A'` under the Redis transport, so BMC firmware
  version is not reported on a SONiC BMC.
- BMC firmware update, BMC reset, root-password reset, session management and
  debug-log collection are unavailable under the Redis transport and raise
  `NotImplementedError`.
- The Redis transport is unauthenticated.
- An OpenBMC platform that upgrades without configuring `os` explicitly changes
  behaviour, per section 7.5.
- The design assumes a single BMC at a single address, as the existing `bmc.json`
  schema and the Redfish path do.

### 13. Testing Requirements/Design

Every requirement in section 5 is proven by at least one case below; the
exemptions in section 5.1 are not behaviours and are not covered. This section
states what must be proven. The test strategy stage expands it into the detailed
plan, including how each assertion is written and which of the existing Redfish
mock layers each case reuses.

#### 13.1. Unit Test cases

These run in CI with no BMC attached, so the BMC's Redis is mocked, as the Redfish
transport is mocked today.

The **existing** `sonic-platform-common` BMC suite is affected and must be updated
as part of this change. Its cases construct a `BMCBase` subclass directly with no
`CONFIG_DB` present, so under the rules in section 7.15 they would resolve to
`openbmc` and keep testing the Redfish path — which is the intent, but it must be
made explicit rather than relied on, and pinned, because a test environment that
substitutes a mock for `sonic_py_common` would otherwise yield a transport value
that is neither accepted string. Since unit tests are not run locally in this
flow, an unpinned suite surfaces as a CI failure on the pull request rather than
before it.

- Transport selection routes an EEPROM read to the Redis accessor under `sonic`
  and to the Redfish accessor under `openbmc`; a successful read with no value
  selects `sonic`; and the accessor is consulted once per process rather than once
  per call (requirements 1, 3). The per-process case must reset the vendor
  singleton, or it passes vacuously against a cached object.
- Each fallback resolves to `openbmc` and logs distinctly: a `CONFIG_DB` read that
  raises, an accessor that is unavailable, and a value that is neither accepted
  string (requirement 1).
- A successful Redis read returns the mapped dictionary, with each TLV landing on
  its intended key and `State` derived correctly (requirements 2, 3).
- `get_model()` and `get_serial()` return the right values from a Redis-sourced
  dictionary, with no transport-specific code exercised (requirement 4).
- Each decorated Redfish-only operation raises `NotImplementedError` under
  `sonic`, and the exception escapes the wrapper rather than being converted into
  an error tuple by `bmc_base.py:44-48` (requirement 5).
- `open_session` and `wait_until_redfish_ready` also raise under `sonic`. These
  need their own cases: they are undecorated, so a decorator-level test passes
  while they remain ungated (requirement 5).
- `get_version()` returns `'N/A'` under `sonic` and does not raise
  (requirement 6).
- The Redis connection is constructed with the database index, address, port and a
  finite timeout, asserted on the constructor arguments rather than on wall-clock
  behaviour, which a mock cannot demonstrate (requirement 7).
- Each failure mode returns an empty dictionary: no address available, connection
  failure, `Initialized` not set, and an initialised table missing `Model` or
  `SerialNumber` (requirement 8).
- An invalid checksum returns the values with `Health` other than `OK`, rather
  than an empty dictionary (requirements 2, 8).
- The `DeviceBase` APIs return identical results under both transports
  (requirement 9).
- Both `show platform bmc` subcommands render under `sonic`, showing the
  available fields and `N/A` for `Manufacturer`, `PowerState` and firmware
  version, and neither aborts (requirement 10).
- The CLI accepts both valid transport values, rejects an invalid one, creates the
  row when absent, and the YANG model validates accordingly (requirement 11).

#### 13.2. System Test cases

On a platform whose BMC runs SONiC, with a real BMC attached:

- EEPROM reads return the BMC's true model, part number and serial number under
  the Redis transport, matching what the BMC reports locally (requirement 2).
- Switching the configured transport changes which path is used, without a reboot.
  Because the accessor memoises per process (section 7.3), the two halves must be
  run as separate command invocations (requirement 1).
- A BMC that is unreachable, and a BMC whose `EEPROM_INFO` is not yet
  initialised, both produce the documented failure in bounded time rather than
  hanging the CLI (requirements 7, 8).
- The configured transport, once saved, is still in effect after a warm reboot and
  after a fast reboot — asserted on the selector itself, not merely on the EEPROM
  read succeeding (requirement 11).
- On a platform whose BMC runs OpenBMC, with the transport configured explicitly,
  all existing BMC operations including firmware update continue to work — a
  regression check on the Redfish path.

### 14. Open/Action items

| Item | Owner |
|---|---|
| Fill in the author name in section 1 before review | Feature owner |
| Decide the concrete timeout value used for the Redis connection (section 7.6) | Feature owner |
| Confirm which configuration restore paths run YANG validation, and therefore whether a configuration saved by this release can be restored on a previous one (section 9.2) | Feature owner |
| Confirm whether the BMC's FRU parsing should be extended to expose a manufacturer field, which would let `Manufacturer` be served over Redis (section 7.5) | BMC platform owner |
| Confirm whether a DB migrator step or a platform-data default is preferred over requiring explicit configuration on upgrade (section 7.5) | Feature owner and release owner |
| Decide whether the base MAC and label revision TLVs should be exposed once a consumer exists (section 7.5) | Feature owner |
| Confirm whether an authentication story is required for the host-to-BMC Redis path, given the existing unauthenticated usage (section 5.1) | Security reviewer |
