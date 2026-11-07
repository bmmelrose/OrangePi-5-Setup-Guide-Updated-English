# Orange Pi 5 Setup Guide - MicroSD Boot with WiFi/Bluetooth NVME

*Tested and validated as of 7 November 2026, using Debian 13 minimal via Armbian and KDE Plasma using Wayland; and a System76 Laptop running Garuda Linux*

**Recommended path:** flash Armbian 26.8 (Ubuntu 26.04 "Resolute", KDE Plasma, **vendor 6.1 kernel**) to a microSD card you have tested first, boot it, then turn on the GPU, the AP6275P module and USB-C DisplayPort with a few overlay settings. The vendor kernel is the only Armbian kernel whose Orange Pi 5 device tree drives DisplayPort over USB-C, so it is the default here. Every step ends with a check, so you know it worked before moving on.

Facts were checked on 3 Nov 2026 against Armbian's download page, Armbian Imager v2.5.0, Orange Pi's official wiki and tools, and Armbian forum threads from 2026. Linked in Sources at the end.

## 1. Before you start

### Confirm it is a plain Orange Pi 5

Images for the 5B, 5 Plus, 5 Pro, 5 Max and 5 Ultra are different and will not boot properly on a plain Orange Pi 5. Check the board itself:

- The silkscreen next to the Orange Pi logo reads **OPi 5 V1.x** (for example V1.2 or V1.3.2). Write the revision down. It matters for the M.2 slot in section 13.
- One HDMI port, one Gigabit Ethernet port, no Wi-Fi antenna connectors on the board, no eMMC socket.
- The SoC reads **RK3588S**.
- Every Armbian file for this board has `Orangepi5_` in its name. A name containing `Orangepi5b`, `Orangepi5-plus`, `Orangepi5pro`, `Orangepi5-max` or `Orangepi5-ultra` is for another board.

### What you need

| Item | Spec | Why |
| --- | --- | --- |
| Power supply | USB-C, **5 V / 4 A** (Orange Pi's official spec) | Weak supplies are a leading cause of no-boot. A laptop USB-PD charger often gives only 5 V / 3 A. |
| microSD card | 256 GB C10 U3 V30 A1, after the tests in section 3 (smaller sizes can work)| Holds the OS |
| USB card reader | USB 3, or the SD slot | Flashing |
| HDMI cable and monitor | Any | First boot. Use HDMI first, USB-C display comes later. |
| Ethernet cable | Cat5e or better | Updates before Wi-Fi is set up |
| USB keyboard and mouse | Any | First-run wizard |
| Heatsink or fan | Any RK3588S heatsink; the board has a 5 V fan header | The SoC throttles without one |
| USB-A to USB-C data cable | Must carry data, not charge-only | Only for MaskROM recovery (section 11). A USB-C to USB-C data cable often works too (section 11, B) |
| Optional | 3.3 V USB-to-TTL adapter, USB-C to DisplayPort cable, AP6275P module | Sections 12, 14 and 13 |

### Port and part map

Positions follow Orange Pi's [labelled board diagram](http://www.orangepi.org/img/pi5-fix/pi5-21.png) (top view, Ethernet on the right).

| Part | Where | Notes |
| --- | --- | --- |
| USB-C **PWR IN** | Bottom-left corner | Power only, 5 V / 4 A |
| USB-C **TYPE-C** (USB 3.1) | Bottom edge, right of HDMI | USB 3.1 data and DisplayPort 1.4 alt mode. Also the port RKDevTool talks to in MaskROM mode. |
| HDMI 2.1 | Bottom edge, centre | Up to 8K60, eARC |
| USB 3.0 + USB 2.0 | Stacked port, top right | USB 3.0 is the upper one |
| USB 2.0 | Right edge, below Ethernet | Orange Pi marks it "shared with Type-C": it shares USB 2.0 lines with the USB-C port |
| Gigabit Ethernet | Right edge | YT8531C PHY on V1.3.2 boards |
| 3.5 mm audio | Bottom edge, left | Headphones and mic in (ES8388 codec); onboard mic on the left edge |
| 26-pin header | Top edge | GPIO, UART, I2C, SPI, PWM, CAN |
| Debug UART | 3-pin header right of the 26-pin header, marked TX RX GND | 3.3 V, 1,500,000 baud (section 12) |
| LEDs | Top edge, right of the UART header | **Red** = power, always on, hardware-controlled. **Green** = status, blinks once the Linux kernel is running. |
| MaskROM key | Bottom-left, beside the audio jack, marked MASKROM | Forces USB recovery mode (section 11) |
| On/off key | Left edge, above the mic | Power button |
| Recovery key | Top-left, beside the 26-pin header | Android recovery; not needed for Linux |
| M.2 M-key socket | Underside | PCIe 2.0 x1. Takes an NVMe SSD **or** the AP6275P Wi-Fi module, not both. |
| 16 MB SPI NOR flash | Onboard chip | Can hold a bootloader. A stale one here is the top cause of a red-LED-only board. |
| microSD slot | Top-left edge | Boot device for this guide |
| MIPI camera and LCD connectors | Various | Need overlays; not covered here |

## 2. Choose the OS

Use **Armbian 26.8, Ubuntu 26.04 Resolute, KDE Plasma, vendor 6.1.115 kernel**. It is the only option that ships Plasma and supports USB-C DisplayPort, the AP6275P module and the NPU on this board today. Armbian rates the Orange Pi 5 as "Standard" support, with an active maintainer.

| Option | Desktop | Kernel | USB-C DisplayPort | AP6275P | Verdict |
| --- | --- | --- | --- | --- | --- |
| [Armbian](https://www.armbian.com/orangepi-5/) Ubuntu 26.04, KDE Plasma, **vendor** | Plasma, preinstalled | 6.1.115 (Rockchip BSP) | Yes. The vendor device tree enables DP alt mode. | Yes, `orangepi-5-ap6275p` overlay | **Use this** |
| Armbian Ubuntu 26.04, KDE Plasma, current | Plasma, preinstalled | 6.18.43 (mainline) | No. Alt-mode patches were still pending upstream as of Apr 2026. | Yes, overlay added Jul 2026 | Only if you never need a USB-C display |
| Armbian Debian 13 Trixie Minimal, vendor | Add Plasma yourself (section 8) | 6.1.115 | Yes | Yes | If you prefer Debian |
| Orange Pi official Ubuntu 22.04 / Debian | GNOME or XFCE | Orange Pi BSP (5.10 / 6.1) | Yes, per Orange Pi's wiki | Yes, through `orangepi-config` | Older base. Keep it as a known-good test image (section 11). |
| Orange Pi OS (Arch) | See its download folder | Orange Pi BSP | Not checked | Not checked | Official Arch-based option, rarely updated |
| BredOS (Arch-based, RK3588-focused) | Check bredos.org | Varies | Not checked | Not checked | Listed by Orange Pi as a third-party image. Its site returned a 502 error on 3 Nov 2026, so its status is unconfirmed. |
| Ubuntu Rockchip (Joshua Riek) | GNOME | 6.1 | Not checked | Yes | Archived on 29 Apr 2026, no more updates. Avoid for a new install. |

Why not Arch, given you run Garuda: Arch Linux ARM ships no kernel or U-Boot for the RK3588S, so every Arch-based image relies on a third party and lags on hardware support. If you want rolling-release software later, Flatpak on Armbian gets you current desktop apps without giving up the vendor kernel.

The trade-off of the vendor kernel: it is Rockchip's 6.1 tree, not mainline. GPU acceleration needs one overlay (section 8), and one forum report from Oct 2026 says 5 GHz Wi-Fi on the AP6275P fails on Armbian 26.x vendor builds (section 13).

## 3. Check the microSD card

On paper Armbian's bar: Class 10, U3/V30 and A1 or better. The risk is the individual card (a fake or failing unit), not the rating. Armbian says over 95% of boot and stability problems come from the card or the power supply. Test it before you flash. Both tests below erase the card.

### Test 1: real capacity and data integrity

**Garuda (fastest, about 2 minutes):**

```bash
sudo pacman -S f3
lsblk -o NAME,SIZE,MODEL,TRAN    # find the card, e.g. sdb (about 238G)
sudo umount /dev/sdX?* 2>/dev/null    # unmount it first
sudo f3probe --destructive --time-ops /dev/sdX
```

Pass: `Good news: The device '/dev/sdX' is the real thing.` Any other verdict means a fake or faulty card.

**Garuda or macOS (full test, 2 to 4 hours for 256 GB):** format the card as exFAT with the label CARD and mount it (on Garuda, the "Prepare the card" commands under Test 2 do both; on macOS, erase it in Disk Utility as ExFAT named CARD), then fill it and read it back. On macOS install with `brew install f3`; `f3probe` is Linux-only, but `f3write` and `f3read` work.

```bash
f3write /run/media/$USER/CARD     # macOS: /Volumes/CARD
f3read  /run/media/$USER/CARD
```

Pass: `f3read` reports 0 corrupted, 0 changed and 0 overwritten sectors. `f3write` also prints the average write speed.

**Windows 11:** run H2testw (from heise.de) on the card's drive letter with "all available space", then **Write + Verify**. Pass: `Test finished without errors.`

### Test 2: speed

Use KDiskMark on Garuda. On Windows, CrystalDiskMark has the same layout, defaults and row names, so the same steps and limits apply. KDiskMark tests a folder on a mounted filesystem, not a raw device. Test 1 leaves the card unreadable (f3probe) or full (f3write), so give it a fresh exFAT partition first.

**Prepare the card on Garuda (erases it).** Replace `sdX` with the card from `lsblk`. In a built-in slot the card is `mmcblk0` and its partition is `mmcblk0p1`.

```bash
sudo pacman -S --needed kdiskmark exfatprogs parted
lsblk -o NAME,SIZE,MODEL,TRAN
sudo umount /dev/sdX?* 2>/dev/null
sudo wipefs -a /dev/sdX
sudo parted -s /dev/sdX mklabel msdos mkpart primary 1MiB 100%
sudo mkfs.exfat -L CARD /dev/sdX1
udisksctl mount -b /dev/sdX1
df -h /run/media/$USER/CARD
```

Pass: `df` shows about 238G free at `/run/media/<your user name>/CARD`. If `udisksctl` says the card is already mounted, KDE mounted it for you, which is fine.

**Run the test:**

1. Open KDiskMark from the app menu.
2. In the top row, leave the runs at **5** and the size at **1 GiB** (the defaults). Set the storage drop-down to `/run/media/<your user name>/CARD`. Check it is not your home folder or another laptop drive, or you will measure the laptop's SSD. If the card is not listed, choose **Add a directory** and pick the card's folder.
3. Leave **Profile → Default** selected. Leave **Settings → Flush Pagecache** and **Settings → Use O\_DIRECT** ticked (both are on by default), so the numbers come from the card and not from RAM.
4. Click **All** and enter your password when asked. KDiskMark gets administrator rights through polkit. The button reads **Stop** while the test runs.
5. Wait about 5 to 10 minutes, and leave the card and reader alone until the button reads **All** again.
6. Optional: **File → Save** writes a text report, which is handy if you ask for help on the Armbian forum. Test 3 wipes the card afterwards, so there is nothing to delete.

**Read the results.** Each row shows a Read and a Write value. Compare these three with the table below:

- **SEQ1M Q1T1**, Write column → Sequential write
- **RND4K Q1T1**, Read column → 4K random read, queue depth 1
- **RND4K Q1T1**, Write column → 4K random write, queue depth 1

The Q8T1 and Q32T1 rows use deeper queues and are not part of the pass mark. To compare with the A1 rating directly, switch the units drop-down from **MB/s** to **IOPS**: RND4K Q1T1 should be near 1,500 read and 500 write.

| Measure | Minimum for this card's rating | Replace the card if below |
| --- | --- | --- |
| Sequential write | 30 MB/s (V30) | 10 MB/s |
| 4K random read, queue depth 1 | about 6 MB/s (A1 = 1,500 IOPS) | 3 MB/s |
| 4K random write, queue depth 1 | about 2 MB/s (A1 = 500 IOPS) | 1 MB/s |

**If the numbers look wrong:**

| What you see | Likely cause | Fix |
| --- | --- | --- |
| Above about 200 MB/s, or results in GB/s | The test ran on a laptop drive, or hit the RAM cache | Re-select the card's folder. Keep Flush Pagecache and Use O\_DIRECT ticked |
| Read and write both stop near 40 MB/s | USB 2 card reader or port | Run `lsusb -t`: 480M on the reader's line means USB 2, 5000M means USB 3. Use a USB 3 reader in a USB 3 port, or the laptop's own slot |
| No password prompt, or "Benchmark Failed" | No polkit agent running | KDE and GNOME include one. On other desktops, install and start polkit-kde-agent or polkit-gnome |
| Card missing from the storage list | Not mounted, or mounted read-only | Mount it with `udisksctl`. If it mounts read-only, check the lock switch on the SD adapter; if that is not it, the card may be failing |
| Writes much slower when you run All a second time | The card's fast cache filled, or the card is failing | Let the card rest a few minutes and run All again. If SEQ1M Q1T1 write stays below the replace column, replace the card |

### Test 3: reset the card

Tests 1 and 2 leave test files and a home-made partition on the card. This step uses the SD Association's official **SD Memory Card Formatter** to put back the file system and alignment the card was designed for. Armbian recommends it for any card that has been used before. Flashing overwrites the card anyway, so this is a clean start rather than a hard requirement.

**Which option:** Quick format is enough. On Garuda, add `--discard`, which also tells the card that every block is free; that can help its write speed when the reader supports it. Use Overwrite format only to wipe old personal data from a card used elsewhere. It rewrites the whole card and takes roughly 1 to 3 hours for 256 GB, so keep the computer plugged in and stop it from sleeping.

**Garuda (SD Memory Card Formatter for Linux 1.0.3, a command-line tool):**

1. On sdcard.org go to Downloads → SD Memory Card Formatter for Linux → **Linux\_x86\_64 Download**, accept the licence and download it. SDA lists Arch Linux as tested.
2. Extract it into a folder in your home directory. Don't run it from a USB stick: SDA notes that removable drives are often mounted without permission to run programs.
3. Open a terminal in the folder that holds `format_sd` and run the commands below. Give the tool the whole card (`/dev/sdX`, or `/dev/mmcblk0` in a built-in slot), never a partition such as `sdX1`.

```bash
chmod +x format_sd
sudo ./format_sd --about
lsblk -o NAME,SIZE,MODEL,TRAN
sudo umount /dev/sdX?* 2>/dev/null
sudo ./format_sd -l OPI5 --discard /dev/sdX
lsblk -f /dev/sdX
```

Pass: the tool ends with `Formatted volume "/dev/sdX" successfully as exFAT!`, and `lsblk -f` shows one exfat partition labelled OPI5 of about 238G. If the reader doesn't support discard, the tool does a quick format instead, which is fine. For a full wipe, use `--overwrite` in place of `--discard`.

**Windows 11:**

1. Download **SD Memory Card Formatter for Windows** (5.0.4) from sdcard.org, accept the licence, run the installer and click **Yes** at the User Account Control prompt. Orange Pi's official tools folder has a copy too, but it may be older.
2. Insert the card and open **SD Card Formatter**. Click **Yes** when it asks for administrator rights.
3. Pick the card's drive letter in the card list at the top. Click **Refresh** if it isn't there.
4. Check that **Card type** reads SDXC and **Size** is about 238 GB (Windows counts in units of 1,024). A much smaller size means the wrong drive or a fake card.
5. Choose **Quick format** and type a volume label such as OPI5. Leave **CHS format size adjustment** alone; it only applies to cards of 8 GB or less.
6. Click **Format**, then **Continue**.

Pass: the progress line finishes, and the card's Properties in File Explorer show exFAT and about 238 GB.

**macOS (MacBook Pro):**

1. Download **SD Memory Card Formatter for Mac** from sdcard.org and install it. On Apple silicon it may ask to install Rosetta; accept.
2. Put the microSD card in a full-size SD adapter with the lock switch up (unlocked), and insert it into the MacBook Pro's SDXC slot.
3. Open **SD Card Formatter** from Applications and enter your password when asked. If macOS asks whether it may access a removable volume, click **Allow**.
4. Pick the card in the list. **Size** should read about 255 GB; macOS counts in units of 1,000, so the same card looks bigger than on Windows.
5. Choose **Quick format**, type OPI5 as the volume label, click **Format**, then **Continue**.

Pass: in Terminal, `diskutil info /Volumes/OPI5 | grep -E "Personality|Disk Size"` shows ExFAT and about 255 GB.

SDA lists macOS 11 to 15 as tested. If the app won't open or can't see the card on a newer macOS, do this step on Windows or Garuda instead.

**If the reset fails:**

| Problem | Fix |
| --- | --- |
| The card is not in the list | Click Refresh, or take the card out and put it back. On Windows the card needs a drive letter; if it has none, use the Garuda steps |
| Write-protected or read-only error | Slide the adapter's lock switch up, toward the contacts. MacBook slots are sensitive to loose switches, so try another adapter or a USB reader |
| Garuda: `Permission denied` | Run `chmod +x format_sd`, and keep the tool in your home folder, not on a USB stick |
| Garuda: device busy | Unmount every partition first (`sudo umount /dev/sdX?*`) and close any Dolphin window showing the card |
| Size far below 238 GB, or the format stops with an error | Repeat Test 1. If it fails again, replace the card |

The card is now ready for section 4 (download the image) and section 5 (flash it).

### Replace the card if

- f3 or H2testw reports a fake capacity or any error
- Speeds fall below the replace column above
- The board will not boot from it, but boots the same image from a known-good card

### Recommended replacements

Buy from the maker's store or a large retailer; counterfeit cards are common on marketplaces. 64 to 128 GB is plenty for the OS. A2 is not required.

| Card | Why |
| --- | --- |
| Samsung PRO Endurance | Built for continuous writes. The best fit for an always-on desktop. |
| SanDisk High Endurance or Max Endurance | Same idea from SanDisk. Orange Pi's wiki recommends SanDisk cards. |
| Samsung PRO Plus or EVO Plus (U3, V30, A2) | Fast, widely used on single-board computers |
| SanDisk Extreme (U3, V30, A2) | Fast, widely used on single-board computers |
| Raspberry Pi official microSD (A2, V30) | Tested by Raspberry Pi for SBC workloads |

## 4. Download and verify the image

Skip this section if you flash with Armbian Imager and let it download the image: it checks the SHA-256 for you. Do it when you download by hand, or for any other flasher.

### Download

Open [armbian.com/orangepi-5](https://www.armbian.com/orangepi-5/) and confirm the page title is "Orange Pi 5". When checked, the similar address armbian.com/orange-pi-5 showed the Orange Pi 5+ page instead. Under Desktop, download the row **Ubuntu 26.04 resolute | Kde Plasma | vendor 6.1.115**, plus its **SHA** and **ASC** files, into one folder.

From a terminal instead (Garuda or macOS; `--trust-server-names` keeps the real file names):

```bash
mkdir -p ~/opi5 && cd ~/opi5
for f in Resolute_vendor_kde-plasma Resolute_vendor_kde-plasma.sha Resolute_vendor_kde-plasma.asc; do
  wget --trust-server-names "https://dl.armbian.com/orangepi5/$f"
done
ls -lh
```

You should get an `.img.xz` of about 1.5 GB whose name starts with `Armbian_` and contains `Orangepi5_resolute_vendor` and `kde-plasma`, plus matching `.sha` and `.asc` files. Install wget on macOS with `brew install wget`.

### Check the SHA-256 checksum

This proves the download is complete and uncorrupted.

| Computer | Command | Pass |
| --- | --- | --- |
| Garuda | `sha256sum -c Armbian_*.img.xz.sha` | `...img.xz: OK` |
| macOS | `shasum -a 256 -c Armbian_*.img.xz.sha` | `...img.xz: OK` |
| Windows (PowerShell) | `(Get-FileHash .\Armbian_*.img.xz -Algorithm SHA256).Hash` then `Get-Content .\Armbian_*.img.xz.sha` | The two hex strings match (case does not matter) |

### Check the GPG signature

This proves the image came from Armbian. Install GnuPG first: it is on Garuda already, `brew install gnupg` on macOS, Gpg4win on Windows.

```bash
gpg --keyserver hkp://keyserver.ubuntu.com --recv-key DF00FAF1C577104B50BF1D0093D6889F9F0E78D5
gpg --keyserver hkp://keyserver.ubuntu.com --recv-key 8CFA83D13EB2181EEF5843E41EB30FAF236099FE
# if the keyserver is unreachable:
# wget -O- https://apt.armbian.com/armbian.key | gpg --import -
gpg --verify Armbian_*.img.xz.asc Armbian_*.img.xz
```

Pass: `Good signature from "Igor Pecovnik"`, with primary key fingerprint `DF00 FAF1 C577 104B 50BF 1D00 93D6 889F 9F0E 78D5`. The warning `This key is not certified with a trusted signature!` is expected and safe to ignore. `BAD signature` means delete the file and download it again.

If either check fails twice, download from a different mirror (the torrent link on the same page works well), then check again.

## 5. Flash with Armbian Imager

Armbian Imager is Armbian's own flasher and the method Armbian recommends. It downloads the image, checks its SHA-256, writes the card, then reads every byte back to verify it. The current release is **v2.5.0 (28 Oct 2026)**. Get it from the [GitHub releases page](https://github.com/armbian/imager/releases); when checked, the imager.armbian.com landing page still showed an older v1.2.8.

### Install it

**Garuda (best choice of the three: Linux reads the ext4 card natively for the checks in this guide).**

```bash
cd ~/Downloads
sha256sum Armbian.Imager_2.5.0_amd64.AppImage
# must print 2555f17527964d05f6967cb5b06f63bbc13d39b88b67239264e615a51c8a2b0a
chmod +x Armbian.Imager_2.5.0_amd64.AppImage
./Armbian.Imager_2.5.0_amd64.AppImage
```

It uses UDisks2 and polkit for disk access, both standard on Garuda KDE; you will get a password prompt when it writes. If it refuses to start with a FUSE error, run `sudo pacman -S fuse2`. Versions before 2.0.3 could show a blank window; 2.5.0 has the fix.

**MacBook Pro (Apple Silicon).** Download `Armbian.Imager_2.5.0_aarch64.dmg`, open it and drag the app to Applications. The app is not signed with an Apple Developer certificate, so macOS blocks the first launch. Open it once, dismiss the warning, then go to System Settings > Privacy & Security, scroll down and click **Open Anyway**. This is needed only once. Touch ID works for the write prompt.

**Windows 11 desktop.** Download the x64 `.exe` installer from the same release (Windows builds are code-signed), install it and accept the Administrator (UAC) prompt.

### Flash the card

1. Insert the card. Unplug any other USB drives, so there is only one target to choose.
2. **Manufacturer:** Orange Pi (it may be listed as Xunlong). **Board:** Orange Pi 5. Do not pick 5B, 5 Plus, 5 Pro, 5 Max or 5 Ultra.
3. **Image:** Ubuntu 26.04 Resolute, **KDE Plasma**, kernel **vendor**, stable build.
4. **Storage:** pick the card. It shows as about 256 GB (238 GiB). The app hides system disks and accepts only a whole disk it detected itself.
5. Optional, new in 2.5.0: a first-boot profile with your user name, time zone, language, Wi-Fi and SSH keys. Skip it on the first try so the board's own setup wizard runs; that is easier to troubleshoot.
6. Click write and confirm the erase prompt. Wait for all four stages: download, decompress, write, **verify**. Do not cancel verification. Cancelling it still shows a success screen, which proves nothing.

To flash an image you already downloaded and checked in section 4, choose "load your own image" at step 2 and pick the `.img.xz`.

### Verify the result

- The app finishes on its success screen only after the read-back passes. A verification error means a bad card or reader: try another reader, then re-run the card tests in section 3.
- On Garuda, `lsblk -f /dev/sdX` should show one partition with an ext4 filesystem (Armbian labels it `armbi_root`), only a few GB in size. It grows to fill the card on first boot.
- On **macOS**, a dialog saying the disk is not readable is normal (macOS cannot read ext4). Click **Ignore** or **Eject**, never **Initialize**.
- On **Windows**, a prompt to format the disk is normal. Click **Cancel**, never Format.

### If Imager misbehaves

| Symptom | Fix |
| --- | --- |
| Card not listed | Re-insert it; try another reader. On Garuda check `systemctl status udisks2`. On macOS allow the app access to removable volumes in Privacy & Security. |
| Download or SHA check fails | Retry. The app caches images (5 to 100 GB, set in Settings), so a retry is fast. Otherwise download manually (section 4) and load it as your own image. |
| Verification fails | Bad card, reader or USB port. Change reader and port, then test the card with f3 or H2testw. |
| Windows asks for Administrator rights | Accept the UAC prompt; the app needs it to write to disks. |
| Need details for a bug report | Settings > Developer mode turns on detailed logs and a log viewer. |

## 6. Other ways to flash

Use these if Imager will not run, or to rule Imager out while troubleshooting. Start from the `.img.xz` you checked in section 4.

| Tool | Runs on | Takes `.img.xz` directly | Verifies the write |
| --- | --- | --- | --- |
| `dd` + `cmp` (below) | Garuda, macOS | Yes, through `xzcat` | Yes, with the `cmp` step |
| USBImager | All three | Yes | Yes, tick **Verify** |
| Raspberry Pi Imager | All three | Yes ("Use custom") | Yes, on by default |
| balenaEtcher | All three | Decompress first | Yes, on by default |
| Win32 Disk Imager | Windows | Decompress first (7-Zip) | Yes, the **Verify Only** button after writing |

balenaEtcher and Win32 Disk Imager are the two flashers Orange Pi's own wiki uses. Armbian's docs advise against Etcher because of image corruption during its decompression step, which is why the table says to decompress first.

### Garuda: dd, then compare

```bash
cd ~/opi5
lsblk -o NAME,SIZE,MODEL,TRAN          # be certain which sdX is the card
IMG=$(ls Armbian_*Orangepi5_*vendor*kde-plasma*.img.xz)
sudo umount /dev/sdX?* 2>/dev/null
xzcat "$IMG" | sudo dd of=/dev/sdX bs=4M conv=fsync oflag=direct status=progress
sync

# verify: compare the card byte-for-byte with the decompressed image
SIZE=$(xz --robot --list "$IMG" | awk '/^totals/{print $5}')
xzcat "$IMG" | sudo cmp -n "$SIZE" - /dev/sdX && echo "VERIFIED: card matches image"
```

Pass: `VERIFIED: card matches image`. A line like `differ: byte 123456` means the write failed. Reflash, then change reader or card.

### macOS: dd, then compare

```bash
brew install xz
diskutil list                          # find the card, e.g. /dev/disk4 (external, physical)
diskutil unmountDisk /dev/diskN
IMG=$(ls ~/opi5/Armbian_*Orangepi5_*vendor*kde-plasma*.img.xz)
xzcat "$IMG" | sudo dd of=/dev/rdiskN bs=4m
# press Ctrl+T while it runs to see progress
SIZE=$(xz --robot --list "$IMG" | awk '/^totals/{print $5}')
xzcat "$IMG" | sudo cmp -n "$SIZE" - /dev/diskN && echo "VERIFIED: card matches image"
diskutil eject /dev/diskN
```

Write to `/dev/rdiskN` (the raw device, much faster) but compare against `/dev/diskN`. Picking the wrong disk number erases that disk, so check `diskutil list` twice.

### Windows: Win32 Disk Imager or USBImager

1. Decompress the `.img.xz` with 7-Zip to get the `.img`.
2. In Win32 Disk Imager, pick the `.img` and the card's drive letter, click **Write**, then **Verify Only**. Pass: `Verify Successful`.
3. In USBImager, pick the `.img.xz` directly, tick **Verify**, then click Write. Pass: the status bar reports the write and verify as done with no error.

Cancel any Windows "format this disk" prompts afterwards.

## 7. First boot

### Connect in this order

1. Fit the heatsink or fan.
2. Leave the M.2 slot empty for now, and do not connect a serial adapter unless you are using one.
3. Insert the card fully into the microSD slot.
4. Connect HDMI (the HDMI port, not USB-C), Ethernet, keyboard and mouse. Turn the monitor on.
5. Connect the 5 V / 4 A supply to **PWR IN** last.

### What should happen

| Time after power-on | Expected | If not |
| --- | --- | --- |
| Immediately | Red LED on | No red LED: power supply, cable or board. Section 10. |
| 5 to 30 s | Green LED starts blinking: the kernel is running | Red LED only after 2 minutes: section 10 |
| 10 to 60 s | Boot text or the Armbian logo on HDMI | Green blinking but black screen: the board is up, see "No picture" below |
| First boot only | The root partition grows to fill the 256 GB card; the board may reboot once | Wait up to 5 minutes before deciding it failed |
| Then | Armbian's first-run wizard (on screen, or on the serial console) |  |

### First-run wizard

If you skipped the Imager profile, Armbian asks you to:

1. Log in as `root` with password `1234`, then set a new root password.
2. Choose a shell (bash is fine).
3. Create your normal user, which gets `sudo`.
4. Confirm locale and time zone (it guesses from your IP).

The desktop image then starts SDDM, the KDE login screen. Log in as your new user.

### No picture, but the green LED blinks

The board has booted; only the display is the problem. Try another HDMI cable or monitor input, and plug HDMI in before powering on. Meanwhile, reach it over the network:

```bash
# on Garuda: find the board (Armbian's default hostname is orangepi5)
ip -4 route | awk '/default/{print $3}'   # your router, e.g. 192.168.1.1
sudo nmap -sn 192.168.1.0/24 | grep -i -B2 orangepi
ssh root@<board-ip>                     # first login: password 1234
```

Or look for "orangepi5" in your router's DHCP client list. Windows 11 and macOS both have `ssh` built in.

### Update and confirm the install

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

After the reboot, check:

| Command | Pass |
| --- | --- |
| `cat /etc/armbian-release \| grep -E "BOARD=\|BRANCH=\|VERSION="` | `BOARD=orangepi5`, `BRANCH=vendor`, a 26.x version |
| `uname -r` | Contains `vendor-rk35xx`, e.g. `6.1.115-vendor-rk35xx` |
| `df -h /` | Size close to 235G (the filesystem grew to the whole card) |
| `cat /sys/firmware/devicetree/base/model; echo` | `Orange Pi 5` (or `Xunlong Orange Pi 5`) |
| `sudo dmesg --level=err,crit \| head -40` | No storage (mmc) or filesystem errors |

`sudo armbian-config` is the main settings tool from here on: kernels, overlays, desktops, networking and installing to other storage.

If the card boots from a USB card reader but not from the microSD slot, or only older images boot, you most likely have a stale bootloader in SPI flash. Section 10 fixes it.

## 8. KDE Plasma and GPU acceleration

The KDE Plasma image ships Plasma ready to use. What it may not ship is GPU acceleration: Armbian's plan to switch it on by default for RK3588 vendor desktops was closed unmerged in Jul 2026, and a 2025 test of a vendor image found software rendering (`llvmpipe`). Check, and switch on the open-source Panthor driver if needed.

### Confirm Plasma

```bash
plasmashell --version          # plasmashell 6.x
echo $XDG_SESSION_TYPE         # wayland (run this in Konsole on the desktop)
```

If you flashed a Minimal or GNOME image instead, add Plasma with `sudo armbian-config` (Software > Desktops), or directly:

```bash
sudo apt install kde-plasma-desktop sddm    # use kde-standard for the fuller app set
sudo dpkg-reconfigure sddm                  # choose sddm as the login manager
sudo reboot
```

### Check GPU acceleration

```bash
sudo apt install mesa-utils
glxinfo -B | grep -E "OpenGL renderer|OpenGL version"
```

| Renderer string | Meaning |
| --- | --- |
| `Mali-G610 (Panfrost)` | Hardware acceleration through Panthor and Mesa. Done. |
| `llvmpipe (LLVM ...)` | Software rendering on the CPU. Turn on Panthor below. |

KDE shows the same thing in System Settings > About this System, on the Graphics Processor line.

### Turn on Panthor

The vendor kernel defaults to Rockchip's proprietary Mali kernel driver, which Mesa cannot use. The `panthor-gpu` overlay swaps in the open-source Panthor driver.

```bash
sudoedit /boot/armbianEnv.txt
```

Make sure the file has exactly one of each of these lines. Overlay names go on one `overlays=` line, separated by spaces; you will add more in sections 13 and 14.

```
overlay_prefix=rockchip-rk3588
overlays=panthor-gpu
```

Save, then `sudo reboot`. The overlay file it loads is `/boot/dtb/rockchip/overlay/rockchip-rk3588-panthor-gpu.dtbo`; `ls` it if in doubt. `sudo armbian-config` can also tick overlays for you, under System > Kernel (the wording varies by version).

Verify after the reboot:

| Command | Pass |
| --- | --- |
| `glxinfo -B \| grep "OpenGL renderer"` | `Mali-G610 (Panfrost)` |
| `lsmod \| grep panthor` | A `panthor` line |
| `sudo dmesg \| grep -i panthor` | Panthor probe lines and no errors |
| `glmark2-es2-wayland` (package `glmark2-es2-wayland`) | Runs smoothly; the score is far above an llvmpipe run |

Trade-off: Panthor replaces the proprietary Mali driver, so apps built for Rockchip's `libmali` will not use the GPU. A community test matrix from Nov 2025 also reported conflicts between the Panthor path and the NPU (RKNN). For a KDE desktop, Panthor is the right choice. To undo it, remove `panthor-gpu` from the `overlays=` line and reboot.

### If Plasma misbehaves

| Symptom | Fix |
| --- | --- |
| Black screen after login, mouse cursor only | Switch to a text console with Ctrl+Alt+F3, log in, remove `panthor-gpu` from `armbianEnv.txt`, reboot. If that fixes it, run `sudo apt full-upgrade` for newer Mesa and try again. |
| "No signal" on the monitor right after login, though the login screen was fine | Plasma chose a mode the HDMI link can't carry. SSH in as your user, run `export WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/$(id -u)`, then `kscreen-doctor output.HDMI-A-1.enable output.HDMI-A-1.mode.1920x1080@60` (take the output name from `kscreen-doctor -o`). For a higher mode, try it with 8-bit color, such as `kscreen-doctor output.HDMI-A-1.maxbpc.8 output.HDMI-A-1.mode.3840x2160@60`, or step up in System Settings > Display & Monitor, which switches back by itself after about 15 seconds if the screen goes blank. 4K at 60 Hz needs a Premium High Speed HDMI cable. |
| Desktop works but slow, `llvmpipe` | Overlay line misspelled, or `overlay_prefix` missing. Check both, then reboot. |
| SDDM does not start | `sudo systemctl status sddm` and `journalctl -b -u sddm` for the reason |
| Tearing or stutter at 4K | Use 60 Hz, not 30 Hz, in System Settings > Display & Monitor. Check `glxinfo -B` again. |

## 9. Verify every port and controller

Work down this list with the board on the Plasma desktop. Install the test tools first:

```bash
sudo apt install usbutils pciutils ethtool alsa-utils hdparm gpiod lm-sensors nvme-cli
```

- [ ] **Ethernet:** `ip -br addr` shows the wired interface UP with an address; `sudo ethtool <iface> | grep Speed` shows `1000Mb/s`.
- [ ] **HDMI video:** `grep -H . /sys/class/drm/card*-HDMI-A-*/status` shows `connected`. `kscreen-doctor -o` lists the output and its modes.
- [ ] **HDMI audio:** `aplay -l` lists an HDMI card. Run `speaker-test -D plughw:<card>,0 -c 2 -t wav -l 1` with that card's number; you hear "front left, front right" from the monitor.
- [ ] **Headphone jack:** `aplay -l` lists the ES8388 card. Run the same `speaker-test` with its number; you hear it in headphones.
- [ ] **Microphone (onboard or headset):** `arecord -D plughw:<es8388-card>,0 -f cd -d 5 /tmp/mic.wav && aplay /tmp/mic.wav` plays your voice back. Orange Pi's wiki has GUI versions of both audio tests.
- [ ] **USB 3.0 port (upper stacked port):** plug in a USB 3 drive; `lsusb -t` shows it under a `5000M` hub. `sudo hdparm -t /dev/sda` reads well over 100 MB/s with a fast drive.
- [ ] **USB 2.0 ports (two):** a device in each shows under a `480M` hub in `lsusb -t`.
- [ ] **USB-C data:** plug a USB-C drive or hub into the TYPE-C port; it shows in `lsusb -t`. DisplayPort is section 14.
- [ ] **microSD:** `lsblk` shows `mmcblk1` holding `/`; `sudo hdparm -t /dev/mmcblk1` gives a steady read figure.
- [ ] **SPI flash:** `lsblk` shows `mtdblock0` at `16M`.
- [ ] **GPU:** `glxinfo -B` shows `Mali-G610 (Panfrost)` (section 8).
- [ ] **CPU clusters:** `grep . /sys/devices/system/cpu/cpufreq/policy*/cpuinfo_max_freq` lists `1800000` for the A55 cluster and `2400000` (or close) for the two A76 clusters.
- [ ] **Temperatures and fan:** `sensors` shows SoC temperatures, ideally under 70 °C at idle with a heatsink. A fan on the 5 V header spins.
- [ ] **Status LED:** green blinks. `cat /sys/class/leds/*/trigger` shows `[heartbeat]` on one of them.
- [ ] **GPIO header:** `gpiodetect` lists `gpiochip0` to `gpiochip4`.
- [ ] **NPU (vendor kernel only):** `sudo cat /sys/kernel/debug/rknpu/version` prints an RKNPU driver version.
- [ ] **Video decode block (vendor kernel only):** `ls /dev/mpp_service` exists. Using it needs players built for Rockchip MPP, so treat it as present, not tested.
- [ ] **M.2 NVMe (if fitted instead of Wi-Fi):** `sudo nvme list` shows the drive. If `sudo dmesg | grep -i pcie` says `PCIe Link Fail, LTSSM is 0x3` on a V1.3.2 board, apply the M.2 power fix in section 13. It applies to SSDs too.
- [ ] **Wi-Fi and Bluetooth:** section 13.
- [ ] **USB-C DisplayPort:** section 14.

For help on the Armbian forum, `sudo armbianmonitor -u` uploads a full diagnostic log to paste.armbian.com and prints a link to share. Only run it if you are happy to publish that log.

## 10. Troubleshooting a board that will not boot (red LED only)

A red LED with no green blinking means the kernel never started. Power reaches the board, but the bootloader on the card either never ran or stopped early. Work down the tree; the numbered steps below give the commands and the check for each branch.

&#91;embedded content: No-boot decision tree · 5 questions, 6 outcomes\]

Why a stale SPI bootloader matters: the RK3588S boot ROM looks in SPI flash before the microSD card. A valid bootloader there runs first. If it is an old or mismatched version, it can hang while loading the card, even though it still boots USB drives. Armbian forum threads from Jun 2026 show exactly this with Armbian 26.5 vendor images. Used boards, and boards once set up to boot from NVMe, are the usual suspects.

### Steps

1. **Wait, then simplify.** Give a first boot 5 minutes. Then strip the board down to card, power and HDMI: no USB devices, nothing in M.2, no serial adapter. Check: the green LED blinks.
2. **Power.** Use a 5 V / 4 A supply on PWR IN with a short, thick cable. Laptop USB-PD chargers often supply 5 V / 3 A at most. If the red LED does not light at all, try another supply and cable; if it still does not, the board is faulty. Check: the red LED is steady, not flickering.
3. **Card and image.** Confirm the file name contains `Orangepi5_` and `vendor`. Reflash with verification (section 5 or 6), and re-seat the card. Then try a second card, ideally one from the recommended list. Check: the verify step passed and the board boots.
4. **USB-reader test.** Put the same card in a USB card reader in the upper USB 3.0 port, with the microSD slot empty, and power on. Check: if it boots now, the cause is a bootloader in SPI flash. Go to step 5.
5. **Back up and wipe SPI flash from Linux.** While booted from the USB reader (or any system running on this board):

```bash
lsblk | grep mtd                                  # mtdblock0 16M
sudo dd if=/dev/mtdblock0 of=~/opi5-spi-backup.img bs=1M status=progress
sudo apt install mtd-utils
sudo flash_erase /dev/mtd0 0 0                    # erases all 16 MB
# alternative without mtd-utils:
# sudo dd if=/dev/zero of=/dev/mtdblock0 bs=1M count=16 conv=fsync status=progress
sudo poweroff
```

Copy `~/opi5-spi-backup.img` to another computer; it restores the old bootloader with `sudo dd if=opi5-spi-backup.img of=/dev/mtdblock0 bs=1M conv=fsync`. Now remove the USB reader, put the card in the microSD slot and power on. Check: the green LED blinks and Armbian boots from the slot. Later, if you want the board to boot NVMe or USB without a card, `sudo armbian-install` can write Armbian's own matching bootloader to SPI flash.

6. **Known-good image.** Flash Orange Pi's official Ubuntu image to a second card (section 11, A). Check: if it boots and Armbian does not, erase SPI with RKDevTool or rkdeveloptool (section 11, B or C) and reflash Armbian.
7. **MaskROM erase.** If nothing boots at all, erase SPI from MaskROM mode (section 11, B on Windows or C on Garuda). Check: RKDevTool or `rkdeveloptool ld` shows a MaskROM device, and the erase completes.
8. **Serial console.** Read the boot log (section 12) to see the exact stage where it stops.
9. **Hardware fault.** If no MaskROM device ever appears with a known data cable, or the erase fails every time, contact the seller or Orange Pi with the serial log.

## 11. Troubleshooting with Orange Pi's official tools

Orange Pi's [Orange Pi 5 download page](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-pi-5.html) has an **Official Tools** link to a Google Drive folder. What it held on 3 Nov 2026, and what each tool is for here:

| Tool (folder name) | Version | Use in this guide |
| --- | --- | --- |
| RKDevTool and driver | RKDevTool\_Release\_v3.15, DriverAssistant\_v5.12 | Windows: talk to the board in MaskROM mode, erase SPI flash |
| MiniLoader ("what is needed for burning Linux images") | MiniLoaderAll.bin, rkspi\_loader.img, rk3588\_linux\_\*.cfg | The loader RKDevTool needs; Orange Pi's SPI bootloader |
| Windows Formatting Software (SDCardFormatter) |  | Reset a misbehaving card (section 3) |
| Linux image writing tool: balenaEtcher, Win32Diskimager |  | Flash and verify the official test image (section 6) |
| MobaXterm |  | Windows serial console (section 12) |
| SDDiskTool | v1.72 | Android images only; not needed |
| NoMachine, FileZilla, wiringOP |  | Remote desktop, file transfer, GPIO library; optional |

### A. Known-good test: Orange Pi's official image

This splits "Armbian or bootloader problem" from "card, power or board problem" in one step.

1. From the same page, open **Ubuntu Image** and download the newest Orange Pi 5 Ubuntu 22.04 (Jammy) desktop image. It is Orange Pi's own build for this exact board.
2. Flash it with balenaEtcher (decompress first) or Win32 Disk Imager, with verification (section 6).
3. Boot it. The default login is user `orangepi`, password `orangepi`.

| Official image | Armbian | Likely cause | Next step |
| --- | --- | --- | --- |
| Boots | Does not | Bootloader mismatch, often a stale bootloader in SPI flash | Erase SPI (B or C below), then reflash Armbian |
| Does not | Does not | Card, power supply or SPI | Another card and supply, then B or C |
| Does not, even after erasing SPI with another card and supply | Does not | Board hardware | Serial console (section 12), then the seller |

On the official image, `sudo orangepi-config` is Orange Pi's settings tool. Its System > Hardware menu turns on overlays such as `wifi-ap6275p`, which is a useful cross-check for section 13.

### B. Erase SPI flash with RKDevTool (Windows 11 desktop)

Erasing SPI is safe on this board: with SPI empty, the boot ROM goes on to the microSD card. Back it up first (section 10, step 5, or step 4 of C below) if the board used to boot from NVMe and you may want that bootloader back.

**Which USB cable.** Use a USB-A to USB-C **data** cable in a USB-A port on the computer if you can. Orange Pi only asks for a good-quality Type-C data cable, but Radxa's RK3588 instructions specify USB-A to USB-C, and it always works: a USB-A port always supplies 5 V, and the cable itself tells the board it is plugged into a computer. A USB-C to USB-C data cable often works too. It is less certain because a USB-C port stays off until it detects a device, and in MaskROM mode nothing has set up the board's USB-C controller yet.

- Either cable must carry data. Many USB-C cables only charge; any USB 2.0 or faster data cable is fast enough.
- Plug it into the board's **TYPE-C** port, never PWR IN, and keep powering the board from its own supply on PWR IN.
- With USB-C to USB-C, if the board doesn't show up, unplug the cable, flip the plug over at the board end and enter MaskROM again. If it still doesn't, use a USB-A to USB-C cable, or put a USB-C to USB-A adapter on the computer end.

1. Download **DriverAssitant\_v5.12.zip** (Orange Pi's spelling), **RKDevTool\_Release\_v3.15.zip** and the whole **MiniLoader** folder.
2. Unzip DriverAssistant, run `DriverInstall.exe`, click **Driver Installation**, wait for "driver installed successfully".
3. Unzip RKDevTool and run `RKDevTool.exe` (no install). It shows "No device found".
4. Remove the microSD card. Unplug the board's power.
5. Connect the **data** cable (see **Which USB cable** above) from the PC to the board's **TYPE-C** port (not PWR IN).
6. Press and hold the **MaskROM** key. While holding it, connect power to **PWR IN**. Release the key after about 2 seconds.
7. RKDevTool now shows "Found One MASKROM Device".
8. Open the **Advanced Function** tab. In the Boot field, pick `MiniLoaderAll.bin` from the MiniLoader folder and click **Download**. Wait for the download to succeed.
9. Set the storage device to **SPINOR** and click **Switch Storage**.
10. Click **Erase All**. The log ends with a success line for the erase.
11. Unplug power and the USB cable, insert the Armbian card and power on.

If step 7 never shows a device: try another cable (many USB-C cables are charge-only), a USB 2.0 port on the PC, and re-run Driver Installation. With a USB-C to USB-C cable, flip the plug at the board end and enter MaskROM again, or switch to a USB-A to USB-C cable. If you get "Test Device Fail" or "Get FlashInfo Fail" every time, the SPI chip or board may be faulty; see section 12 for the serial log.

### C. Same erase from Garuda with rkdeveloptool

The same erase from the Garuda laptop. It clears only the 16 MB SPI flash; the NVMe drive and the microSD card are untouched. You need the cable described under **Which USB cable** in B, the board's power supply, and Orange Pi's **MiniLoader** folder (B, step 1). The rkdeveloptool package's udev rule doesn't cover the RK3588, so every command here uses `sudo`.

**Step 1: install rkdeveloptool.** It isn't in Garuda's or Chaotic-AUR's repos, so paru builds it from the AUR (yay works too). The package builds Rockchip's current source, which includes the `cs` storage command added in March 2025.

```bash
sudo pacman -S --needed base-devel git
paru -S rkdeveloptool
rkdeveloptool -h | grep ChangeStorage
```

Pass: the last command prints `ChangeStorage: cs [storage: 1=EMMC, 2=SD, 9=SPINOR]`.

**Step 2: put the board in MaskROM mode.**

1. Remove the microSD card and unplug the board's power.
2. Connect the data cable from the laptop to the board's **TYPE-C** port, not PWR IN.
3. Press and hold the **MaskROM** key, beside the audio jack.
4. While holding it, connect power to **PWR IN**. Release the key after about 2 seconds.

```bash
lsusb | grep 2207:350b
sudo rkdeveloptool ld
```

Pass: `lsusb` prints a line containing `2207:350b`, and `ld` shows `DevNo=1 Vid=0x2207,Pid=0x350b,... Maskrom`.

**Step 3: load the loader and select SPI flash.** Change the path if you saved the MiniLoader folder somewhere else.

```bash
sudo rkdeveloptool db ~/Downloads/MiniLoader/MiniLoaderAll.bin
sudo rkdeveloptool cs 9
sudo rkdeveloptool rfi
```

Pass: `db` prints `Downloading bootloader succeeded.`, `cs 9` prints `Change Storage OK.`, and `rfi` shows `Flash Size: 16 MB`. If `rfi` shows a much larger size, the wrong storage is selected: stop, and don't run the erase.

**Step 4: back up the SPI flash (optional, recommended).** This saves all 16 MB (32,768 sectors of 512 bytes), including any bootloader you may want back:

```bash
sudo rkdeveloptool rl 0 32768 ~/opi5-spi-backup.img
```

To restore it later, repeat steps 2 and 3, then run `sudo rkdeveloptool wl 0 ~/opi5-spi-backup.img`. From Linux on the board, the `dd` command in section 10, step 5 restores it too.

**Step 5: erase and confirm.**

```bash
sudo rkdeveloptool ef
sudo rkdeveloptool rl 0 32768 ~/opi5-spi-after.bin
tr -d '\377' < ~/opi5-spi-after.bin | wc -c
```

Pass: `ef` ends with `Erasing flash complete.` (don't unplug anything before it does), and the last command prints `0`: every byte reads as erased (0xFF).

**Step 6: boot from the card.** Unplug power and the USB cable, insert the Armbian card and power on. With SPI flash empty, the boot ROM goes on to the microSD card, and the green LED should start blinking within a minute or so.

**If something goes wrong:**

| Problem | Fix |
| --- | --- |
| `not found any devices!`, or no `2207:350b` in `lsusb` | Try another data cable and another USB port, then repeat step 2, holding the key until power is connected. With a USB-C to USB-C cable, flip the plug at the board end, or switch to USB-A to USB-C |
| `Creating Comm Object failed!` | Check that the command started with `sudo`. If it did, look for the same cable, power or hardware faults as in the row above |
| `Change Storage failed! Storage 9 is not available.` | Run the `db` command again, then `cs 9`. If it still fails, the loader can't see the SPI chip: get the serial log (section 12) |
| `rl` fails after the erase | Run step 3 again, then repeat the read-back |
| `Getting flash info from device failed!` every time | The SPI chip or board may be faulty: get the serial log (section 12) |
| `ChangeStorage` missing from `rkdeveloptool -h` | The build is too old: reinstall it from the AUR (step 1), or use RKDevTool on Windows (B) |

## 12. Serial console (when you buy an adapter)

A serial console shows the boot log from the first instruction onward, even with no HDMI picture and no network. It is the one tool that tells you exactly where a red-LED-only board stops.

### What to buy

A **3.3 V USB-to-TTL adapter that supports 1,500,000 baud**, plus three female-to-female jumper (DuPont) wires.

| Chip | 1.5 Mbaud | Notes |
| --- | --- | --- |
| CH340 / CH343 | Yes | What Orange Pi recommends. Drivers are built into Linux, macOS and Windows 11. |
| FTDI FT232RL | Yes (up to 3 Mbaud) | Good; avoid suspiciously cheap clones |
| CP2102N | Yes (up to 3 Mbaud) | Fine; the older CP2102 without the N is not |
| CP2102, PL2303 | No or unreliable | Orange Pi says not to use them |

If the adapter has a 3.3 V / 5 V jumper, set it to **3.3 V**.

### Wire it

The debug header is the 3-pin header beside the 26-pin header, marked TX RX GND.

| Adapter pin | Board pin |
| --- | --- |
| GND | GND |
| RX | TX |
| TX | RX |
| VCC / 3V3 / 5V | **Not connected** |

TX and RX cross over. If you see nothing at all, swap the two; that is harmless. Plug the adapter into your computer first, start the terminal program, then power the board, so you capture the log from the start.

Settings for every program: **1500000 baud, 8 data bits, no parity, 1 stop bit, no flow control**. Linux calls this port `ttyS2` on the board side.

### Garuda

```bash
sudo pacman -S picocom              # or: tio
sudo usermod -aG uucp $USER         # Arch's serial group; log out and back in
ls /dev/ttyUSB* /dev/ttyACM*        # usually /dev/ttyUSB0
picocom -b 1500000 /dev/ttyUSB0     # quit: Ctrl+A then Ctrl+X
# save a log: picocom -b 1500000 --logfile opi5-boot.log /dev/ttyUSB0
```

### macOS

```bash
brew install tio
ls /dev/cu.*                        # e.g. /dev/cu.usbserial-1410 or /dev/cu.wchusbserial1410
tio -b 1500000 /dev/cu.usbserial-1410   # quit: Ctrl+T then Q
```

Use `tio` rather than the built-in `screen`, which may not accept a 1,500,000 baud setting.

### Windows 11

1. Device Manager > Ports (COM & LPT): note the adapter's COM number.
2. In **PuTTY**: Connection type Serial, Serial line `COM5` (yours), Speed `1500000`. Under Connection > Serial set Flow control to None. Click Open.
3. Or in **MobaXterm** (in Orange Pi's official tools): Session > Serial, pick the COM port, type `1500000` as the speed, set flow control to None under Advanced settings.

### Reading the log

| Last thing you see | Meaning | Go to |
| --- | --- | --- |
| Nothing at all | Adapter wiring or baud rate. Swap TX and RX, re-check 1500000. |  |
| `DDR ... LPDDR4X` lines, then stops | RAM training failed: power supply or board fault | Better supply, then seller |
| `U-Boot SPL` then `Trying to boot from MMC2` then an error (`spl: partition error`) | SPL can't load the next stage from the card | Reflash and verify; another card |
| A U-Boot banner that is not Armbian's, or an old date, then nothing from the card | A stale bootloader in SPI flash | Section 10, erase SPI |
| `Starting kernel ...` then silence | Wrong device tree or a bad overlay | Remove the last overlay you added from `armbianEnv.txt` |
| Kernel messages, then a login prompt | It booted; your problem is display or network | Section 7 |

A good Armbian boot shows a U-Boot banner with `armbian` in it and `Model: Xunlong Orange Pi 5`, then `Applying kernel provided DT overlay ...` for each overlay, then the kernel log. Armbian's default `console=both` gives you a login prompt on the serial port too.

To stop autoboot and poke around, press any key at `Hit any key to stop autoboot`. `help` lists commands; `boot` continues.

## 13. AP6275P Wi-Fi 6 and Bluetooth module

Do this after the board passes section 9. The AP6275P is Orange Pi's M.2 card for this board: a Broadcom BCM43752 with 2x2 Wi-Fi 6 over PCIe and Bluetooth 5 over a UART (`/dev/ttyS9`), both through the M.2 slot. On the vendor kernel it is switched on by one overlay, `orangepi-5-ap6275p`, and an Armbian maintainer wrote in Jul 2026 that it "works flawlessly on vendor kernel" with it.

### Install the module

1. Shut down (`sudo poweroff`) and unplug power.
2. Snap both antennas onto the module's two U.FL/IPEX sockets first: press straight down until they click. Without antennas, range is a few metres at best.
3. Insert the module into the M.2 socket on the underside at about 30°, press it flat and fit the retaining screw.
4. The M.2 slot now holds Wi-Fi, so an NVMe SSD cannot be fitted at the same time. Keep booting from microSD.
5. Route the antenna leads away from the heatsink and fan.

### Turn it on

```bash
sudoedit /boot/armbianEnv.txt
```

Add the overlay to your existing `overlays=` line (one line, space-separated):

```
overlay_prefix=rockchip-rk3588
overlays=panthor-gpu orangepi-5-ap6275p
```

Then `sudo reboot`. With that prefix, the file loaded is `/boot/dtb/rockchip/overlay/rockchip-rk3588-orangepi-5-ap6275p.dtbo`. `sudo armbian-config` can tick the same overlay for you.

### Verify, in this order

| Step | Command | Pass |
| --- | --- | --- |
| 1. Overlay file exists | `ls /boot/dtb/rockchip/overlay/ \| grep ap6275p` | `rockchip-rk3588-orangepi-5-ap6275p.dtbo` |
| 2. PCIe link | `sudo dmesg \| grep -i "pcie.*link"` | `PCIe Link up`, not `Link Fail` |
| 3. Device on the bus | `lspci` | A Broadcom network controller line |
| 4. Driver | `sudo dmesg \| grep -i -E "wlan\|dhd"` | `wifi_chip_type = ap6275p` and no firmware errors |
| 5. Interface | `ip link show wlan0` and `nmcli device` | `wlan0` present, type `wifi` |
| 6. Scan and connect | `nmcli dev wifi list`, then `nmcli --ask dev wifi connect "YourSSID"` | Your networks listed; connection succeeds. The Plasma network applet works too. |
| 7. Bluetooth service | `systemctl status bluetooth-hciattach` | `active (running)` |
| 8. Bluetooth controller | `bluetoothctl show` | A controller with `Powered: yes` |
| 9. Bluetooth scan | `bluetoothctl --timeout 15 scan on` | Nearby devices appear. Pair from Plasma's Bluetooth settings. |

On Armbian vendor images, Bluetooth comes up through a service, `bluetooth-hciattach`, which runs `rfkill unblock 0` and then `hciattach -n -s 115200 /dev/ttyS9 bcm43xx 1500000`. Mainline (current or edge) images do not use it; there the kernel binds Bluetooth itself.

### When it does not work

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `lspci` empty and `dmesg` shows `PCIe Link Fail, LTSSM is 0x3` | **M.2 power-pin bug on PCB V1.3.2.** The device tree powers the slot through GPIO0\_C5, but V1.3.2 boards switch it with GPIO2\_A1, so the slot never gets power. Reported on vendor 6.1.115, current 6.18 and edge 7.1 kernels in Aug to Oct 2026; V1.2 boards are not affected. It breaks NVMe the same way. | Apply the user overlay below, if your board says V1.3.2 |
| `lspci` empty, no `LTSSM` lines at all | Overlay not loaded: a typo in its name or a missing `overlay_prefix` (the cause in one 2026 forum thread) | Fix `armbianEnv.txt`. On the serial console (section 12), look for `Applying kernel provided DT overlay rockchip-rk3588-orangepi-5-ap6275p.dtbo` |
| Still empty after both fixes | Module not seated, or faulty | Re-seat and re-screw it, then try test D below |
| Device in `lspci`, no `wlan0` | Missing firmware | `sudo dmesg \| grep -i firmware` shows the path it wanted. Armbian's firmware set has `fw_bcm43752a2_pcie_ag.bin`, `nvram_ap6275p.txt` and `clm_bcm43752a2_pcie_ag.blob`. Run `sudo apt install --reinstall armbian-firmware-full`, then reboot. |
| 2.4 GHz works, 5 GHz will not connect | Reported on Armbian 26.x vendor builds in Oct 2026; the same user says 5 GHz works on the 7.1 and 7.2 kernels | Use 2.4 GHz for now. Or set the router's 5 GHz band to a non-DFS channel (36 to 48), 80 MHz width and WPA2/WPA3 mixed, then retry. Moving to an edge kernel fixes it but loses USB-C DisplayPort (section 14). |
| Weak or dropping signal | Antenna missing, loose or against the heatsink | Re-seat both antennas, move the leads |
| No Bluetooth controller | Service not running, or the radio is blocked | `sudo rfkill list` then `sudo rfkill unblock all`. `sudo systemctl enable --now bluetooth-hciattach`. To see its errors, stop it and run the `hciattach` command above by hand. |
| `bluetoothctl show` says `Powered: no` and `PowerState: off-blocked` | rfkill is blocking the controller. The service's `rfkill unblock 0` runs before the controller's own `hci0` entry exists, so it doesn't clear a block on that entry. Seen on an Armbian 26.8 KDE image in Nov 2026. | `sudo rfkill unblock bluetooth`, then `bluetoothctl power on`. Always run rfkill with `sudo`: it lives in `/usr/sbin`, so without `sudo` a normal user can get command not found. Reboot and check it still says `Powered: yes`. If not, switch Bluetooth on in System Settings > Bluetooth, because Plasma reapplies its saved on/off setting at every login. |
| Bluetooth works until reboot | Service disabled | `sudo systemctl enable bluetooth-hciattach` |

### Fix for the V1.3.2 M.2 power pin

Only for boards whose silkscreen reads **V1.3.2**. Do not apply it to a V1.2 board, which uses the original pin. This overlay was posted on the Armbian forum on 11 Oct 2026 and confirmed by other users on the vendor kernel.

```bash
sudo apt install device-tree-compiler      # if armbian-add-overlay asks for dtc
cat > /tmp/pcie-m2-power-fix.dts <<'EOF'
/dts-v1/;
/plugin/;
/ {
    compatible = "rockchip,rk3588";
    fragment@0 {
        target = <&vcc3v3_pcie2x1l2>;
        __overlay__ {
            gpios = <&gpio2 1 0>;
        };
    };
};
EOF
sudo armbian-add-overlay /tmp/pcie-m2-power-fix.dts
sudo reboot
```

`armbian-add-overlay` compiles it into `/boot/overlay-user/` and adds a `user_overlays=` line to `armbianEnv.txt`. After the reboot, repeat steps 2 and 3 of the verify table. On current and edge kernels the regulator is named `vcc3v3_pcie20`, so change the target to `<&vcc3v3_pcie20>` there. To undo, delete the `user_overlays=` line and reboot.

### D. Cross-check with Orange Pi's official image and orangepi-config

1. Boot Orange Pi's official Ubuntu image (section 11, A) with the module fitted.
2. Run `sudo orangepi-config`, then System > Hardware, highlight `wifi-ap6275p`, press Space to select it, then Save, Back and Reboot. This writes `overlays=wifi-ap6275p` to `/boot/orangepiEnv.txt`.
3. Check `ip addr show wlan0` and `hciconfig -a` (Orange Pi's own tests). `hciconfig` should show `Bus: UART` and `Manufacturer: Broadcom Corporation`.

If it works on the official image but not on Armbian, the module is fine: recheck the overlay and the V1.3.2 fix. If it fails on both, re-seat the module or replace it.

## 14. USB-C DisplayPort output

The TYPE-C port carries DisplayPort 1.4 in alt mode, so a monitor can run from it instead of HDMI. This works on the **vendor kernel**: its Orange Pi 5 device tree enables the DisplayPort controller (`dp0`, on video port 2), the DP lanes of the USB-C PHY, and the FUSB302 Type-C controller with the DisplayPort alt mode (SVID `0xff01`). No overlay is needed. On Armbian's current (6.18) and edge kernels it does not work yet: as of Apr 2026 the mainline alt-mode patches were still pending, and the Armbian change that fixed USB-C on mainline left DisplayPort untested.

### What you need

- The **TYPE-C** port, beside HDMI. PWR IN carries no data or video.
- A **USB-C to DisplayPort** cable, or a USB-C to HDMI adapter, that says "DP Alt Mode". Cables sold as Thunderbolt 3/4 compatible display cables usually qualify. A "USB display adapter" (DisplayLink and similar) is a different technology that needs its own driver; it will not work this way.
- Video and USB 3 share the port's high-speed lanes. Orange Pi describes DP 1.4 and USB 3.1 as multiplexed, so plug the monitor in directly at first, not through a hub or dock.

### Set it up

1. Boot to the Plasma desktop on HDMI as usual.
2. Plug the cable into TYPE-C and the monitor's DisplayPort input. Select that input on the monitor.
3. Check that the board sees it:

```bash
uname -r                                        # must contain vendor
grep -H . /sys/class/drm/card*-DP-*/status      # ...DP-1/status:connected
ls /sys/class/typec/                            # port0 and port0-partner
sudo dmesg | grep -i -E "fusb302|typec|altmode|dp0" | tail -20
kscreen-doctor -o                               # lists DP-1 with its modes
```

4. In System Settings > Display & Monitor (Display Configuration), enable the DP output, pick its resolution and refresh rate, and make it the primary display. To run on DisplayPort only, disable the HDMI output there. Click Apply, then Keep.

The same from Konsole on the desktop. Over SSH, first run `export WAYLAND_DISPLAY=wayland-0 XDG_RUNTIME_DIR=/run/user/$(id -u)` as the user who is logged in to Plasma:

```bash
kscreen-doctor output.DP-1.enable output.DP-1.mode.1920x1080@60 output.DP-1.priority.1
kscreen-doctor output.HDMI-A-1.disable          # optional: DisplayPort only
```

Use the exact output names that `kscreen-doctor -o` printed.

5. Test a DisplayPort-only boot: shut down, unplug HDMI, power on. U-Boot's boot messages and logo appear only on HDMI, so the DP screen stays dark for roughly 20 to 40 seconds, until the kernel's display driver loads. The SDDM login screen then appears on DisplayPort.

### Verify

- [ ] `DP-1/status` reads `connected`
- [ ] The desktop shows on the DP monitor at the resolution you set
- [ ] After a reboot with HDMI unplugged, SDDM appears on DP
- [ ] Both monitors work at once, if you want dual displays (HDMI and DP use separate video ports)

### When the monitor stays dark

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `status` reads `disconnected` | Wrong port, a cable without DP alt mode, or a mainline kernel | Use TYPE-C, not PWR IN. Confirm `uname -r` says vendor. Try another alt-mode cable. |
| Works one way up only | Plug orientation was not detected | Turn the USB-C plug over, then unplug and replug |
| No `port0-partner` in `/sys/class/typec` | The FUSB302 did not negotiate with the cable | Replug after the desktop is up. Check `dmesg` for fusb302 errors. Try a shorter cable or another adapter. |
| `connected` but black or "no signal" | A mode the link cannot carry, or the monitor dislikes the default | Set 1920x1080@60 with `kscreen-doctor` over SSH, then step up. An earlier Armbian forum report found trouble with one monitor's native 4K mode while another monitor worked. |
| DP works, USB 3 devices on the same port stop | Lane sharing between DP and USB 3 | Expected. Use the USB 3.0 Type-A port for fast storage. |
| The lower-right USB 2.0 port acts up while USB-C is in use | It shares USB 2.0 lines with the USB-C port | Move that device to the stacked USB 2.0 port |
| Stopped working after you changed kernels | You moved off the vendor kernel | `sudo armbian-config`, System > Kernel, choose the vendor kernel again |
| Still nothing | Cable or monitor | Cross-check on Orange Pi's official image (section 11, A), whose wiki documents display output through the Type-C port. If it fails there too, the cable or monitor is the problem. |

## Sources

Opened between 3 and 6 Nov 2026.

**Armbian**

- [Orange Pi 5 download page](https://www.armbian.com/orangepi-5/): images, kernels, SHA and ASC files, "Standard" support
- [Armbian Imager releases](https://github.com/armbian/imager/releases) (v2.5.0, 28 Oct 2026) and [imager.armbian.com](https://imager.armbian.com)
- [Getting started](https://docs.armbian.com/getting-started/), [choosing an image and verifying it](https://docs.armbian.com/getting-started/choosing-an-image/), [SD cards](https://docs.armbian.com/getting-started/sd-cards/), [writing the image](https://docs.armbian.com/getting-started/writing-the-image/)
- [Vendor kernel Orange Pi 5 device tree](https://github.com/armbian/linux-rockchip/blob/rk-6.1-rkr7.2/arch/arm64/boot/dts/rockchip/rk3588s-orangepi-5.dts): DP alt mode, FUSB302, `dp0`
- Vendor overlays: [orangepi-5-ap6275p](https://github.com/armbian/linux-rockchip/blob/rk-6.1-rkr7.2/arch/arm64/boot/dts/rockchip/overlay/orangepi-5-ap6275p.dts) and [rockchip-rk3588-panthor-gpu](https://github.com/armbian/linux-rockchip/blob/rk-6.1-rkr7.2/arch/arm64/boot/dts/rockchip/overlay/rockchip-rk3588-panthor-gpu.dts)
- [Orange Pi 5 board config](https://github.com/armbian/build/blob/main/config/boards/orangepi5.conf) and the [bluetooth-hciattach extension](https://github.com/armbian/build/blob/main/extensions/bluetooth-hciattach.sh)
- [AP6275P firmware files](https://github.com/armbian/firmware/tree/master/ap6275p)
- [PR 9624, USB-C fix for mainline](https://github.com/armbian/build/pull/9624) (merged 16 Apr 2026; DP alt mode untested), [PR 10289, AP6275P on mainline kernels](https://github.com/armbian/build/pull/10289) (merged 27 Jul 2026), [PR 8979, GPU default](https://github.com/armbian/build/pull/8979) (closed unmerged)

**Armbian forum**

- [AP6275P not working, Jun to Jul 2026](https://forum.armbian.com/topic/60325-orange-pi-5-pcie-wifi-adapter-ap6275p-not-working/)
- [M.2 / PCIe power-pin bug on V1.3.2, Aug to Oct 2026](https://forum.armbian.com/topic/61753-orangepi-5-nvme-pcie-voltage-regulator-gpio-bug/)
- [AP6275P 5 GHz on 26.x vendor, Oct 2026](https://forum.armbian.com/topic/62460-ap6275-driver-not-connecting-to-5g-wifi-on-armbian-26x-with-vendor-kernel/)
- [Cannot boot from microSD, stale SPI, Jun 2026](https://forum.armbian.com/topic/60051-cannot-get-orange-pi-5-to-boot/)
- [Bricked board and MaskROM, Feb 2025](https://forum.armbian.com/topic/49922-orange-pi-5-bricked/)
- [AP6275P Bluetooth and hciattach, 2023](https://forum.armbian.com/topic/28950-orange-pi-5-ap6275p-wireless-module-bluetooth-not-working/)
- [Default GPU acceleration on RK3588, Nov 2025](https://forum.armbian.com/topic/56374-expected-default-graphics-acceleration-for-rk3588/)

**Orange Pi (official)**

- [Orange Pi 5 wiki](http://www.orangepi.org/orangepiwiki/index.php/Orange_Pi_5): specs, buttons, flashing with balenaEtcher
- [Orange Pi 5 downloads](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/service-and-support/Orange-pi-5.html): Official Tools, images, user manual
- [Orange Pi 5 product page](http://www.orangepi.org/html/hardWare/computerAndMicrocontrollers/details/Orange-Pi-5.html) and [labelled board diagram](http://www.orangepi.org/img/pi5-fix/pi5-21.png)
- Wiki: [onboard LEDs](http://www.orangepi.org/orangepiwiki/index.php/Onboard_LED_Light_Test_Instructions), [AP6275P module](http://www.orangepi.org/orangepiwiki/index.php/How_to_use_AP6275P_PCIe_network_card), [Bluetooth](http://www.orangepi.org/orangepiwiki/index.php/How_to_use_Bluetooth)
- [Orange Pi 5 Plus wiki](http://www.orangepi.org/orangepiwiki/index.php/Orange_Pi_5_Plus): the RKDevTool SPI-erase and serial-console procedures, which use the same RK3588 tools

**Other**

- [Ubuntu Rockchip repository](https://github.com/Joshua-Riek/ubuntu-rockchip), archived 29 Apr 2026
- [Radxa: erase SPI flash with rkdeveloptool](https://docs.radxa.com/en/rock5/rock5b/low-level-dev/erase-spi-from-usb-otg), same RK3588 tooling
- [rkdeveloptool source](https://github.com/rockchip-linux/rkdeveloptool) (Rockchip): commands and messages, the `cs` storage command (merged March 2025), udev rules
- [AUR: rkdeveloptool](https://aur.archlinux.org/packages/rkdeveloptool): builds from Rockchip's current source
- [Radxa: Rock 5B USB install](https://wiki.radxa.com/Rock5/install/usb-install-rock-5b): MaskROM with a USB-A to USB-C cable
- [Radxa: Rock 3 USB install](https://wiki.radxa.com/Rock3/install/usb-install): USB-C to USB-C seen failing, an adapter worked
- [Radxa forum: MaskROM not working](https://forum.radxa.com/t/maskrom-not-working-what-am-i-doing-wrong/13672): USB-C to USB-C working for one RK3588 user
- [Pine64 forum thread](https://forum.pine64.org/printthread.php?tid=8927): tip to flip the USB-C plug
- [KDiskMark repository](https://github.com/JonMagon/KDiskMark): v3.3.0, default profile and settings, polkit use
- SD Association: [SD Memory Card Formatter for Windows/Mac](https://www.sdcard.org/downloads/formatter/) (5.0.4) and [for Linux](https://www.sdcard.org/downloads/sd-memory-card-formatter-for-linux/) (1.0.3), with their user manuals: options, tested OS versions, Quick vs Overwrite
- [Ubuntu 26.04 rfkill package](https://packages.ubuntu.com/resolute/rfkill): installs `/usr/sbin/rfkill`, so a normal user needs `sudo` (section 13)
- [kscreen-doctor source (libkscreen)](https://invent.kde.org/plasma/libkscreen): the `mode`, `maxbpc` and `priority` options used in sections 8 and 14

*Guide developed by me with research and translation assistance by Claude Opus 5.5 Extra*
