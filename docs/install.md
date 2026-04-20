---
layout: default
title: Install Guide
---

# Install Guide

Download the latest release from the [Releases page](../../releases/latest).

| Platform | File | 
|----------|------|
| Windows 10/11 (64-bit) | `Vektory-1.0.0.exe` 
| macOS 12+ (Intel + Apple Silicon) | `Vektory-1.0.0.dmg` 
---

## Windows

### Steps

1. Download `Vektory-1.0.0.exe` from the Releases page.
2. Run the installer. **Windows SmartScreen may show a warning** — see the note below.
3. The installer registers Vektory under Apps & Features and creates a Start Menu shortcut. No administrator rights are required for a per-user install.
4. Launch Vektory from the Start Menu.

### SmartScreen warning

When you run the installer, SmartScreen may show:

> "Windows protected your PC — Microsoft Defender SmartScreen prevented an unrecognized app from starting."

Click **"More info"** → **"Run anyway"**.

This warning appears because the binary is not signed with a paid Windows EV code-signing certificate. The binary is built from source using GitHub Actions — you can inspect the build log in the [Actions tab](../../actions) of this repository.

### Uninstall

Control Panel → Apps & Features → Vektory → Uninstall.

---

## macOS

### Requirements

- macOS 12 Monterey or later
- Intel or Apple Silicon (universal binary — one download works on both)

### Steps

1. Download `Vektory-1.0.0.dmg` from the Releases page.
2. Open the DMG and drag Vektory to your Applications folder.
3. The first time you open Vektory, **macOS Gatekeeper will block it** — see the steps below.

### Handling Gatekeeper (required on first launch)

Vektory is not notarised with an Apple Developer ID certificate. Gatekeeper will block the first launch. There are three ways to allow it:

**Method A — System Settings (macOS 13 Ventura and later):**

1. Try to open Vektory. The block dialog will appear.
2. Open **System Settings → Privacy & Security**.
3. Scroll to the Security section. You will see: *"Vektory was blocked from use because it is not from an identified developer."*
4. Click **"Open Anyway"**. Authenticate with Touch ID or your password if prompted.
5. Vektory opens. Subsequent launches work normally.

**Method B — Right-click (all macOS versions):**

1. In Finder, right-click (or Ctrl-click) Vektory in Applications.
2. Choose **"Open"** from the context menu.
3. A dialog asks whether you are sure — click **"Open"**.
4. Vektory launches. Subsequent launches work normally.

**Method C — Terminal (if the above methods do not work):**

```bash
xattr -d com.apple.quarantine /Applications/Vektory.app
```

Then open Vektory normally.

### Why Gatekeeper shows a warning

The binary is built on a GitHub Actions macOS runner and is not notarised. Notarisation requires a paid Apple Developer account ($99/year). You can inspect the build log in the [Actions tab](../../actions).
