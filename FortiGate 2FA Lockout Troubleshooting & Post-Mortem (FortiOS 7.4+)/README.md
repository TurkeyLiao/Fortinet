# FortiGate 2FA Lockout Troubleshooting & Post-Mortem (FortiOS 7.4+)

## Overview
This document outlines a lockout scenario on FortiGate (FortiOS 7.4+) triggered by manually re-binding a FortiToken to the sole `super_admin` account, the recovery paths evaluated, official Fortinet TAC feedback, and the required recovery procedure.

---

## 1. Environment Details
* **Device Model:** FortiGate 60F
* **Firmware Version:** FortiOS 7.4.x (Build 2731)
* **Account Configuration:** Single `super_admin` account, no secondary administrator configured.
* **Management Tier:** FortiGate Cloud (Free Tier, without paid Standard/Premium subscription).

---

## 2. Problem Description & Root Cause

### Trigger
While reviewing administrator settings in the GUI, the administrator noticed the FortiToken toggle appeared unselected on the active `super_admin` account. Manually re-selecting the assigned FortiToken and saving triggered an immediate session termination.

### Root Cause
1. **Token Re-initialization Failure:** Re-assigning or re-saving the FortiToken configuration invalidated the existing seed/OTP generation on the mobile FortiToken app.
2. **Missing Activation Dispatch:** The system did not generate or dispatch a new activation code / QR code upon saving.
3. **Strict 2FA Enforcement:** All subsequent access vectors (GUI, SSH, and local serial console) strictly required the new OTP token, which was inaccessible.

---

## 3. Evaluated Recovery Paths & Limitations

| Recovery Vector | Status | Reason for Failure |
| :--- | :--- | :--- |
| **Mobile FortiToken App** | **Blocked** | Pre-existing OTP tokens invalidated; token de-synchronized. |
| **Local Serial Console** | **Blocked** | FortiOS enforces 2FA across local console connections; no local bypass. |
| **Maintainer Account (`bcpb`)** | **Unavailable** | The legacy maintainer backdoor mechanism was permanently removed in FortiOS 7.4+. |
| **FortiGate Cloud CLI Scripting** | **Unavailable** | Executing CLI scripts to remove 2FA requires a paid FortiGate Cloud Standard/Premium license. |
| **FortiGate Cloud Config Backup** | **Blocked** | Fetching/restoring via cloud prompted a mandatory firmware patch upgrade, which in turn required 2FA authentication. |
| **Fortinet TAC Remote Assistance** | **Unavailable** | TAC cannot bypass or inject credentials without an accessible admin session. |

---

## 4. TAC Official Assessment
> **TAC Summary:**  
> When all administrative access paths are blocked by 2FA, no secondary `super_admin` exists, FortiManager/paid Cloud is absent, and the maintainer account is non-functional in FortiOS 7.4+, **no non-destructive recovery path exists**. The only viable resolution is a factory reset and configuration restore.

---

## 5. Resolution & Recovery Procedure

Because the most recent configuration backup was 2 months old, the following recovery process must be executed:

### Step 1: Access Bootloader & Factory Reset
1. Connect via RJ45/USB Serial Console cable (Baud rate: `9600`, Data bits: `8`, Parity: `None`, Stop bits: `1`, Flow control: `None`).
2. Power-cycle the FortiGate unit.
3. Press any key during boot when prompted to enter the bootloader menu.
4. Format the boot device / reset configuration to factory defaults.

### Step 2: Sanitize Backup Configuration File
Open the offline backup `.conf` file in a text editor and strip the two-factor authentication parameters from the administrator profile:

```fortios
config system admin
    edit "admin"
        unset two-factor
        unset fortitoken
    next
end
```

### Step 3: Restore Configuration
1. Log in to the factory-reset device via console or default IP (`192.168.1.99` on `mgmt` / `internal` interface).
2. Restore the sanitized configuration file via Web GUI (`System > Configuration > Restore`) or TFTP/USB via CLI.
3. Verify basic connectivity and re-provision missing configuration deltas from the last 2 months.

---

## 6. Preventative Best Practices

To prevent complete lockout scenarios in FortiOS 7.4+:

1. **Secondary Super Admin:** Always maintain a secondary, local break-glass `super_admin` account with a strong password and no 2FA (or hardware-based token) stored in a secure credential vault.
2. **Pre-Change Backups:** Export a full configuration backup immediately prior to modifying administrative authentication settings.
3. **Maintainer Removal Awareness:** Treat all administrative changes under FortiOS 7.4+ with zero expectation of console `bcpb` password recovery.
4. **Cloud/FMG Out-of-Band Channel:** Ensure an active management channel capable of pushing CLI templates (FortiManager or paid Cloud tier) if physical console intervention is restricted.
