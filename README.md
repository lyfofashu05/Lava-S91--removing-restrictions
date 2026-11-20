<div align="center">

# Lava Tab S91 — Device Owner (CloudDPC) Removal Guide

### Force-remove a stuck Work Profile / Device Owner app on the Lava Tab S91 when normal uninstall fails

![Tablet](https://img.shields.io/badge/Tablet-Lava%20Tab%20S91-A5D8FF?style=flat)
![No Root](https://img.shields.io/badge/Bootloader-No%20Root-B2F2BB?style=flat)
![Termux](https://img.shields.io/badge/Tool-Termux-FFD6A5?style=flat)
![License](https://img.shields.io/badge/License-MIT-FFC9DE?style=flat)
![Status](https://img.shields.io/badge/Status-Tested-D0BFFF?style=flat)

</div>

---

## Before You Start

> [!WARNING]
> This guide removes a Device Owner (DPC) app such as `com.google.android.apps.work.clouddpc` from a **Lava Tab S91** by editing system files directly, using a temporary root shell. Read this section fully before running anything.
>
> - Use this only on **your own tablet** — for example, a second-hand Lava Tab S91 with leftover MDM enrollment from a previous owner.
> - Do not use this on a company-issued or school-issued tablet you are not authorized to de-enroll.
> - Always back up before deleting anything. Backup commands are included below — do not skip them.
> - Editing system files incorrectly can cause a bootloop. Follow the steps in order, exactly as written.

> [!IMPORTANT]
> This guide was tested on a **Lava Tab S91 running the October 2026 security patch**. File paths, package names, and table structures may differ on other security patch levels or firmware builds. Tread carefully — verify each step's output against your own device before proceeding to the next.

> [!CAUTION]
> **Disclaimer:** I take no responsibility for anything that happens to your device while following this guide — including bootloops, data loss, or a hard brick. You are following these steps entirely at your own risk and on your own judgment.

---

## Table of Contents

- [What You Need](#what-you-need)
- [Step 1 — Get Temporary Root](#step-1--get-temporary-root)
- [Step 2 — Check Device Owner Status](#step-2--check-device-owner-status)
- [Step 3 — Remove Device Owner Policy Files](#step-3--remove-device-owner-policy-files)
- [Step 4 — Uninstall the App](#step-4--uninstall-the-app)
- [Step 5 — Clean the Work Account from SQLite](#step-5--clean-the-work-account-from-sqlite)
- [Step 6 — Clear Play Store & Play Services Cache](#step-6--clear-play-store--play-services-cache)
- [Step 7 — Final Reboot & Verification](#step-7--final-reboot--verification)
- [Backups](#backups)
- [FAQ](#faq)

---

## What You Need

| Requirement | Purpose | Link |
|---|---|---|
| Lava Tab S91 | The tablet this guide is written for | — |
| Termux | Terminal app used to run every command below | [F-Droid](https://f-droid.org/packages/com.termux/) |
| DirtyFrag | Exploit-based tool used to obtain a temporary root shell without unlocking the bootloader | [Releases](https://github.com/mitschud/DirtyFrag/releases) |
| KernelSU | Alternative root manager, usable if DirtyFrag does not grant root on your build | [Releases](https://github.com/tiann/KernelSU/releases) |
| `sqlite3` package | Lets us edit Android's account database | Installed with `pkg install sqlite -y` |

> [!NOTE]
> This guide uses a **temporary root** session (via DirtyFrag or KernelSU) rather than a permanently unlocked bootloader. If your `su` session resets after reboot, you will need to re-run the root tool again before continuing with later steps.

### Factory Reset via Recovery Mode (if you need a clean starting point)

If you'd rather start fresh before attempting root, or want a clean baseline to fall back to:

1. Power off the tablet completely.
2. Press and hold the **Power button**.
3. The tablet will vibrate **twice** as it boots. The moment you feel the **first vibration**, press and hold **Volume Up** as well (while still holding Power).
4. This will boot you into **Recovery Mode**.
5. Navigate with Volume buttons and select with Power to choose **Wipe Data / Factory Reset**, then confirm.
6. Reboot the system once the wipe finishes.

---

## Step 1 — Get Temporary Root

Download and run **DirtyFrag** on the Lava Tab S91 following its release instructions:

[github.com/mitschud/DirtyFrag/releases](https://github.com/mitschud/DirtyFrag/releases)

Run the exploit and **wait** for it to open KernelSU

[github.com/tiann/KernelSU/releases](https://github.com/tiann/KernelSU/releases)

Once done you will see a green banner with **Jailbreak Mode** in KernelSU, confirm it works inside Termux:

```bash
su
```

Your prompt should change (commonly ending in `#` instead of `$`), confirming root access.

---

## Step 2 — Check Device Owner Status

With root access active, check whether a Device Owner is set:

```bash
dpm list-owners
```

Expected output if a Device Owner is present:

```
User 0: admin=com.google.android.apps.work.clouddpc/.receivers.CloudDeviceAdminReceiver,DeviceOwner,Affiliated
```

If `DeviceOwner` appears next to a package name, that app has full management control over the tablet and cannot be removed through a normal uninstall.

> [!NOTE]
> Running `dpm remove-active-admin` directly on a true Device Owner will fail with `SecurityException: Attempt to remove non-test admin`. This is expected — proceed to Step 3 instead.

---

## Step 3 — Remove Device Owner Policy Files

Android stores the Device Owner's ownership record as a few files in `/data/system`.

Go to the folder:

```bash
cd /data/system
```

List the relevant files (names can vary slightly by build):

```bash
ls -la | grep -i device
```

You should see files similar to:

```
device_owner_2.xml
device_policies.xml
device_policies_version
device_policy_state.xml
```

Create a backup folder:

```bash
mkdir -p /sdcard/dpm_backup
```

Back up the policy files:

```bash
cp device_owner_2.xml /sdcard/dpm_backup/
cp device_policies.xml /sdcard/dpm_backup/
cp device_policy_state.xml /sdcard/dpm_backup/
```

Delete them:

```bash
rm device_owner_2.xml
rm device_policies.xml
rm device_policy_state.xml
```

Reboot:

```bash
reboot
```

After the tablet restarts, re-establish root (Step 1, if needed) and verify:

```bash
su
dpm list-owners
```

Expected result: empty output — no Device Owner remains.

---

## Step 4 — Uninstall the App

```bash
su
pm uninstall --user 0 com.google.android.apps.work.clouddpc
```

If that fails, disable it instead:

```bash
pm disable-user --user 0 com.google.android.apps.work.clouddpc
```

---

## Step 5 — Clean the Work Account from SQLite

Even after the app is removed, a leftover work account entry (something like `work-XXXXXXXX@android-for-work.gserviceaccount.com`) can remain in Android's internal account databases. Removing it from Settings directly can crash Settings or trigger a factory reset prompt, so it is removed here instead.

### 5.1 — Locate the database files

```bash
find /data/system* -iname "accounts*.db" 2>/dev/null
```

Expected paths:

```
/data/system_ce/0/accounts_ce.db
/data/system_de/0/accounts_de.db
```

### 5.2 — Back up both files

```bash
mkdir -p /sdcard/dpm_backup
cp /data/system_ce/0/accounts_ce.db /sdcard/dpm_backup/
cp /data/system_de/0/accounts_de.db /sdcard/dpm_backup/
```

### 5.3 — Install sqlite3 (normal Termux shell, not inside `su`)

```bash
pkg install sqlite -y
```

### 5.4 — Clean `accounts_ce.db`

> [!IMPORTANT]
> Inside a root (`su`) shell, Termux's installed tools are often not on the `PATH`. Always call `sqlite3` using its full path, shown below, to avoid a `sqlite3: inaccessible or not found` error.

```bash
/data/data/com.termux/files/usr/bin/sqlite3 /data/system_ce/0/accounts_ce.db
```

Inside the SQLite prompt:

```sql
.tables
SELECT _id, name, type FROM accounts;
```

Find the row where `type` is `com.google.work` and note its `_id` (commonly `1`). Then delete it and its related rows — `accounts_ce.db` typically has `authtokens` and `extras` tables:

```sql
DELETE FROM authtokens WHERE accounts_id = 1;
DELETE FROM extras WHERE accounts_id = 1;
DELETE FROM accounts WHERE _id = 1;
```

Verify:

```sql
SELECT * FROM accounts;
```

Exit:

```sql
.quit
```

### 5.5 — Clean `accounts_de.db`

This is a separate database with a different table structure, so it must be cleaned independently.

```bash
/data/data/com.termux/files/usr/bin/sqlite3 /data/system_de/0/accounts_de.db
```

```sql
.tables
SELECT _id, name, type FROM accounts;
```

`accounts_de.db` typically does not have `authtokens`/`extras`, but has `grants` and `shared_accounts` instead:

```sql
DELETE FROM grants WHERE accounts_id = 1;
DELETE FROM shared_accounts WHERE account_id = 1;
DELETE FROM accounts WHERE _id = 1;
```

> [!TIP]
> If a `DELETE` command returns `no such table`, that table does not exist in this database — skip that line and continue. Trust what `.tables` shows over the example above, since table names can vary slightly between builds.

Verify:

```sql
SELECT * FROM accounts;
```

Exit:

```sql
.quit
exit
```

---

## Step 6 — Clear Play Store & Play Services Cache

The work profile often leaves sync data cached inside Play Store, and sometimes Play Services. Clear it to finish the cleanup:

```bash
su
pm clear com.android.vending
```

Optional, deeper clean:

```bash
pm clear com.google.android.gms
```

> [!CAUTION]
> Clearing `com.google.android.gms` signs you out of every Google account on the tablet, including your personal one. Only run this if you are prepared to sign back in afterward.

---

## Step 7 — Final Reboot & Verification

```bash
exit
reboot
```

After the tablet restarts, re-establish root if needed and run the full checklist:

```bash
su
dpm list-owners
pm list packages | grep clouddpc
```

### Success Checklist

- [ ] `dpm list-owners` returns nothing — no Device Owner
- [ ] `pm list packages | grep clouddpc` returns nothing, or shows the app disabled
- [ ] Settings → Accounts opens normally, with no crash or reset prompt
- [ ] No work account visible anywhere in Settings
- [ ] Play Store opens and asks for a fresh sign-in (expected after `pm clear`)

---

## Backups

Everything backed up during this process is saved here:

```
/sdcard/dpm_backup/
├── device_owner_2.xml
├── device_policies.xml
├── device_policy_state.xml
├── accounts_ce.db
└── accounts_de.db
```

Keep this folder until the tablet is confirmed stable, then delete it alongwith above three apps if youre not familiar with superuser. 

---

## FAQ

<details>
<summary><b>Why does <code>dpm remove-active-admin</code> fail?</b></summary>
<br>
Because the app is set as full Device Owner, not just a regular device admin. Android only allows that command to remove a Device Owner on test/debug builds. On the Lava Tab S91's production firmware, the ownership record has to be cleared directly from <code>/data/system</code>, which is what Step 3 does.
</details>

<details>
<summary><b>Why clean two separate SQLite databases?</b></summary>
<br>
Android keeps two copies of account data:
<ul>
<li><code>accounts_ce.db</code> — Credential Encrypted, accessible only after the tablet is unlocked</li>
<li><code>accounts_de.db</code> — Device Encrypted, accessible even before unlock, for early boot services</li>
</ul>
The work account can exist in both, so both need cleaning.
</details>

<details>
<summary><b>I got "sqlite3: inaccessible or not found" — what now?</b></summary>
<br>
The root (<code>su</code>) shell does not inherit Termux's PATH. Always use the full path:

```bash
/data/data/com.termux/files/usr/bin/sqlite3 <db-path>
```
</details>

<details>
<summary><b>Do I need to re-root after every reboot?</b></summary>
<br>
If root was obtained through DirtyFrag's exploit rather than a persistent method, yes — the root session can reset on reboot. Re-run the tool from Step 1 before continuing with later steps. KernelSU, once installed, generally persists across reboots.
</details>

<details>
<summary><b>What if the Device Owner comes back after reboot?</b></summary>
<br>
This can happen if the tablet re-checks in with an MDM server over the internet and gets re-enrolled automatically (common when the account shows "Affiliated" status). If this happens, enable Airplane Mode right after reboot, complete Steps 4–6 while offline, then reconnect.
</details>

---

## Support This Project

<div align="center">

If this guide saved your tablet, consider supporting it.

[![Buy Me A Chai](https://img.shields.io/badge/Buy%20Me%20A%20Chai-FFD6A5?style=flat)](https://buymeachai.in/lyfofashu)

</div>

---

<div align="center">

**Did this help you?** Consider starring this repo — it helps others find it too.

If something didn't work, [raise an issue](../../issues) and I'll take a look.

Use responsibly — only on tablets you own and control.

</div>
