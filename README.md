# BPI-R4 / OpenWrt 25.12.5 XGSPONST2001 SFP Fix

This repository provides a patched Linux SFP kernel module for
OpenWrt 25.12.5 on the Banana Pi BPI-R4.

# Project Overview
 
This project was created while preparing a Banana Pi BPI-R4
for AT&T Fiber service using an OEM XGSPONST2001 XGS-PON
SFP+ module.
 
The stock OpenWrt 25.12.5 kernel repeatedly disabled the
SFP interface due to TX_FAULT events. Investigation showed
that the required Linux driver quirks existed upstream but
had not yet been incorporated into the OpenWrt release.
 
To resolve the issue, I backported the upstream Linux
patches, rebuilt the sfp.ko kernel module, and validated
stable 10Gbps operation.

## Problem

The OEM XGSPONST2001 XGS-PON SFP module causes the stock OpenWrt
6.12.94 SFP driver to repeatedly report:

    sfp sfp1: module transmit fault indicated
    sfp sfp1: module transmit fault recovered
    sfp sfp1: module transmit fault indicated
    ...
    sfp sfp1: module persistently indicates fault, disabling

The result is that `sfp-wan` is disabled by the SFP state machine.

## Root Cause
 
The XGSPONST2001 reports TX_FAULT and LOS conditions
that do not accurately represent link health.
 
The upstream Linux quirk associates this module with
sfp_fixup_potron() which suppresses the incorrect fault
behavior and applies extended startup handling.
 
Without this quirk, the OpenWrt SFP state machine
eventually disables the interface.
 
With the quirk applied, the module initializes normally.

## Investigation Timeline

**Observed:**
- Interface repeatedly entered TX_FAULT state

**Hypothesis:**
- Faulty SFP module
- Hardware compatibility issue
- Missing driver support

**Testing:**
- Verified EEPROM values
- Compared logs with upstream reports

**Discovery:**
- Found upstream Linux commits

**Resolution:**
- Backported fixes
- Rebuilt sfp.ko
- Loaded patched module

**Outcome:**
- Stable link at 10Gbps   

## Hardware tested

- Banana Pi BPI-R4
- OpenWrt 25.12.5
- Linux 6.12.94
- OEM XGSPONST2001
- 10G SFP+ WAN port

EEPROM identification:

    Vendor: OEM
    Part: XGSPONST2001
    Revision: A-01

## Fix

This is a backport of the upstream Linux SFP quirk for the
OEM XGSPONST2001.

The fix:

- allows prefix matching of the module part number
- associates XGSPONST2001 with `sfp_fixup_potron()`
- ignores the module's broken TX_FAULT/LOS indications
- uses the extended startup handling

Upstream Linux commits:

- f53167e29b8e9178ce030f7de54634af1eb0bc0e
- 03fa69146f2fe18742c0e12cfbf1d10c23d5b567

## Result

With the patched module loaded, the BPI-R4 reported:

    mtk_soc_eth 15100000.ethernet sfp-wan:
    Link is Up - 10Gbps/Full

and the TX_FAULT shutdown loop did not occur.

The patched module also survived a reboot and loaded normally.

## Compatibility

This binary was built specifically for:

    OpenWrt 25.12.5
    Linux 6.12.94
    aarch64
    BPI-R4

Do not assume the binary is compatible with another OpenWrt
release or kernel version.

For other versions, rebuild the module using the included patches.

## Installation

Back up the existing module:

    cp /lib/modules/6.12.94/sfp.ko /root/sfp.ko.stock

Replace it with the patched module and reboot.

The original module can be restored from the backup if necessary.

## Lessons Learned
 
- Upstream Linux may contain fixes not yet available in OpenWrt.
- SFP modules can require vendor-specific quirks.
- Kernel module compatibility is tied to exact kernel versions.
- Verifying upstream patches can save significant troubleshooting time.

## Important

This is an unofficial backport/build for OpenWrt 25.12.5.
The underlying SFP fix is an upstream Linux change.
