# Kerub — Recovery Procedures

| | |
|---|---|
| **Status** | Draft — procedures are marked with the roadmap task that implements them |
| **Related** | [lab-setup.md](lab-setup.md), [architecture.md](../architecture.md) §8.4, [threat-model.md](../threat-model.md) (TM-RSP-1), [requirements.md](../requirements.md) (FR-CORE-007, NFR-REL-001) |

Kerub's first principle is **never lock the user out**. This document
describes how to get control back when something goes wrong anyway: no
network, blocked devices, a service that misbehaves, a forgotten password.

> **Keep this document available offline.** If the network is cut, a web
> page is useless. Kerub installs a local copy
> (`C:\Program Files\Kerub\docs\recovery.html`), linked from the agent's
> tray menu. Consider printing the escalation ladder (§2).

---

## 1. Before enabling enforcement

Check these **before** switching any module from audit to enforce mode:

- [ ] You know the password of a local **administrator** account.
- [ ] Your **BitLocker recovery key** is saved somewhere other than this PC
      (printed, or in your Microsoft account). Without it, offline recovery
      (§4.6) is impossible on an encrypted disk.
- [ ] You know how to reach the **Windows Recovery Environment** (§5).
- [ ] A **System Restore point** exists (the installer creates one before
      the first enforcement).
- [ ] You have read this document once.

---

## 2. Escalation ladder

Always start at the top: each step is more drastic than the previous one.

| Step | Tool | Needs | Effect |
|------|------|-------|--------|
| 1 | Agent: *Undo* on an action, or maintenance mode | Unlock password / elevation | Reverts one action, or suspends enforcement for up to 60 min |
| 2 | `kerub-cli` (elevated) | Administrator account, running service | Same, from a command line |
| 3 | `kerub-cli emergency-reset` | Administrator account; **no service needed** | Removes all of Kerub's enforcement at once |
| 4 | `emergency-reset.ps1` | Administrator account; no Kerub binary needed | Same as step 3, in PowerShell |
| 5 | Safe Mode | Administrator account | Kerub's service does not start; run step 3 or 4 |
| 6 | Windows Recovery Environment | BitLocker recovery key if encrypted | Offline removal of device policies |
| 7 | System Restore | — | Returns the whole system configuration to before Kerub's enforcement |
| 8 | Uninstall | Administrator account | Removes Kerub completely |

In the lab, the real step 1 is simply: **revert the VM snapshot**.

---

## 3. Scenarios

### 3.1 No network access

*Causes: kill switch active while the VPN is down, captive portal, faulty
filter, IP block hitting the gateway.*

1. Check the tray icon: if the kill switch is active and the VPN is down,
   that is **the kill switch doing its job**. Reconnect the VPN, or use
   *Allow captive portal* for a hotel or station network.
2. Otherwise, look at the timeline for a recent `ip-blocked` or
   `network-profile` action and **Undo** it.
3. Still offline: open an elevated terminal and run
   `kerub-cli maintenance enter --minutes 30`.
4. Still offline, or the service does not answer:
   `kerub-cli emergency-reset`.

Local access (keyboard, screen, local accounts) is never affected by network
filters: steps 1 to 4 always remain possible.

### 3.2 A USB storage device is blocked

Not a lockout: this is Kerub working. Approve the device from the prompt,
or from the agent's *Devices* page. If the prompt does not appear, enter
maintenance mode (step 1 or 2).

### 3.3 Keyboard or mouse does not work

*Cause: a new input device waiting for confirmation (FR-DEV-003).*

This is the most serious lockout, because recovery tools need input. It is
prevented by design (§6), but if it happens:

1. Approve it with your **mouse**, or with a keyboard Kerub already trusts:
   the prompt ignores input from the device in question. Without an answer
   within 60 seconds the device stays blocked; approve it later from the
   agent's *Devices* page.
2. Plug the device into the **same port** it used before: Windows may see a
   device moved to another port as a new device.
3. Reconnect a previously approved keyboard or mouse, or use the laptop's
   built-in keyboard and touchpad.
4. Last resort: offline removal from the Recovery Environment (§4.6).

### 3.4 The service crashes or restarts in a loop

1. Collect information: `kerub-cli diagnostics export`.
2. If protections are in the way, `kerub-cli emergency-reset`.
3. Stop the loop: boot into **Safe Mode**, where Kerub's service does not
   start, then set it to manual start
   (`sc config KerubSvc start= demand`) and restart normally.
4. Report the issue with the diagnostic bundle (review it first, see
   [privacy.md](../privacy.md) §6).

### 3.5 Forgotten unlock password

The unlock password cannot be recovered, only replaced. From an elevated
terminal: `kerub-cli password reset`. This is logged in the audit log.

### 3.6 Invalid configuration or damaged database

- **Configuration:** the service falls back automatically to the last known
  good version and raises an alert. To choose a version explicitly:
  `kerub-cli config rollback <version>`.
- **Database:** stop the service, move `kerub.db` out of
  `C:\ProgramData\Kerub\data\`, start the service. Kerub creates an empty
  database; history is lost, protections are rebuilt from the configuration.

---

## 4. Tools in detail

### 4.1 Maintenance mode *(task: CORE milestone)*

`kerub-cli maintenance enter --minutes <1-60> --reason "<text>"` suspends
every enforcement for the given duration. Protections come back
automatically at the end. `kerub-cli maintenance exit` ends it early.

### 4.2 `kerub-cli emergency-reset` *(task: before any WFP or device-policy task)*

Designed to work when everything else is broken. It:

- **does not need** the service, the database, the configuration or the
  network;
- reads what to undo from Kerub's **footprint manifest** (§4.4);
- stops the service (and, with `--disable-service`, prevents it from
  starting again);
- deletes every WFP filter owned by Kerub's provider, then Kerub's
  sublayer, then the provider itself;
- removes every device installation restriction Kerub created;
- restores system settings to the previous values recorded in the manifest;
- is **idempotent**: running it twice is harmless;
- supports `--dry-run`, which lists what would be removed without changing
  anything;
- logs what it did to the Windows Event Log and to
  `C:\ProgramData\Kerub\logs\emergency-reset.log`.

### 4.3 `emergency-reset.ps1` *(same task)*

A PowerShell fallback for when the Kerub binaries are damaged or missing.
Installed as `C:\Program Files\Kerub\scripts\emergency-reset.ps1` and kept
in the repository under `scripts/`. It performs the same steps as 4.2, using
the WFP management API through inline P/Invoke declarations. Run it from an
elevated PowerShell:

```
powershell -ExecutionPolicy Bypass -File "C:\Program Files\Kerub\scripts\emergency-reset.ps1"
```

### 4.4 Footprint manifest

Every change Kerub makes outside its own folders — WFP objects, device
policies, system settings with their previous values — is also recorded in
a small **footprint manifest** stored in the registry under
`HKLM\SOFTWARE\Kerub\Footprint` (SYSTEM and Administrators only),
independently of the database. This is what lets the emergency reset work
even if the database is gone.

This mechanism complements the action journal
([architecture.md](../architecture.md) §5.3).

### 4.5 Safe Mode

Only services explicitly registered for Safe Mode start there; Kerub's
service is not one of them. Safe Mode is therefore a reliable way to get a
session where Kerub is not running, to run the emergency reset.

### 4.6 Offline removal from the Recovery Environment

If input devices are unusable (§3.3), boot into the Windows Recovery
Environment, where Windows loads its own drivers and Kerub's policies do
not apply. From *Troubleshoot → Advanced options → Command Prompt*:

1. Unlock the disk if BitLocker asks for the recovery key.
2. Load the installed system's `SOFTWARE` hive with `reg load`.
3. Delete the device installation restrictions under
   `Policies\Microsoft\Windows\DeviceInstall\Restrictions` in the loaded
   hive.
4. Unload the hive with `reg unload`, and restart.

The exact commands are finalized and tested in the lab with the device
control task.

### 4.7 System Restore

The installer creates a restore point before the first enforcement. Restoring
it returns system settings and firewall policy to their state before Kerub
enforced anything. Personal files are not affected.

---

## 5. Reaching the Recovery Environment and Safe Mode

- From Windows: hold **Shift** while clicking **Restart**, then
  *Troubleshoot → Advanced options*.
- If Windows cannot be used: interrupt the boot (power off during the Windows
  logo) twice in a row; the third boot opens the Recovery Environment.
- Or boot from a Windows installation USB stick and choose *Repair your
  computer*.
- Safe Mode: *Troubleshoot → Advanced options → Startup Settings → Restart*,
  then option 4.

---

## 6. Safeguards built into Kerub

Recovery is the last line of defense; these design rules make it rarely
needed. They must be implemented before the feature they protect is
enabled in enforce mode.

| Safeguard | Protects against |
|-----------|------------------|
| Every module starts in **audit mode** (SEC-RSP-001) | Untested rules blocking real use |
| **Never-block list**: loopback, gateway, DNS servers, VPN server (FR-RSP-002) | Kerub cutting its own machine off |
| Temporary blocks always **expire** | Forgotten blocks |
| Input devices **present at installation** and built-in keyboards and touchpads are approved automatically | Keyboard or mouse lockout |
| New input devices can be approved with the **mouse or any already-trusted keyboard** | Keyboard or mouse lockout |
| **Captive portal** exception (FR-NET-008) | Kill switch preventing any public Wi-Fi use |
| Emergency reset and this document **exist and are tested before** the first WFP or device-policy feature | Having no way back |

---

## 7. Testing the recovery tools

Every release is checked in the lab, from the `lab-baseline` snapshot:

- [ ] Enable every module in enforce mode, then run `kerub-cli emergency-reset`:
      network and devices work again, `netsh wfp show filters` shows no Kerub
      filter.
- [ ] Same with `emergency-reset.ps1`, after deleting the Kerub binaries.
- [ ] Run each reset twice: no error the second time.
- [ ] Delete `kerub.db`, run the reset: it still works (footprint manifest).
- [ ] Boot into Safe Mode: `kerub-svc` is not running.
- [ ] Offline removal from the Recovery Environment restores a blocked
      keyboard.