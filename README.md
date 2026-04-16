# HDD I/O Error Data Recovery Guide

> Tested Environment: WDC WD40PURZ-85TTDY0 4TB, 1 pending sector, Ubuntu 24.04 LTS Live USB, ddrescue 1.27, TestDisk 7.1

**Languages / 語言 / 言語:** [English](README.md) | [繁體中文](docs/zh-TW/README.md) | [日本語](docs/ja/README.md)

---

## 📋 Use Cases: Read This First

| Situation | Applicable | Notes |
|-----------|------------|-------|
| HDD shows I/O device error, but Windows/BIOS can still detect it | ✅ Yes | Main use case of this guide |
| Few bad sectors (SMART C5 > 0) | ✅ Yes | The earlier you act, the higher the success rate |
| Partition table corrupted (GPT damaged) | ✅ Yes | Restore with sgdisk --restore |
| Windows cannot mount but BIOS detects it | ✅ Yes | Linux has higher tolerance for mounting |
| SMART shows large increase in C6 (Uncorrectable sectors) | ⚠️ Caution | Act fast, drive may deteriorate rapidly |
| Drive was dropped or making clicking sounds | ⚠️ Caution | Can try, but head damage requires cleanroom |
| PCB burned out, completely undetectable | ❌ Not applicable | Requires hardware-level repair |
| Head damaged, persistent clicking on boot | ❌ Not applicable | Cleanroom head replacement needed, ~$150–$600 USD |

**Core principle: As long as the drive can still be detected, there is a chance to recover data.**

---

## 🚨 Golden Rules: Read Before You Start

The drive is on the edge of failure. Every power cycle and every write operation consumes what little life it has left. Remember these before you begin:

**1. Stop all writes to the failing drive immediately**  
No downloads, no installs, no automatic syncing. Any write can overwrite your data or accelerate head failure.

**2. Do not let Windows auto-repair**  
Windows may attempt to run chkdsk when it detects a faulty drive. This is dangerous before data is backed up. If Windows shows a repair prompt, cancel it or unplug the drive.

**3. Do not repeatedly power cycle for testing**  
Every boot forces the heads to reposition and the motor to spin up — extra physical stress on an already-damaged drive. Plan your steps before you start.

**4. Treat the failing drive as read-only until data is safely copied**  
No repair, formatting, or partition operations until data is safely on another drive. The only exception is Step 6B partition table restore, and only after imaging is complete or confirmed impossible.

**5. Time is data**  
Bad sectors tend to spread over time. The moment you discover the problem is the golden recovery window. Don't put it off.

---

## 🗂️ Background: What Happened in This Case

This guide is based on a real recovery incident.

**What happened:**  
A WD 4TB Purple surveillance drive (WD40PURZ) with ~7,800 hours of use suddenly showed "The request could not be performed because of an I/O device error." Windows File Explorer could not access it, but Device Manager still detected it. CrystalDiskInfo showed only 1 pending sector (C5 = 1), all other values normal.

**Result:**  
Approximately 2–3TB of video and photo data was fully recovered.

---

## 🛠️ Equipment

### What Was Actually Used

| Equipment | Purpose | Notes |
|-----------|---------|-------|
| Failing drive (WD 4TB) | Recovery source | SATA 3.5" HDD |
| Healthy drive (WD 4TB) | Store image file and final data | Same capacity as failing drive — caused image truncation (see FAQ) |
| 28GB USB flash drive | Ubuntu Live USB boot drive | Also occupied by the system, unusable for other purposes |
| Another USB flash drive (any capacity) | Partition table backup | Only a few KB needed, but must be separate |
| Another computer | Create Live USB | Not recommended to use the same computer as the failing drive |

### Recommended Setup (Avoid the Pitfalls)

| Equipment | Why |
|-----------|-----|
| Target drive capacity **at least 10% larger than failing drive** | Same capacity causes image truncation at the end, cutting off the GPT backup partition table |
| Extra small USB flash drive (any capacity) | Dedicated to partition table backup, separate from Live USB |
| SATA-to-USB enclosure or PC with multiple SATA ports | Need to connect failing drive, target drive, and Live USB simultaneously |

---

## Part 1 — Assess the Situation

### Method A: Minor Damage on Windows (Try This First)

If the drive is still visible in Windows, first try copying data directly.

**Check SMART status:**

Download and run [CrystalDiskInfo](https://crystalmark.info/en/software/crystaldiskinfo/), focus on:

| Attribute | Notes |
|-----------|-------|
| C5 Current Pending Sector | > 0 means problem sectors exist — the main cause of I/O errors |
| C4 Reallocation Event Count | Number of confirmed bad sectors |
| C6 Uncorrectable Sector Count | > 0 means serious damage |
| 01 Raw Read Error Rate | Seagate drives normally show large values here; WD should be near 0 |

**Identify drive by serial number (when multiple same-model drives exist):**

```powershell
wmic diskdrive get index,model,serialnumber
```

**Try copying with Robocopy (suitable for minor damage):**

Robocopy is built into Windows — no installation needed:

```cmd
robocopy D:\ E:\backup_D\ /E /R:1 /W:1 /LOG:C:\rescue_log.txt
```

| Flag | Description |
|------|-------------|
| `/E` | Include all subdirectories |
| `/R:1` | Retry only once on error, then skip |
| `/W:1` | Wait 1 second between retries |
| `/LOG` | Log all operations — search for ERROR afterward |

After completion, open `C:\rescue_log.txt` in Notepad and search for `ERROR` to see which files failed.

> ⚠️ If Robocopy immediately fails with `Error 1117 (0x0000045D) Scanning Source Directory`, the bad sector is in a critical location (root directory or MFT). Skip to Method B.

> ⚠️ **Never let Windows run chkdsk on the failing drive** before data is safely backed up. chkdsk can worsen the damage. Only run it after data is fully copied to another drive.

---

### Method B: Linux Recovery (Severe Damage or Windows Cannot Mount)

If Robocopy fails, or Windows can no longer mount the drive after a reboot, continue below.

---

## Part 2 — Create Live USB and Boot

### 1. Create Live USB (on another computer)

1. Download [Ubuntu 24.04 LTS ISO](https://ubuntu.com/download/desktop)
2. Download [Rufus](https://rufus.ie/)
3. Insert USB drive, open Rufus
4. Select the ISO file
5. **⚠️ Important: Set "Image option" to "DD Image mode"** (not the default ISO mode) — ISO mode may produce a non-bootable USB
6. Click Start

### 2. Connect All Drives

Connect simultaneously:
- Failing drive (source)
- Target drive (destination)
- Backup USB flash drive

### 3. Boot from USB

Enter BIOS and select USB as boot device. When Ubuntu loads, choose **"Try Ubuntu"** (do not install).

Once on the desktop, disable sleep immediately: Settings → Power → set everything to **Never**.

---

## Part 3 — Execute Recovery

> ⚠️ **The order of the following steps is critical. Do not skip or reorder.**

### Step 1: Install Required Tools

```bash
sudo apt update && sudo apt install gddrescue gdisk smartmontools -y
```

### Step 2: Identify Drive Device Names

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL
```

Match serial numbers to device names, for example:
- Failing drive → `/dev/sdb`
- Target drive → `/dev/sda`
- Backup USB → `/dev/sdd`

> ⚠️ Replace `/dev/sdb`, `/dev/sda`, etc. with your actual device names. **Wrong device names can cause data to be overwritten.**

### Step 3: Back Up the Partition Table Immediately ⭐ Most Critical Step

> ⚠️ Do this before anything else. If the image gets truncated later due to insufficient space, this backup is what saves you.

Mount the backup USB:

```bash
sudo mkdir /mnt/usb
sudo mount /dev/sdd1 /mnt/usb
```

> If the USB is brand new and unformatted: `sudo mkfs.fat -F32 /dev/sdd1`, then mount again.

Back up partition table and GPT headers:

```bash
# Full partition table backup
sudo sgdisk --backup=/mnt/usb/partition_table.bak /dev/sdb

# GPT header (beginning of disk, contains primary partition table)
sudo dd if=/dev/sdb of=/mnt/usb/gpt_head.bin bs=1M count=1

# GPT footer (end of disk, contains backup partition table)
sudo dd if=/dev/sdb bs=512 skip=$(( $(sudo blockdev --getsz /dev/sdb) - 33 )) count=33 of=/mnt/usb/gpt_tail.bin
```

Confirm all three files exist:

```bash
ls -lh /mnt/usb
```

You should see `partition_table.bak`, `gpt_head.bin`, `gpt_tail.bin` — combined less than 2MB.

### Step 4: Mount Target Drive

```bash
sudo mkdir /mnt/edata
sudo mount /dev/sda2 /mnt/edata
```

### Step 4.5: Assess Bad Sector Severity with smartctl

> This step is read-only — completely safe for the failing drive.  
> Purpose: Spend a few minutes assessing bad sector severity before committing 6–15 hours to ddrescue, so you can decide in advance whether to get a larger target drive.

Check current SMART values:

```bash
sudo smartctl -A /dev/sdb
```

Focus on the `RAW_VALUE` column for these three attributes:

| Attribute | ID | Notes |
|-----------|----|-------|
| Current_Pending_Sector | C5 | Suspect sectors awaiting decision; > 0 is the cause of I/O errors |
| Offline_Uncorrectable | C6 | Confirmed unrecoverable bad sectors; > 0 means serious damage |
| Reallocated_Sector_Ct | 05 | Cumulative relocated sectors; increasing over time means drive is deteriorating |

Run SMART short test (~2 minutes):

```bash
sudo smartctl -t short /dev/sdb
sleep 120
sudo smartctl -l selftest /dev/sdb
```

**Decision table:**

| Result | Recommended Strategy |
|--------|---------------------|
| C5=0, C6=0, short test passed | Run ddrescue directly, low risk → Step 6A |
| C5 single digits, C6=0 | Run ddrescue, ensure target drive has enough space → Step 6A |
| C5 or C6 in the tens or more | More bad sectors; ddrescue will take longer. Without a large enough target drive, consider Step 6B |
| short test failed with `Completed: read failure` | Serious damage. Get a larger target drive before running ddrescue, otherwise image truncation risk is high |

> If you don't have a large enough target drive and bad sectors are minimal, you can skip directly to Step 6B — mount the original drive directly and copy without imaging.  
> **However, this means no image backup. If the drive dies mid-copy, data cannot be recovered. Assess the risk yourself.**

### Step 5: Create Disk Image with ddrescue

> ⚠️ **Confirm target drive has more free space than the total size of the failing drive before running:**
> ```bash
> df -h /mnt/edata
> ```
> If space is insufficient, **do not run ddrescue**. Skip directly to Step 6B.

```bash
sudo ddrescue -d -r1 /dev/sdb /mnt/edata/rescue.img /mnt/edata/rescue.log
```

| Flag | Description |
|------|-------------|
| `-d` | Direct read, bypasses system cache |
| `-r1` | Retry bad sectors only once before skipping |

Normal ddrescue progress indicators:
- `error rate: 0 B/s` — no bad sectors encountered yet
- `bad areas: 0` — no damaged regions found
- `average rate: 30–80 MB/s` — normal SATA HDD speed

Expected duration: 6–15 hours depending on drive size and bad sector count. If interrupted, re-running the same command resumes from where it left off (via rescue.log).

After ddrescue completes, check image quality before deciding which path to take:

```bash
cat /mnt/edata/rescue.log | grep -E "error|bad"
```

---

### Step 6A: No Errors → Mount Image File (Safer)

**When to use:** ddrescue completed with `read errors: 0` and `bad areas: 0`, and the target drive had enough space for a complete image.

> ⚠️ **Before proceeding, verify image integrity:**
> ```bash
> sudo gdisk -l /mnt/edata/rescue.img
> ```
> If you see `GPT: damaged` or the correct partition is missing, the image tail was truncated. **Switch to Step 6B.**

Get the partition start sector from gdisk output (run the same command above), find the `Basic data partition` (Type 0700) row, note the `Start` value (e.g., 32768), then calculate offset:

```
offset = Start sector × 512
Example: 32768 × 512 = 16777216
```

Mount the image:

```bash
sudo mkdir /mnt/rescue
sudo mount -o loop,ro,offset=16777216 /mnt/edata/rescue.img /mnt/rescue
ls /mnt/rescue
```

If you can see your folder structure, proceed to Step 7A.

---

### Step 6B: Errors Present or Insufficient Space → Restore Partition Table and Mount Original Drive

**When to use:** ddrescue had `read errors` or `bad areas`, or target drive space was insufficient causing image truncation, making the image unmountable.

> ⚠️ **This step writes directly to the failing drive.** Although it only modifies the partition table and does not touch data, there is a small risk of worsening damage.  
> Prerequisite: Step 3 partition table backup must be complete before proceeding.  
> If Step 3 was not done, **go back and do it now.**

Restore the GPT from backup:

```bash
sudo sgdisk --restore=/mnt/usb/partition_table.bak /dev/sdb
```

Mount the data partition in read-only mode:

```bash
sudo mkdir /mnt/rescue
sudo mount -o ro /dev/sdb2 /mnt/rescue
ls /mnt/rescue
```

If you can see your folder structure, proceed to Step 7B.

---

### Step 7A: Copy Data from Image to Target Drive

> ⚠️ **Confirm the target drive has enough space for both the image file and the extracted data:**
> ```bash
> df -h /mnt/edata
> du -sh /mnt/rescue
> ```
> If space is tight, delete the image first (see Step 7B for deletion procedure).

```bash
sudo cp -av /mnt/rescue/* /mnt/edata/
```

`-av` shows each file being copied. When done, the terminal returns to the `ubuntu@ubuntu:~$` prompt.

---

### Step 7B: Copy Data from Original Drive to Target Drive

> ⚠️ **Confirm Step 3 partition table backup is complete before deleting rescue.img.** The image cannot be recovered after deletion, but the partition table backup on the USB can still be used for future restoration.

Delete image to free space:

```bash
sudo rm /mnt/edata/rescue.img
```

Confirm space is freed:

```bash
df -h /mnt/edata
```

Copy data from original drive to target:

```bash
sudo cp -av /mnt/rescue/* /mnt/edata/
```

When done, the terminal returns to the `ubuntu@ubuntu:~$` prompt.

---

## Part 4 — Wrap Up and Verify

### Verify Recovery

```bash
df -h /mnt/edata
ls /mnt/edata
```

Confirm that capacity has increased appropriately and folders are present.

### Back in Windows

- The target (healthy) drive can be plugged back in and read normally — no extra steps needed
- If Windows asks to run chkdsk on the failing drive after re-inserting it, **it is now safe to let it run** — data has been safely backed up
- Retire the failing drive. Do not use it as primary storage again.

---

## Appendix: Prevention

```
3-2-1 Backup Rule:
  3 copies of data
  2 different storage media (e.g., HDD + cloud)
  1 offsite copy
```

Monitor SMART regularly with CrystalDiskInfo. The moment C5 > 0, back up immediately.

---

## Appendix: FAQ

**Q: Which mode should I use in Rufus?**  
A: Select "DD Image mode," not the default ISO mode. ISO mode may produce a non-bootable Live USB.

**Q: Robocopy fails immediately with an I/O error even during the scan?**  
A: The bad sector is in a critical location (root directory or MFT). Windows tools cannot work around this. Use Linux ddrescue.

**Q: ddrescue is running extremely slowly, only a few KB/s?**  
A: This is normal when retrying bad sectors. `-r1` limits retries to once per sector before skipping. As long as overall progress is moving, it will complete.

**Q: Image mount fails with `NTFS signature is missing`?**  
A: The image was likely truncated (insufficient target drive space), cutting off the GPT backup partition table. If Step 3 was done, use `sgdisk --restore` to restore the partition table on the original drive and mount it directly.

**Q: TestDisk finds many strange partitions, none of which are my data?**  
A: When GPT is corrupted, TestDisk scan results contain many false positives. Restoring from the partition table backup is more reliable than TestDisk scanning.

**Q: What if my target drive is the same size as the failing drive?**  
A: The last few hundred MB of the image will be truncated, cutting off the GPT backup partition table and causing mount failure. The fix: back up the partition table first (Step 3), then use `sgdisk --restore` on the original drive — no dependency on the image's partition table.

**Q: The computer went to sleep during recovery — what now?**  
A: Both ddrescue and cp will be interrupted. Set Settings → Power → everything to Never before starting. ddrescue can resume from the same point by re-running the same command. cp must restart from the beginning.

---

*Based on a real recovery incident. PRs and corrections welcome.*
