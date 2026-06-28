# makedrive User Guide

**Version:** Build 201  
**Updated:** 2026-06-28  
**Author:** Ian Williams  
**Contact:** [ian@themacgenie.com](ian@themacgenie.com)  
**Project:** [https://github.com/TheMacGenie/makedrive](https://github.com/TheMacGenie/makedrive)

---

## Contents

- [Introduction](#introduction)
- [Hardware & Software Requirements](#hardware--software-requirements)
- [Quick Start](#quick-start)
- [The Main Menu](#the-main-menu)
- [makedrive.conf](#makedriveconf)
- [Adding Images to restorekit](#adding-images-to-restorekit)
- [Mac App Store Installers](#mac-app-store-installers)
- [Building Install Drives](#building-install-drives)
- [Restoring to USB or SD Card](#restoring-to-usb-or-sd-card)
- [Volume Icons](#volume-icons)
- [Pushover Notifications](#pushover-notifications)
- [Uninstalling makedrive](#uninstalling-makedrive)
- [Security Considerations](#security-considerations)
- [Changelog](#changelog)
- [Future Plans](#future-plans)
- [Formalities and Colophon](#formalities-and-colophon)

---

## Introduction

makedrive builds ready-to-use macOS install drives from a
human-navigable folder of DMG files called **restorekit**. Instead of manually
partitioning disks, mounting images, and running `asr` by hand, makedrive
automates the whole process: it verifies each installer, processes it into a
restorable image, and deploys your chosen set of installers onto a target disk
in a single pass.

A single makedrive disk can carry many macOS installers side by side — from the
oldest retail DVDs through the current release — so a technician can boot or
re-install almost any Mac from one drive. Building these drives by hand is
tedious and error-prone; makedrive makes it consistent and repeatable, and lets
you re-purpose a drive quickly when a new macOS version ships.

> **Note for users of older makedrive releases:** makedrive was historically a
> diagnostic/triage tool tied to service provider workflows. Those
> diagnostic features have been removed. The current tool is focused on building
> macOS install drives.

---

## Hardware & Software Requirements

**Imaging host (the Mac that runs makedrive):**

- A reasonably current version of macOS. makedrive is maintained against the
  latest macOS releases (through macOS 26/27) and runs on both Apple Silicon and
  Intel Macs.
- The **Xcode Command Line Tools.** makedrive uses `setfile` (to apply volume
  icons) and `python3` (to check installer versions against Apple's catalog). If
  the tools are not present, makedrive offers to install them at launch and then
  exits so installation can complete.
- A **local (non-network) administrator account.** makedrive uses command-line
  tools that require root access by way of `sudo`, so full administrator rights
  on the host are required.
- High-speed ports (Thunderbolt, USB‑C, or USB 3) for the fastest deployment
  across the widest range of target disks.

**Target disk (the drive being built):**

- Size the target to the installers you intend to bundle plus optional DataDrive
  space. A drive that carries many installers needs to be correspondingly large;
  acquiring the largest disks practical keeps a drive useful as new macOS
  versions are added.
- A disk offering both modern (USB 3 / USB‑C) and legacy (FireWire) connectivity
  gives the widest hardware compatibility if you support vintage Macs.

---

## Quick Start

makedrive relies on the **restorekit** folder as its source for the DMG files it
deploys. restorekit must sit in the **same directory as the makedrive script**.
The simplest setup is to keep `makedrive.command` and `restorekit` together in
the Desktop folder of a dedicated imaging account — double-clicking
`makedrive.command` then launches it in Terminal.

At the start of a session you are prompted for the administrator password. Once
authenticated, the password is not requested again for the rest of the session,
no matter how many drives you image.

If a DMG that one of your configurations expects is missing from restorekit, a
notice listing the missing files appears above the Main Menu. Add the missing
installers (Main Menu option 1) and the notice clears.

If restorekit is not found next to the script, makedrive offers to create an
empty one so you can start from scratch and add images from within the script.

---

## The Main Menu

```
Welcome to makedrive's Main Menu.

1. Add installers to the restorekit folder
2. Compress and scan DMGs for restore
3. Build your install drive
4. Restore image to USB or SD card
5. Configure Pushover notifications
6. Uninstall makedrive from this Mac
X. Exit makedrive
```

- **1 — Add installers to the restorekit folder.** Verifies an installer (by
  volume name and macOS build number) and copies it into the correct place in
  restorekit, then compresses and scans it for restore.
- **2 — Compress and scan DMGs for restore.** Explicitly checks the DMGs in
  restorekit, compressing any that aren't already compressed and adding `asr`
  scan data where it's missing. Images added through option 1 are already
  processed; this option is mainly for images placed into restorekit by hand.
- **3 — Build your install drive.** Erases a chosen target disk and deploys the
  installers for the build type you select. This is where most time is spent.
- **4 — Restore image to USB or SD card.** Restores a single installer image to
  a USB flash drive, SD card, or other small media in one step. See
  [Restoring to USB or SD Card](#restoring-to-usb-or-sd-card).
- **5 — Configure Pushover notifications.** Optional push notifications when a
  build finishes. See [Pushover Notifications](#pushover-notifications).
- **6 — Uninstall makedrive.** Removes makedrive from the host. See
  [Uninstalling makedrive](#uninstalling-makedrive).

If you add DMGs to restorekit with the Finder rather than through option 1, they
are neither verified nor compressed/scanned — use option 2 (or, preferably,
always use option 1) to be sure they are ready for restore.

---

## makedrive.conf

Everything makedrive deploys is described in an external configuration file,
**makedrive.conf**. Keeping this in a separate file means new macOS versions and
build configurations can be added by editing one place, without touching the
script.

makedrive.conf defines:

- **Image definitions** — one block of variables per installer, keyed by a
  unique prefix (for example `inst2600u`). Each block sets the DMG path, the
  deploy type, display name, final volume name, partition size and filesystem,
  the macOS build number to verify against, and the label shown in the add-image
  menu.
- **Build types** — the groupings offered when you build a drive (see
  [Building Install Drives](#building-install-drives)).
- **Catalog URLs** — the Apple software-update catalog index URLs makedrive
  consults to keep installer build numbers current.

**Where it lives.** makedrive.conf is stored at:

```
/Library/Application Support/makedrive/makedrive.conf
```

On launch, makedrive loads the configuration from next to the script if one is
present there; otherwise it loads it from Application Support. A makedrive.conf
found next to the script is migrated into Application Support (replacing any
existing copy) once makedrive has root. This makes updating simple: **ship a new
makedrive.conf alongside a new script, run it once, and the new configuration is
installed.**

When makedrive.conf is migrated, the previous copy is archived with a timestamp
in a **conf archive** subfolder of Application Support. makedrive keeps the 10
most recent archived copies, which gives you a safety net if a conf migration
produces unexpected results.

**Automatic version checking.** At launch, makedrive fetches a lightweight copy
of Apple's software-update catalog and compares it against makedrive.conf. When
Apple publishes a newer minor release for a macOS version you track, makedrive
updates the build number (and the version shown in menus and volume names) in
makedrive.conf automatically, and notes the change above the Main Menu.

---

## Adding Images to restorekit

makedrive adds images into restorekit from within the script, which is strongly
preferred over arranging the folder by hand. From the Main Menu choose option 1
to see the list of installers makedrive knows how to add:

```
Choose the software you'd like to add to the restorekit folder:

 1. 10.3.7 PPC Install        ...        26.x App Store Install
 2. 10.4.7 PPC Install                   27.x App Store Install
 ...
 X. Exit Image Installer
```

(The exact list is defined in makedrive.conf and reflects the macOS versions you
support.)

Depending on the image, makedrive either processes an App Store installer
already in `/Applications` (see [Mac App Store Installers](#mac-app-store-installers))
or prompts you to drag a pre-built DMG into the Terminal window. A progress
indicator is shown during the copy, and afterward makedrive ensures the DMG is
compressed and scanned for restore.

Before copying, makedrive verifies the source against the expected volume name
and macOS build number so the wrong image can't be added by mistake. If a build
number is left empty in makedrive.conf, the check is skipped for that entry.

---

## Mac App Store Installers

Beginning with OS X 10.7, Apple distributes macOS exclusively as an installer
application. From 10.9 onward the downloaded installer requires additional
processing to create a restorable image. makedrive handles this for you:

1. Download or place the macOS installer app into `/Applications`. If it launches
   automatically, quit it.
2. In makedrive (option 1), choose the matching installer entry.
3. makedrive runs `createinstallmedia`, builds the final DMG, applies the volume
   icon, and copies the result into restorekit.

makedrive removes the Mac App Store receipt from the processed installer so no
personally identifiable information ends up on deployed disks.

Two deploy types are supported, both handled automatically based on the entry's
configuration:

- **GENERIC** — a Mac App Store installer processed with `createinstallmedia`
  (Mojave and later use one flag set; El Capitan through High Sierra use
  another; makedrive selects the correct path).
- **INST‑A** — a pre-built or legacy DMG (for example an older retail install
  image) copied directly into restorekit.

---

## Building Install Drives

Main Menu option 3 builds a drive. First choose a build type:

```
 10. Apple Silicon & Intel Installers
 11. Older PPC & Intel 10.3-10.11 Installers
 12. DataDrive Only
  X. Return to the main menu
```

Build types and the volumes they contain are defined in makedrive.conf, so the
exact list reflects your configuration.

After choosing a build type you are asked whether the contents of restorekit
should also be copied to a **DataDrive** volume on the target. A populated
DataDrive doubles as a portable backup of restorekit: if the imaging host is ever
lost, restorekit can be rebuilt by copying the DataDrive folder structure into an
empty folder named `restorekit`.

makedrive then lists the disks attached to the Mac (`diskutil list`) and asks
which disk number to erase and restore onto:

```
Enter the disk number you would like to erase & restore. For example,
if the disk you want to erase is disk2, enter "2" and hit return.
```

makedrive will not let you select the disk you are currently booted from. On
Intel Macs the boot disk is identified with `bless`; on Apple Silicon the boot
volume is protected at the system level and makedrive recognizes it as well.

Once a valid disk number is entered, no further input is needed until imaging
finishes. `asr` decompresses and restores the images — this is CPU- and
I/O-intensive, so the host may be sluggish for other work meanwhile. When the
build completes, makedrive asks whether you'd like to image another disk.

---

## Restoring to USB or SD Card

Main Menu option 4 restores a single installer image to a USB flash drive, SD
card, or other small media in one step. Unlike option 3 (which builds a
full multi-installer drive with an optional DataDrive), this path formats the
entire target disk as a single volume and restores exactly one image onto it.
DataDrive is not copied.

This is the fastest way to hand someone a bootable installer for a specific
macOS version, or to quickly re-purpose a small drive without going through a
full build.

All configured image types appear in the chooser — both **GENERIC**
(createinstallmedia-based) and **INST-A** (pre-built or legacy) images. The same
boot-disk protection applies: makedrive will not let you erase the disk you are
currently running from.

When imaging is complete makedrive ejects the disk and asks whether you'd like to
restore another.

---

## Volume Icons

makedrive applies a custom volume icon to deployed install volumes so they are
easy to identify in the Startup Manager / boot picker. The icon is extracted
automatically from the macOS installer's application bundle when an image is
added — modern installers store it as `ProductPageIcon.icns` or
`InstallAssistant.icns`, while older installers (10.3 Panther through 10.6 Snow
Leopard) keep a differently named icon inside the bundle, which makedrive locates
as well. No separate icon download is required.

When an icon is applied, makedrive copies it to the volume as `.VolumeIcon.icns`
and sets the Finder custom-icon flag so the icon displays both in the boot picker
and when the volume is mounted.

---

## Pushover Notifications

makedrive can send a push notification (via [Pushover](https://pushover.net))
when a build finishes or fails, so you don't have to watch the screen. This
replaces the long-retired Boxcar notification support from older versions.

Configure it from Main Menu option 5:

- **Configure or update credentials** — enter your Pushover user key and
  application (API) token. They are stored in the macOS **System Keychain**, not
  in any plain-text file.
- **Send a test notification** — confirms your credentials work.
- **Remove credentials** — deletes the stored keys from the keychain; makedrive
  stops sending notifications.

Pushover is entirely optional. If no credentials are configured, makedrive
simply doesn't send notifications.

---

## Uninstalling makedrive

Main Menu option 6 removes makedrive from the host. To prevent accidents it
requires you to type `UNINSTALL` to confirm. When confirmed it removes:

- the `/Library/Application Support/makedrive` folder (including makedrive.conf
  and all conf archives),
- Pushover credentials from the System Keychain (if any are present), and
- the makedrive script itself.

restorekit and any drives you've already built are not touched.

---

## Security Considerations

makedrive's tools require root on the host, but because makedrive is an open
shell script, its behavior can be readily audited against your security policy.

makedrive makes network connections in two situations, both of which can be
avoided if you prefer an offline workflow:

1. **Version checking.** At launch makedrive fetches Apple's software-update
   catalog to compare installer build numbers. The catalog URLs are defined in
   makedrive.conf. A network failure here is non-fatal — makedrive simply leaves
   the configuration unchanged and continues.
2. **Pushover notifications.** Only if you configure them (Main Menu option 5),
   and only to send the notification you requested.

makedrive.conf is the configuration that drives partitioning and deployment; it
is controlled by the administrator who runs makedrive and is the intended trust
boundary. Pushover credentials are stored in the System Keychain rather than in
any file on disk.

---

## Changelog

Full release history is in [RELEASE_HISTORY.md](RELEASE_HISTORY.md).

### Build 201 — 2026-06-28

- **conf archive:** makedrive.conf migrations now archive the previous conf with
  a timestamp in a "conf archive" subfolder of Application Support, keeping the
  10 most recent copies as a safety net against unintended changes.
- **Restore to USB or SD card:** new Main Menu option 4 restores a single
  installer image to a USB drive, SD card, or other small media. The full disk
  is used for the install volume; DataDrive is not copied.
- **All image types in the quick-deploy chooser:** the image picker for option 4
  now includes both GENERIC (createinstallmedia) and INST-A (pre-built/legacy)
  images; previously only GENERIC images appeared.
- **Shared deploy pipeline:** restore, rename, bless, Spotlight-disable, and
  unmount are now handled by a single `deploy_restore_and_configure` function
  used by both the full drive build (option 3) and quick deploy (option 4),
  eliminating duplicated logic.
- **Consolidated createinstallmedia implementation:** the Mojave+ and El
  Capitan–High Sierra createinstallmedia paths share a single implementation with
  thin public wrappers, removing approximately 100 lines of duplication.
- **Pushover keychain helpers:** credential reads and deletes are now handled by
  dedicated helper functions rather than repeated inline across four Pushover
  functions.
- **`disp_pause_for_input` helper:** the "Hit enter to continue." prompt is now
  a single shared function, replacing approximately 20 scattered inline pairs
  throughout the script.
- **Two-column layout for quick-deploy image chooser:** the image picker for
  option 4 now uses the same two-column layout as the add-image menu (option 1).
- Dropped the daily build-sequence number from the version string; releases are
  now identified by date and build number alone.
- Cleaned up the script so it passes **ShellCheck** with no warnings: added `-r`
  to all `read` prompts, replaced indirect `$?` exit-status checks with direct
  command tests, and quoted variable expansions where word-splitting was
  unintended. No change to behavior.

### Build 200 — 2026-06-27 (first public release)

- Refocused makedrive as a macOS install/restore drive builder; the legacy
  diagnostic/triage subsystem (ASD/AXD downloads, diagnostic and triage build
  menus, icon-zip import) has been removed.
- Configuration externalized to **makedrive.conf**, stored in
  `/Library/Application Support/makedrive/`, with automatic migration of a
  conf placed next to the script.
- Added **Pushover** notifications (replacing the retired Boxcar service), with
  credentials stored in the System Keychain.
- Added **automatic version checking** against Apple's software-update catalog,
  keeping installer build numbers current in makedrive.conf.
- Added an **Uninstall** option to the Main Menu.
- Added installer **build-number verification** before adding to restorekit.
- Volume icons are now extracted automatically from the installer app bundle,
  including the differently located icon used by 10.3–10.6 installers.
- Boot-disk protection now covers **Apple Silicon** in addition to Intel.
- Installer coverage spans **10.3 Panther through the current macOS release.**

*For the complete release history going back to 2011, see [RELEASE_HISTORY.md](RELEASE_HISTORY.md).*

---

## Future Plans

- **A graphical interface.** A native GUI is desired and has been scoped, but it
  is not a current priority; makedrive ships as a Terminal script for now.
- **Saved custom configurations.** Defining build configurations outside the
  script (in makedrive.conf) is in place; further work to make custom
  configurations easier to carry across releases is planned.
- **DataDrive → restorekit recovery.** A function to rebuild a restorekit folder
  directly from a populated DataDrive volume, streamlining distribution and
  recovery of a makedrive installation.
