---
layout: post
title: "Debugging a BLE mouse that pairs but won't work on Windows 11"
date: 2026-05-31
categories: windows bluetooth debugging tools
---

A Bluetooth Low Energy mouse shows up in the device list. Windows tries to connect. "Try connecting your device again." Repeat forever.

This is the full diagnostic arc — from phantom devices to kernel event logs to a driver version two years out of date — and the CLI tools that cut through it.

## The symptom

Mouse appears in **Settings → Bluetooth & devices → Add a device**. Windows even says it connected once. But the cursor doesn't move, and the device shows a driver error in Device Manager. Removing and re-adding loops back to the same failure.

## Layer 1: Phantom devices

Before blaming the driver, check whether Windows has ghost device records from previous connection attempts or USB dongle sessions.

```powershell
# List all phantoms for a specific device by VID/PID
Get-PnpDevice | Where-Object {
    ($_.Problem -eq 'CM_PROB_PHANTOM') -and
    ($_.InstanceId -match 'VID_046D')   # Logitech VID
} | Select-Object FriendlyName, Class, InstanceId | Format-Table -AutoSize
```

`CM_PROB_PHANTOM` means Windows registered the device at some point but it is no longer present. These ghosts prevent clean re-enumeration — the HID stack sees the old records and thinks the device is already handled.

Remove them with `pnputil`:

```powershell
Get-PnpDevice | Where-Object {
    ($_.Problem -eq 'CM_PROB_PHANTOM') -and
    ($_.InstanceId -match 'VID_046D')
} | ForEach-Object {
    pnputil /remove-device "$($_.InstanceId)"
}
```

Also remove the paired BLE device itself so Windows starts a fresh pairing:

```powershell
# Get the BTHLE instance ID first
Get-PnpDevice | Where-Object InstanceId -match 'BTHLE' | Format-List FriendlyName, InstanceId

# Then remove it
pnputil /remove-device "BTHLE\DEV_<mac>\<instance>"
```

Restart Bluetooth services to flush in-memory pairing state:

```powershell
Stop-Service bthserv -Force
Start-Sleep 2
Start-Service bthserv
```

If pairing still fails after this, the problem is deeper.

## Layer 2: Reading the BLE device tree

Even after phantoms are gone and the device re-pairs, "Try connecting your device again" can persist. Check the full PnP tree:

```powershell
Get-PnpDevice | Where-Object InstanceId -match 'BTHLE' |
    Select-Object FriendlyName, Status, Class, Problem, InstanceId |
    Format-Table -AutoSize
```

A working BLE mouse should look like this:

```
LOGI M240              OK    Bluetooth   CM_PROB_NONE   BTHLE\DEV_...
BLE GATT HID device    OK    Bluetooth   CM_PROB_NONE   BTHLEDEVICE\...
HID-compliant mouse    OK    Mouse       CM_PROB_NONE   HID\{00001812...}&COL01\...
```

If the top two nodes are `OK` but the HID children are `CM_PROB_PHANTOM`, the BLE pairing succeeded and GATT enumerated — but the HID class driver never completed enumeration. This points to a driver-level failure during the BLE HID profile negotiation.

## Layer 3: Kernel event log

The Windows Bluetooth-specific logs (`Microsoft-Windows-Bluetooth-BthLEPrepairing/Operational`) are usually empty. The signal is in the **System** log:

```powershell
Get-WinEvent -LogName System -MaxEvents 500 |
    Where-Object {
        $_.TimeCreated -gt (Get-Date).AddMinutes(-15) -and
        $_.Message -match 'Bluetooth'
    } |
    Select-Object TimeCreated, Id, LevelDisplayName, Message |
    Format-List
```

The smoking gun looks like this:

```
Id              : 5
LevelDisplayName: Error
Message         : The Bluetooth driver expected an HCI event with a certain
                  size but did not receive it.
```

**Event ID 5, repeated** during a pairing attempt = HCI protocol mismatch between the BT controller firmware and the driver. The driver is misreading the BLE advertisement/event packets from the mouse.

## Layer 4: Driver version

Check what's actually installed on the Bluetooth adapter:

```powershell
Get-PnpDevice | Where-Object { $_.Class -eq 'Bluetooth' -and $_.Status -eq 'OK' } |
    ForEach-Object {
        $props = Get-PnpDeviceProperty -InstanceId $_.InstanceId `
            -KeyName DEVPKEY_Device_DriverVersion, DEVPKEY_Device_DriverDate, DEVPKEY_Device_Manufacturer
        [PSCustomObject]@{
            Name    = $_.FriendlyName
            Driver  = ($props | Where-Object KeyName -eq DEVPKEY_Device_DriverVersion).Data
            Date    = ($props | Where-Object KeyName -eq DEVPKEY_Device_DriverDate).Data
            Mfr     = ($props | Where-Object KeyName -eq DEVPKEY_Device_Manufacturer).Data
        }
    } | Format-List
```

Also get the hardware ID to identify the exact chip:

```powershell
Get-PnpDeviceProperty -InstanceId (
    Get-PnpDevice | Where-Object FriendlyName -match 'Realtek.*Bluetooth'
).InstanceId -KeyName DEVPKEY_Device_HardwareIds | Select-Object -ExpandProperty Data
```

In this case: `USB\VID_0BDA&PID_B85C` — Realtek RTL8852BE, driver dated November 2023. The HCI event size parsing bug in this driver series is fixed in `18.4017.xxxx` (January 2025).

**The Windows Update Catalog does not have the newer BT driver.** AMD's driver page redirects to OEM. The right path is the OEM's driver management tool.

## Fix: HP CMSL (for HP machines)

HP ships a PowerShell module — **HP Client Management Script Library** — that queries HP's SoftPaq catalog for the exact machine model and finds applicable updates by category.

```powershell
Install-Module -Name HPCMSL -Force -AcceptLicense -Scope CurrentUser

Import-Module HPCMSL

# Find all Bluetooth/network driver updates for this machine
Get-SoftpaqList -Category 'Driver - Network' |
    Where-Object { $_.Name -match 'Bluetooth|RTL885' } |
    Select-Object Id, Name, Version, ReleaseDate |
    Format-Table -AutoSize
```

Output:

```
Id       Name                                    Version    ReleaseDate
--       ----                                    -------    -----------
sp157134 Realtek RTL8xxx Series Bluetooth Driver 1.0.0.166  21-Feb-25
```

Download and install silently:

```powershell
Get-Softpaq -Number sp157134 -DestinationPath "$env:TEMP\sp157134" -Overwrite Yes
Start-Process "$env:TEMP\sp157134\sp157134.exe" -ArgumentList '/s' -Wait
```

After install, verify the driver version changed:

```powershell
Get-PnpDeviceProperty -InstanceId 'USB\VID_0BDA&PID_B85C\<serial>' `
    -KeyName DEVPKEY_Device_DriverVersion, DEVPKEY_Device_DriverDate |
    Select-Object KeyName, Data | Format-Table -AutoSize
```

Before: `1.10.1061.3014` (Nov 2023). After: `18.4017.2412.2303` (Jan 2025).

Put the mouse in pairing mode → pair fresh → done.

## Bonus: Full driver audit

While you have HPCMSL loaded, run a full audit of what else is stale:

```powershell
Import-Module HPCMSL

Get-SoftpaqList -Category 'Driver - Network','Driver - Chipset','Driver - Audio',
    'Driver - Storage','Driver - Keyboard, Mouse and Input Devices','Firmware','BIOS' |
    Select-Object Id, Name, Version, ReleaseDate |
    Format-Table -AutoSize
```

Compare against installed versions:

```powershell
$categories = @(
    @{Name='Realtek WiFi'; Pattern='Realtek.*Wireless|Realtek.*802'},
    @{Name='AMD Chipset';  Pattern='AMD.*Chipset|AMD.*GPIO|AMD.*SMBus'},
    @{Name='Realtek Audio';Pattern='Realtek.*Audio|Realtek.*HD'}
)
foreach ($d in $categories) {
    $dev = Get-PnpDevice |
        Where-Object { $_.FriendlyName -match $d.Pattern -and $_.Status -eq 'OK' } |
        Select-Object -First 1
    if ($dev) {
        $ver  = (Get-PnpDeviceProperty -InstanceId $dev.InstanceId -KeyName DEVPKEY_Device_DriverVersion -EA SilentlyContinue).Data
        $date = (Get-PnpDeviceProperty -InstanceId $dev.InstanceId -KeyName DEVPKEY_Device_DriverDate -EA SilentlyContinue).Data
        "$($d.Name): $ver [$($date.ToString('MMM yyyy'))]"
    }
}
```

Skip SSD firmware and BIOS from bulk updates — those deserve individual review.

## Summary

| Step | Tool | What it told us |
|---|---|---|
| List phantom devices | `Get-PnpDevice` + `CM_PROB_PHANTOM` | 11 ghost HID records blocking re-enumeration |
| Remove phantoms | `pnputil /remove-device` | Clean PnP tree |
| Read BLE device tree | `Get-PnpDevice` on BTHLE | BLE paired OK, HID children phantom |
| Kernel event log | `Get-WinEvent -LogName System` | Event ID 5: HCI size mismatch |
| Identify adapter | `DEVPKEY_Device_HardwareIds` | RTL8852BE, driver Nov 2023 |
| Find updated driver | HP CMSL `Get-SoftpaqList` | sp157134 (Feb 2025) |
| Install | `Get-Softpaq` + silent exe | Driver: 18.4017.2412.2303 |

The diagnostic order matters: phantom cleanup is necessary but not sufficient. The kernel event log is the only signal that points to the actual failure layer.

## References

- [HP Client Management Script Library docs](https://developers.hp.com/hp-client-management)
- [pnputil command syntax](https://learn.microsoft.com/en-us/windows-hardware/drivers/devtest/pnputil-command-syntax)
- [DEVPKEY reference](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/devpkey-device-driverversion)
- [Windows BLE HID enumeration](https://learn.microsoft.com/en-us/windows-hardware/design/component-guidelines/bluetooth-hid-over-gatt)
