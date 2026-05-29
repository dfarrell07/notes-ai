---
date: 2026-05-29T10:00:00-04:00
topic: laptop-setup
tags: [pixel, android, grapheneos, mobile-security, phone-hardening, threat-model]
---

# Pixel Phone Security Hardening for Developer Workstation

Research on securing a Pixel phone used as the mobile side of a developer
workstation. The phone runs GitHub Mobile, Claude Mobile, and voice dictation
for task management, alongside normal personal use. A fine-grained GitHub PAT
scoped to a single repo is present on the device.

## Threat Model

- **Device extraction**: Cellebrite, GrayKey, or border search (CBP authority
  to search devices without warrant at borders)
- **Lost or stolen device**: Opportunistic access to GitHub tokens, Claude
  sessions, and personal data
- **App-to-app data leakage**: One compromised app accessing another's data

## 1. GrapheneOS vs Stock Android

### What GrapheneOS Adds

GrapheneOS is a hardened Android fork exclusively for Pixel devices (~400K
active users as of April 2026). Key security improvements over stock Android:

- **hardened_malloc**: Custom heap allocator with out-of-line metadata,
  zero-on-free, randomized delayed reuse, and slab quarantines
- **Memory Tagging Extension (MTE)**: Uses ARMv8.5-A hardware memory tagging
  on supported Pixels for probabilistic detection of use-after-free and buffer
  overflows
- **Per-app network toggle**: Block any app from internet access entirely
  (stock Android cannot do this without a VPN-based firewall)
- **Storage Scopes**: Apps only see files you explicitly share, not broad
  storage access
- **Auto-reboot timer**: Reboots locked devices after configurable period
  (default 18 hours, range 10 min to 72 hours), putting data into Before First
  Unlock (BFU) state where encryption keys are not in memory
- **USB restrictions**: Blocks new USB connections when device is locked
- **Per-connection MAC randomization**: Prevents tracking across Wi-Fi
  networks (stock Android only randomizes per-network)
- **Duress PIN/password**: Enter a secondary PIN to silently wipe the device
  (see section 8)

### What GrapheneOS Breaks or Complicates

- **Google Pay**: Does not work (no privileged Play Services)
- **Android Auto**: Does not work
- **Google One backups**: Not available
- **Banking apps**: Most work with Sandboxed Google Play, but some fail
  Play Integrity/SafetyNet attestation checks. GrapheneOS passes
  `basicIntegrity` but fails `ctsProfileMatch`. Check the PrivSec.dev
  compatibility list for your specific bank before committing
- **Push notifications**: Work via Sandboxed Google Play + Firebase Cloud
  Messaging, but some users report delayed or missing notifications for
  Signal, Proton Mail, Discord. Grant battery optimization exception to Google
  Play Services to reduce issues
- **DRM-protected content**: Some streaming apps may not work
- **Pixel-specific AI features**: Camera processing, call screening, and other
  Google AI features are absent or reduced

### Sandboxed Google Play (The Compatibility Layer)

GrapheneOS can install Google Play Services as a regular sandboxed app without
system privileges. This preserves hardening while enabling most Play-dependent
apps. Best practice: install Play Services in a dedicated user profile so
Google has no visibility into your main profile's data.

### Is It Worth It for a Developer?

For a developer who uses GitHub Mobile, Claude, and voice dictation: probably
yes, if you are willing to spend a few hours on initial setup and can tolerate
occasional notification delays. The auto-reboot timer and USB restrictions
alone provide significant protection against the device extraction threat
model. The duress PIN is unique to GrapheneOS and unavailable on stock Android.

If banking app compatibility is a hard requirement, test your specific bank
app before committing. If Google Pay is essential, GrapheneOS is not viable.

## 2. Stock Android Hardening (If Staying on Stock)

If GrapheneOS is not practical, these settings should be changed on a stock
Pixel.

### Critical Settings

| Setting | Location | Action |
| --- | --- | --- |
| USB debugging | Developer Options | **Off** (enable only temporarily when needed, disable within 60 seconds of unplugging) |
| OEM unlocking | Developer Options | **Off** (disables Verified Boot if on) |
| Developer Options | Settings > About Phone | Disable entirely after use (7-tap to enable, toggle off when done) |
| Install unknown apps | Settings > Apps > Special app access | **Deny all** sources |
| Smart Lock | Settings > Security > Smart Lock | **Disable all** (Trusted places, Trusted devices, On-body detection) |
| Lock screen notifications | Settings > Notifications | **Hide sensitive content** or show none |
| USB default config | Developer Options > Default USB configuration | **No data transfer** (charge only) |
| Revoke USB debug authorizations | Developer Options | Do this periodically to clear trusted computers |
| Auto-rotate | Quick settings | Personal preference, but disable if device is used in public |

### Biometric vs PIN

- Use a **6+ digit PIN or alphanumeric password** as the primary unlock method
- Fingerprint is acceptable for convenience but understand the legal
  distinction: courts have generally held that PINs are protected testimonial
  knowledge (Fifth Amendment), while biometrics can be compelled
- **Always use Lockdown Mode before border crossings** (disables biometrics,
  forces PIN-only unlock)

### Encryption Verification

All Pixels with Android 10+ use file-based encryption (FBE) by default. To
verify: Settings > Security > Encryption & credentials. Should show
"Encrypted." This is automatic and cannot be disabled on modern Pixels.

### Android 16 Advanced Protection Mode

Available on Pixel 8 and later running Android 16. Single toggle in
Settings > Security & Privacy > Advanced Protection. Enables:

- **USB Protection**: Hardware-level USB data blocking when locked (charging
  still works)
- **Inactivity Reboot**: Restarts after 72 hours locked (puts device in BFU
  state)
- **Intrusion Logging**: Privacy-preserving forensic logs (developed with
  Amnesty International)
- **Sideloading blocked**: Cannot install apps from unknown sources
- **2G network disabled**: Prevents downgrade attacks
- **Insecure Wi-Fi blocked**: No auto-connect to unsecured networks

Trade-offs: Chrome JavaScript optimizer disabled (some sites may break),
no sideloading, possible false positives in call screening.

**This is the single most impactful setting for stock Android users who want
better security without switching to GrapheneOS.**

## 3. App Isolation: Work Profile

### Android Work Profile

Android's Work Profile creates an OS-level container with separate encryption
keys, isolated process space, and independent data storage. Apps in the work
profile cannot access personal profile contacts, photos, messages, or files
(enforced at the kernel level, not application level).

### How to Set It Up Without an Employer

Use **Shelter** (free, open-source app on F-Droid): creates a Work Profile
without requiring corporate MDM enrollment. Install GitHub Mobile and Claude
in the work profile, personal apps in the main profile.

### What Isolation Provides

- GitHub Mobile in the work profile cannot see Claude data in the personal
  profile (or vice versa)
- Work profile has its own clipboard, file storage, and app list
- Apps appear with a briefcase icon to distinguish them
- Work profile can be paused/disabled entirely (stops all work apps from
  running, equivalent to uninstalling them temporarily)
- On Android 11+, work profile apps cannot access SMS/MMS from personal
  profile

### GrapheneOS Alternative: Multiple User Profiles

GrapheneOS supports multiple user profiles with stronger isolation than
Work Profiles. Each profile has separate encryption keys. Install Sandboxed
Google Play in one profile, keep another profile Google-free. Data is fully
isolated between profiles.

### Recommended Setup

- **Personal profile**: Calls, messaging, personal browsing, banking
- **Work profile (via Shelter) or separate user profile (GrapheneOS)**:
  GitHub Mobile, Claude Mobile, any work-related apps

## 4. Lockdown Mode (Disable Biometrics)

Android has had Lockdown Mode since Android 9 Pie. On Pixel:

### How to Activate

1. Press and hold **Power + Volume Up**
2. Tap **Lockdown** in the power menu

### What It Does

- Disables fingerprint and face unlock
- Forces PIN/password/pattern-only unlock
- Hides all lock screen notifications
- Disables Smart Lock

### Important Behavior

- **One-time use**: After you unlock with your PIN, biometrics are re-enabled.
  You must activate Lockdown again before each border crossing or risk
  scenario
- **Does not wipe data**: Only restricts unlock method
- **Does not power off**: Phone stays on, just locked

### When to Use

- Before any border crossing or checkpoint
- Before handing device to anyone
- Before going through airport security
- When entering a protest or high-risk environment

**Combine with**: Disable Smart Lock (Trusted places, Trusted devices,
On-body detection) in Settings so your phone does not auto-unlock in
"trusted" locations.

## 5. USB Data Protection

### The Threat

Cellebrite, GrayKey, and similar forensic tools use USB data connections to
extract device data. They need a USB data channel, not just power.

### Stock Android Options

- **Default USB configuration**: Set to "No data transfer" in Developer
  Options. Phone will only charge when plugged in via USB
- **Android 16 Advanced Protection**: Hardware-level USB data blocking when
  device is locked. New USB connections can only charge. Even if you
  connected a data cable while unlocked, locking the screen disables the data
  connection after a few seconds
- **Physical option**: Use a "USB data blocker" (sometimes called a "USB
  condom") -- a small adapter that physically disconnects the data pins while
  passing power. About $5-10 on Amazon

### GrapheneOS Options

- Blocks all new USB connections when device is locked (default behavior)
- Combined with auto-reboot timer, even a seized device returns to BFU state
  within 18 hours, making extraction significantly harder

### Important Caveats

- USB protection only applies when the screen is locked. If you connect USB
  while unlocked, data transfer is possible
- Some proprietary fast charging protocols use data lines; USB protection may
  limit you to standard charging speeds when locked
- First boot is not protected (USB remains open until boot completes)

## 6. Auto-Wipe After Failed Unlock Attempts

### Stock Android / Pixel

**Stock Pixel does not have a native auto-wipe feature.** After failed
attempts, the device applies progressive lockout timers:

- 5 failed attempts: 30-second lockout
- 10 failed attempts: 30-minute lockout, may require Google account
  verification
- The device never auto-wipes on stock Android

Samsung devices have a built-in "Auto factory reset" after 15 failed
attempts, but Pixels do not.

### GrapheneOS

GrapheneOS also does not currently have auto-wipe after failed attempts
(it is an open feature request, GitHub issue #4512). However, it does have:

- **Stricter fingerprint limits**: Only 5 total attempts (stock Android
  allows 20 with delays)
- **Auto-reboot timer**: Configurable 10 min to 72 hours (default 18 hours).
  After reboot, device is in BFU state with encryption keys not in memory.
  This is not a wipe, but it makes extraction much harder
- **Duress PIN**: Manual wipe trigger (see section 8)
- **Open feature request**: Auto-reboot after N wrong PIN/password entries
  (GitHub issue #5867, filed July 2025)

### Enterprise/MDM Option

If the phone is enrolled in an MDM (Mobile Device Management) solution, most
MDM platforms can enforce auto-wipe after N failed attempts. This applies to
both stock Android and GrapheneOS.

## 7. GitHub PAT Storage

### How Android Stores Sensitive Credentials

Android provides the Keystore system for storing cryptographic keys in
hardware-backed secure elements:

- **Trusted Execution Environment (TEE)**: Isolated processor environment
  where keys are stored and crypto operations happen. Key material never
  leaves the TEE
- **StrongBox** (Android 9+): Even stronger isolation using a dedicated
  Secure Element chip, separate from the main processor. Pixel phones
  support StrongBox via the Titan M security chip
- Hardware-backed keystore has been mandatory for devices shipping with
  Android 7.0+

### Where GitHub Mobile Likely Stores the PAT

GitHub has not publicly documented the exact storage mechanism for the
Android app's authentication tokens. However, well-written Android apps
use the Android Keystore (hardware-backed on Pixels via Titan M) or
EncryptedSharedPreferences. The authentication token is likely:

- Encrypted using a key stored in the hardware-backed Keystore
- Accessible only to the GitHub app's process (Android app sandboxing)
- Protected by Verified Boot (if the OS is tampered with, the Keystore
  can be invalidated)

### What This Means for the Threat Model

- **Device locked (BFU state)**: Keystore keys are not available. Token
  cannot be decrypted even with physical access
- **Device locked (AFU state, after first unlock)**: Keystore keys may be
  in memory depending on key configuration. Auto-reboot (GrapheneOS 18h
  default, Android 16 Advanced Protection 72h) returns to BFU
- **Device unlocked**: Token is accessible to the GitHub app. An attacker
  with an unlocked device can use the app normally
- **Rooted device / compromised OS**: Keys cannot be extracted from
  hardware-backed Keystore even with root, but they can be used locally
  while the device is compromised

### Mitigation

- Use a fine-grained PAT scoped to a single repo (already done)
- Set short expiration on the PAT (90 days or less)
- If device is lost or seized, revoke the PAT immediately from GitHub
  Settings on another device
- The PAT can only do what its scopes allow, so a stolen token from a
  single-repo-scoped PAT has very limited blast radius

## 8. Duress PIN / Password

### Stock Android

**Stock Android does not have a duress PIN feature.** No mechanism exists to
trigger a wipe from the lock screen on unmodified Android.

### GrapheneOS Duress Feature

GrapheneOS has a full duress PIN/password feature:

- **Setup**: Settings > Security & Privacy > Device Unlock > Duress Password
- **Requires both**: A duress PIN and a duress password must be set (to
  account for profiles using different unlock methods)
- **What happens when entered**: Silent, irreversible factory reset begins
  in the background. Erases all encryption keys and eSIM partition. No
  confirmation prompt, no visible indication that a wipe is occurring.
  Phone appears to restart normally into factory-fresh state
- **Where it works**: Lock screen, Developer Options unlock prompt, any
  app authentication prompt
- **Wipes all profiles**: All user profiles (work, personal) are wiped
- **Cannot be interrupted**: Once started, the wipe runs to completion
- **eSIM erased**: Unlike a normal factory reset, the duress wipe also
  destroys the eSIM

### Practical Considerations

- Set the duress PIN to something plausible but distinct from your real PIN.
  Some community members suggest using obvious sequences like 0000 or 1234,
  since an amateur attacker would try those first
- **Legal risk**: Using a duress PIN could be considered obstruction of
  justice or destruction of evidence in some jurisdictions. Understand the
  legal implications before relying on this
- The duress PIN is the primary reason some developers choose GrapheneOS
  over stock Android for the border crossing threat model

## 9. Practical Recommendation for a Developer

The right level of hardening depends on how often you cross borders and how
sensitive the data on the phone is.

### Minimum Viable Hardening (Stock Android, 30 Minutes)

These settings cost nothing and provide meaningful protection against lost
or stolen device:

1. Enable **Android 16 Advanced Protection Mode** (if on Pixel 8+ with
   Android 16)
2. Set a **6+ digit PIN** (not 4 digits, not a pattern)
3. Disable **Smart Lock** entirely
4. Set **USB default to "No data transfer"** in Developer Options
5. Hide **sensitive notifications** on lock screen
6. Disable **USB debugging** and **OEM unlocking**
7. Block **unknown app sources**
8. Learn the **Lockdown Mode** gesture (Power + Volume Up > Lockdown) and
   use it before any border crossing
9. Set up a **Work Profile via Shelter** for GitHub and Claude apps

### Recommended Hardening (GrapheneOS, 2-3 Hours Initial Setup)

For a developer handling GitHub PATs and work sessions on mobile, who
crosses international borders:

1. Install GrapheneOS on Pixel
2. Set up **Sandboxed Google Play** in a dedicated profile
3. Configure **auto-reboot timer** (consider 8-12 hours instead of default
   18 if border crossings are a concern)
4. Set up **duress PIN/password**
5. Install GitHub Mobile and Claude in an **isolated user profile**
6. Keep personal apps (banking, messaging) in a separate profile
7. Set a **strong alphanumeric password** (not just a PIN)
8. Test all critical apps (banking, GitHub, Claude, notifications) before
   relying on the setup

### What Not to Bother With

- **Faraday bags**: Overkill for a developer, and you lose all connectivity
- **Burner phones for travel**: Unnecessary if you have proper lockdown
  mode + duress PIN + BFU via auto-reboot
- **Disabling all biometrics permanently**: Fingerprint is fine for daily
  use; just use Lockdown Mode when the threat is active
- **Custom ROM other than GrapheneOS**: LineageOS, CalyxOS, etc. provide
  weaker security than GrapheneOS and are not recommended for this threat
  model

### Decision Matrix

| Threat | Stock Android + Hardening | GrapheneOS |
| --- | --- | --- |
| Lost/stolen device | Good (encryption + PIN + FRP) | Better (auto-reboot to BFU, stricter lockouts) |
| Border search | Adequate (Lockdown Mode + USB protection) | Strong (duress PIN + auto-reboot + USB blocking) |
| Cellebrite extraction | Good with Android 16 Advanced Protection | Better (hardware USB blocking when locked) |
| App-to-app leakage | Good (Work Profile via Shelter) | Better (full user profile isolation) |
| Remote compromise | Adequate (standard Android sandboxing) | Better (hardened_malloc, MTE, network toggles) |
| Convenience | High | Moderate (notification delays, some app issues) |

### Bottom Line

For a developer who is not a journalist or activist: **stock Android with
Advanced Protection Mode enabled and the "Minimum Viable Hardening" list
above is probably sufficient.** It takes 30 minutes and covers the realistic
threats.

If you cross borders frequently, or if the GitHub PAT gives access to
anything particularly sensitive (even scoped to one repo, if that repo
contains proprietary code), GrapheneOS is worth the 2-3 hour investment.
The duress PIN and aggressive auto-reboot are the two features that stock
Android simply cannot match.
