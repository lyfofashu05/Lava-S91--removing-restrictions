<div align="center">

# 📱 Android Device Owner (CloudDPC) Removal Guide

### Force-remove a stuck Work Profile / Device Owner app when normal uninstall fails

![Platform](https://img.shields.io/badge/platform-Android-3DDC84?style=for-the-badge&logo=android&logoColor=white)
![Root Required](https://img.shields.io/badge/root-required-critical?style=for-the-badge&logo=linux&logoColor=white)
![Termux](https://img.shields.io/badge/tool-Termux-000000?style=for-the-badge&logo=gnu-bash&logoColor=white)
![License](https://img.shields.io/badge/license-MIT-blue?style=for-the-badge)
![Status](https://img.shields.io/badge/status-tested-success?style=for-the-badge)

</div>

---

## ⚠️ Read This First

> [!WARNING]
> This guide removes a **Device Owner (DPC)** app like `com.google.android.apps.work.clouddpc` by editing system files directly. This only works on a **rooted** device.
>
> - ✅ Use this on **your own device** — e.g. a second-hand phone with leftover MDM enrollment from a previous owner.
> - ❌ Do **not** use this on a company-issued or school-issued device you don't have authorization to de-enroll. It may violate your organization's policy.
> - 🧯 Always back up before deleting anything. Steps below include backup commands — don't skip them.
> - 💥 Editing system files incorrectly can cause a bootloop. Follow the steps exactly, in order.

---

## 📋 Table of Contents

1. [What You Need](#-what-you-need)
2. [Step 1 — Check Device Owner Status](#1️⃣-step-1--check-device-owner-status)
3. [Step 2 — Remove Device Owner Policy Files](#2️⃣-step-2--remove-device-owner-policy-files)
4. [Step 3 — Uninstall the App](#3️⃣-step-3--uninstall-the-app)
5. [Step 4 — Clean the Work Account from SQLite](#4️⃣-step-4--clean-the-work-account-from-sqlite)
6. [Step 5 — Clear Play Store & Play Services Cache](#5️⃣-step-5--clear-play-store--play-services-cache)
7. [Step 6 — Final Reboot & Verification](#6️⃣-step-6--final-reboot--verification)
8. [Backups](#️-backups)
9. [FAQ](#-faq)

---

## 🧰 What You Need

| Requirement | Why |
|---|---|
| Rooted Android device | We edit system files that only root can touch |
| [Termux](https://f-droid.org/packages/com.termux/) installed | Gives us a Linux terminal on Android |
| `sqlite3` package | Lets us edit Android's account database — install with `pkg install sqlite -y` |
| `su` access working | Required for every command below |

> 💡 **Beginner tip:** every command below is meant to be copy-pasted *exactly as written*. Don't retype them by hand — one typo in a system file path can cause real problems.

---

## 1️⃣ Step 1 — Check Device Owner Status

Open Termux and type:

```bash
su
```

This gives you root access. Your prompt will usually change to end in `#` instead of `$`.

Now check if there's a Device Owner set on the device:

```bash
dpm list-owners
```

**What you're looking for:** output like this —

```
User 0: admin=com.google.android.apps.work.clouddpc/.receivers.CloudDeviceAdminReceiver,DeviceOwner,Affiliated
```

If you see `DeviceOwner` next to a package name, that app has **full control** over your device — it's not a regular app, and it can't be uninstalled normally. That's exactly why we need this guide.

> [!NOTE]
> Trying `dpm remove-active-admin` on a true Device Owner will fail with a `SecurityException: Attempt to remove non-test admin`. This is expected — don't worry, it just means we need the next step.

---

## 2️⃣ Step 2 — Remove Device Owner Policy Files

Android stores the Device Owner's "ownership record" as a few files in `/data/system`. We're going to back them up, then delete them, which clears the ownership at the system level.

**Go to the folder:**

```bash
cd /data/system
```

**See what's actually there** (file names can differ slightly between Android versions):

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

**Create a backup folder on your SD card:**

```bash
mkdir -p /sdcard/dpm_backup
```

**Back up the policy files:**

```bash
cp device_owner_2.xml /sdcard/dpm_backup/
cp device_policies.xml /sdcard/dpm_backup/
cp device_policy_state.xml /sdcard/dpm_backup/
```

**Now delete them:**

```bash
rm device_owner_2.xml
rm device_policies.xml
rm device_policy_state.xml
```

**Reboot the device:**

```bash
reboot
```

**After the device restarts**, open Termux again and verify:

```bash
su
dpm list-owners
```

✅ Expected result: **empty output** — meaning no Device Owner exists anymore.

---

## 3️⃣ Step 3 — Uninstall the App

Now that the Device Owner lock is gone, a normal uninstall should work:

```bash
su
pm uninstall --user 0 com.google.android.apps.work.clouddpc
```

If that fails for any reason, disable it instead so it stops running:

```bash
pm disable-user --user 0 com.google.android.apps.work.clouddpc
```

---

## 4️⃣ Step 4 — Clean the Work Account from SQLite

Even after the app is gone, a leftover **work account** entry (something like `work-XXXXXXXX@android-for-work.gserviceaccount.com`) can sit inside Android's internal account databases. If you try to remove it from Settings → Accounts, it can crash Settings or trigger a factory reset prompt. We remove it directly and safely.

### 4.1 — Find the database files

```bash
find /data/system* -iname "accounts*.db" 2>/dev/null
```

You'll get two paths:

```
/data/system_ce/0/accounts_ce.db
/data/system_de/0/accounts_de.db
```

### 4.2 — Make a backup folder and back up both files

```bash
mkdir -p /sdcard/dpm_backup
cp /data/system_ce/0/accounts_ce.db /sdcard/dpm_backup/
cp /data/system_de/0/accounts_de.db /sdcard/dpm_backup/
```

### 4.3 — Install sqlite3 inside Termux (if not already installed)

> Run this in your **normal Termux shell**, not inside `su`:

```bash
pkg install sqlite -y
```

### 4.4 — Open and clean `accounts_ce.db`

> [!IMPORTANT]
> Inside an `su` root shell, Termux's installed tools are often **not** on the `PATH`. Always call `sqlite3` using its **full path** as shown below — this avoids the common `sqlite3: inaccessible or not found` error.

```bash
/data/data/com.termux/files/usr/bin/sqlite3 /data/system_ce/0/accounts_ce.db
```

You're now inside the SQLite prompt. Run:

```sql
.tables
SELECT _id, name, type FROM accounts;
```

Look for a row where `type` is `com.google.work`. Note the `_id` number — it is almost always `1`.

Now delete it and its related rows. `accounts_ce.db` typically has `authtokens` and `extras` tables:

```sql
DELETE FROM authtokens WHERE accounts_id = 1;
DELETE FROM extras WHERE accounts_id = 1;
DELETE FROM accounts WHERE _id = 1;
```

Verify it's gone:

```sql
SELECT * FROM accounts;
```

The work account row should no longer appear. Exit SQLite:

```sql
.quit
```

### 4.5 — Open and clean `accounts_de.db`

This is a **separate database file** with a **different table structure** — you must repeat the process here too, with slightly different commands.

```bash
/data/data/com.termux/files/usr/bin/sqlite3 /data/system_de/0/accounts_de.db
```

Check the tables and account first:

```sql
.tables
SELECT _id, name, type FROM accounts;
```

`accounts_de.db` typically does **not** have `authtokens`/`extras` — instead it has `grants` and `shared_accounts`. Delete in this order:

```sql
DELETE FROM grants WHERE accounts_id = 1;
DELETE FROM shared_accounts WHERE account_id = 1;
DELETE FROM accounts WHERE _id = 1;
```

> [!TIP]
> If a `DELETE` command gives you `no such table`, that table simply doesn't exist in this database — just skip that one line and move to the next. Always trust what `.tables` showed you over this guide's example, since table names can vary slightly by Android build.

Verify:

```sql
SELECT * FROM accounts;
```

Exit:

```sql
.quit
```

Back to your shell:

```bash
exit
```

---

## 5️⃣ Step 5 — Clear Play Store & Play Services Cache

The work profile often leaves sync data cached inside Play Store (and sometimes Play Services). Clear it to finish the cleanup:

```bash
su
pm clear com.android.vending
```

**Optional — deeper clean:**

```bash
pm clear com.google.android.gms
```

> [!CAUTION]
> Clearing `com.google.android.gms` signs you **out of every Google account** on the device, including your personal one. Only run this if you're prepared to log back in afterward.

---

## 6️⃣ Step 6 — Final Reboot & Verification

```bash
exit
reboot
```

Once the device restarts, open Termux and run the full checklist:

```bash
su
dpm list-owners
pm list packages | grep clouddpc
```

### ✅ Success Checklist

- [ ] `dpm list-owners` → prints nothing (no Device Owner)
- [ ] `pm list packages | grep clouddpc` → prints nothing (app removed) or shows it disabled
- [ ] Settings → Accounts opens normally, no crash, no reset prompt
- [ ] No "work" account visible anywhere in Settings
- [ ] Play Store opens and asks you to sign in fresh (expected after `pm clear`)

---

## 🗄️ Backups

Everything you backed up during this process lives here:

```
/sdcard/dpm_backup/
├── device_owner_2.xml
├── device_policies.xml
├── device_policy_state.xml
├── accounts_ce.db
└── accounts_de.db
```

Keep this folder until you're 100% sure your device is stable and working normally. You can delete it afterward.

---

## ❓ FAQ

<details>
<summary><b>Why does <code>dpm remove-active-admin</code> fail?</b></summary>
<br>
Because the app isn't just a "device admin" — it's set as full <b>Device Owner</b>, which Android only allows to be removed through this command on test/debug builds. On a normal production ROM, you have to clear the ownership record directly from <code>/data/system</code>, which is what Step 2 does.
</details>

<details>
<summary><b>Why do I need two separate SQLite databases (ce and de)?</b></summary>
<br>
Android keeps two copies of account data for security reasons:
<ul>
<li><code>accounts_ce.db</code> — Credential Encrypted, only accessible after you unlock your device</li>
<li><code>accounts_de.db</code> — Device Encrypted, accessible even before unlock (e.g. for early boot services)</li>
</ul>
The work account can exist in both, so both need cleaning.
</details>

<details>
<summary><b>I got "sqlite3: inaccessible or not found" — what now?</b></summary>
<br>
This happens because the root (<code>su</code>) shell doesn't inherit Termux's PATH. Always use the full path:

```bash
/data/data/com.termux/files/usr/bin/sqlite3 <db-path>
```
</details>

<details>
<summary><b>What if the Device Owner comes back after reboot?</b></summary>
<br>
This can happen if the device re-checks in with an MDM server over the internet and gets re-enrolled automatically (common when the account shows "Affiliated" status). If this happens, turn on Airplane Mode right after reboot, run Steps 3–5 while offline, then reconnect to the internet.
</details>

---

<div align="center">

### 🛡️ Use responsibly — only on devices you own and control.

</div>
