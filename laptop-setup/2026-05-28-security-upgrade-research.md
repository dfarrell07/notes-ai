---
date: 2026-05-28T10:00:00-04:00
topic: laptop-setup
tags: [yubikey, password-manager, security, fido2, ssh, bitwarden, 1password, keepassxc, proton-pass]
---

# Workstation Security Upgrade Research (May 2026)

Two areas: YubiKey hardware and password manager (replacing LastPass).

---

## Part 1: YubiKey Current Generation

### Current Lineup

There is no YubiKey 6. Yubico has been iterating on the YubiKey 5 platform through firmware updates rather than releasing a next-generation numbered series. The current flagship is the **YubiKey 5 Series** running **firmware 5.7.x**.

**Product lines:**

| Series | Protocols | Use Case |
|--------|-----------|----------|
| YubiKey 5 | FIDO2, U2F, PIV, OpenPGP, OTP, OATH | Full-featured, developer/enterprise |
| YubiKey 5 FIPS | Same as 5, FIPS 140-3 validated | Government/regulated environments |
| YubiKey 5 Enhanced PIN | FIDO2/U2F only, pre-configured PIN policy | Enterprise passkey deployment |
| YubiKey Bio | FIDO2/U2F only, fingerprint sensor | Convenience (not more secure than 5) |
| Security Key | FIDO2/U2F only | Budget option, personal accounts |

### Firmware 5.7 Improvements (Over Older Firmware)

The 5.7 firmware is a major upgrade. Key changes:

**Expanded credential storage:**
- Passkey/resident key slots: **100** (up from 25)
- OATH seeds: **64** (up from 32)
- PIV certificates: **24**
- Total credentials: **190**
- This is the single biggest practical improvement for SSH resident keys

**New cryptographic algorithms:**
- RSA-3072 and RSA-4096 support
- Ed25519 and X25519 key types (native PIV/OpenPGP support)
- Aligns with 2023 DoD memo on stronger public key algorithms

**FIDO CTAP 2.1 improvements:**
- Force PIN Change capability
- Minimum PIN Length enforcement
- PIN Complexity (prevents simple patterns)
- Enterprise attestation (serial number readable during FIDO2 registration)

**Replaced Infineon cryptographic library** with Yubico's own (see vulnerability section below).

### Ed25519-sk SSH Support

- Ed25519-sk key pairs supported since firmware 5.2.3 (OpenSSH 8.2+)
- Firmware 5.7 does NOT change the ed25519-sk SSH behavior itself, but massively improves practicality
- With 100 resident key slots (up from 25), storing resident SSH keys is now practical without worrying about slot exhaustion
- `-O resident` keys stored on YubiKey enable portability across machines (require PIN)
- `-O verify-required` requires both PIN and physical touch per use
- Previously, non-resident keys were recommended to conserve the scarce 25 slots. That concern is now eliminated

### EUCLEAK Vulnerability (CVE-2024-45678) -- Reason to Upgrade

Discovered September 2024 by NinjaLab. This is a real vulnerability in all YubiKeys with firmware below 5.7.0.

**What it is:** Side-channel attack on the ECDSA implementation in Infineon's cryptographic library. Present for 14 years in the Infineon library. Allows extraction of ECDSA private keys.

**Affected:** All YubiKey 5 Series and Security Key Series with firmware prior to 5.7.0. Also all Infineon security microcontrollers running the Infineon library (TPMs, etc).

**Exploitability:** Requires physical possession of the YubiKey, specialized equipment, knowledge of target accounts, and technical skill. The extracted key only clones the YubiKey for the specifically targeted account, not all accounts.

**Fix:** Cannot be patched -- YubiKey firmware is not field-updatable. Yubico replaced the Infineon library with their own cryptographic library starting with firmware 5.7.0 (shipping since May 2024).

**Assessment:** Moderate severity for most users. The physical access requirement and specialized equipment make mass exploitation impractical. However, for a developer with SSH keys to production infrastructure, upgrading to 5.7+ firmware eliminates a known attack surface. If current YubiKeys are pre-5.7, this is a legitimate reason to replace them.

### Recommended Model: YubiKey 5C NFC

For USB-C laptop + NFC Android phone (Pixel), the **YubiKey 5C NFC** is the clear choice:
- USB-C plugs directly into laptop
- NFC tap-to-authenticate on Pixel phone
- Supports every protocol Yubico offers
- $58 USD -- same price as the USB-A NFC variant
- Firmware 5.7+ (all newly manufactured units)

Buy two: one primary, one backup. The backup prevents lockout if the primary is lost or damaged. Enroll both on all accounts.

### Price Comparison (All Current Models)

**YubiKey 5 Series (Standard):**

| Model | Connector | NFC | Price |
|-------|-----------|-----|-------|
| YubiKey 5C NFC | USB-C | Yes | $58 |
| YubiKey 5 NFC | USB-A | Yes | $58 |
| YubiKey 5C | USB-C | No | $65 |
| YubiKey 5 Nano | USB-A | No | $68 |
| YubiKey 5C Nano | USB-C | No | $68 |
| YubiKey 5Ci | USB-C + Lightning | No | $85 |

**YubiKey 5 FIPS Series:** $88-$115 (not needed unless compliance requires it)

**YubiKey Bio Series:** $98 (FIDO2 only, no NFC, no PIV/OpenPGP -- not recommended for developer use)

**Security Key Series:** $29 (FIDO2/U2F only -- too limited for developer use)

**Recommendation:** Two YubiKey 5C NFC = $116 total. Best value for the use case.

---

## Part 2: Password Manager -- LastPass Alternatives

### Why Leave LastPass

The 2022 breach consequences continue to mount in 2026:

- FBI confirmed in March 2026 that they linked a $150M cryptocurrency heist to stolen LastPass vault data. Total estimated losses from cracked vaults: over $438M
- UK ICO fined LastPass 1.2M GBP for "failing to implement sufficiently robust technical and security measures"
- $24.45M class action settlement still active through July 2026
- Law enforcement confirmed that cracked vaults continue to be exploited for cryptocurrency theft as of late 2025
- If you stored any credentials in LastPass before 2022 and have not rotated them, there is ongoing active risk

### Comparison of Alternatives

#### Bitwarden

**Security model:** AES-256 client-side encryption, zero-knowledge architecture. PBKDF2 with 600,000 iterations for key derivation. Server never sees master password or decrypted data.

**Breach history:** No direct vault breach ever. However, two notable 2025-2026 incidents:
- **ETH Zurich cryptographic audit (2025):** Researchers demonstrated 12 theoretical attacks against Bitwarden's encryption under a "malicious server" threat model. Bitwarden worked with the researchers to address findings. These were theoretical -- not exploited in the wild.
- **April 2026 npm supply chain attack:** Attackers hijacked the official `@bitwarden/cli` npm package for 90 minutes (version 2026.4.0). Malware harvested SSH keys, cloud credentials, and AI coding tool tokens from developer machines. **Vault data was NOT compromised** -- the malware targeted credentials stored outside the vault (SSH keys on disk, env vars, cloud configs). This is a supply chain attack on the distribution mechanism, not a Bitwarden security failure per se, but it is a real incident that affected real developers.

**Open source:** Yes. Client code and server code both open source. Independently audited annually (SOC 2 Type 2 and SOC 3 compliant).

**Self-hosted:** Yes, via [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (unofficial Rust reimplementation, 50 MB RAM vs Bitwarden's 2+ GB). All premium features free. Compatible with all official Bitwarden clients. One active maintainer is a Bitwarden employee. No formal third-party security audit of Vaultwarden itself.

**CLI:** Two CLIs: `bw` (password manager vault) and `bws` (secrets manager for infrastructure secrets). Both work on Linux.

**YubiKey/FIDO2:** FIDO2 WebAuthn for vault 2FA available on all plans including free. Up to 5 WebAuthn keys on free, 10 on paid.

**Linux support:** Native apps for all platforms. Browser extensions for Chrome. Flatpak and AppImage available. Fedora/RHEL support via Snap or direct download.

**Developer features:** Bitwarden Secrets Manager (separate product) for centralized API keys, DB credentials, SSH keys, certificates. Environment variable injection. The free personal vault tier is genuinely full-featured.

**Pricing:** Free tier (unlimited passwords, unlimited devices, TOTP, secure notes). Premium: $10/year. Family: $40/year (6 users).

**Verdict:** Best overall value. Open source, self-hostable, dirt cheap. The April 2026 supply chain attack is concerning for anyone who installs the CLI via npm, but the vault itself has never been breached. If self-hosting with Vaultwarden, you eliminate the cloud attack surface entirely.

---

#### 1Password

**Security model:** AES-256 encryption + unique 34-character Secret Key (generated on device, never transmitted to 1Password servers). Requires both master password AND Secret Key to decrypt vault. This is architecturally stronger than Bitwarden's master-password-only model.

**Breach history:** Never had a direct vault breach.
- **2023:** Indirectly affected by Okta support system breach. Detected suspicious activity on their Okta instance. Investigation found no user data accessed.
- **2024:** macOS vulnerability discovered (required attacker running malware on the same machine). Fixed in version 8.10.38. No evidence of exploitation.
- **2025:** Targeted phishing campaigns against users (not a 1Password breach -- external attackers sending fake breach alerts).

**Open source:** No. Closed source. Independently audited by Cure53, Recurity Labs, and others. ISO 27001/27017/27018/27701 certified. Bug bounty on HackerOne.

**Self-hosted:** No. Cloud-only.

**CLI:** `op` CLI tool. Biometric authentication support. `op run` injects secrets as environment variables. Shell plugins (as of April 2026, includes Claude Code integration via Touch ID). Well-documented, actively developed.

**YubiKey/FIDO2:** FIDO2 WebAuthn for vault 2FA. The SSH agent stores software-based keys in the vault; for YubiKey-backed ed25519-sk keys, only the public key is stored in 1Password (private key stays on YubiKey, OpenSSH communicates directly with the hardware).

**Linux support:** Native .rpm package for Fedora/RHEL. Automatic updates via yum repo. Active development with Wayland improvements, secure clipboard support. Latest stable: 8.12.20-7 (May 2026). Quality appears solid with regular releases.

**Developer features:** Built-in SSH agent (authorize connections with biometrics, private key never leaves 1Password). Git commit signing. `op run` for environment variable injection. Watchtower (password health monitoring). Travel Mode (temporarily remove sensitive vaults when crossing borders). Strong developer-focused tooling.

**Pricing:** $2.99/month individual, $4.99/month family (5 users). No free tier.

**Verdict:** Best developer experience and strongest security architecture (Secret Key + master password). SSH agent integration is best-in-class for software keys. Closed source and cloud-only are downsides. No self-hosting option. The Travel Mode feature is relevant given the migration plan's travel security section.

---

#### KeePassXC

**Security model:** Local-only database file (.kdbx). AES-256 or ChaCha20 encryption. Argon2id key derivation (stronger than PBKDF2 against GPU attacks). Database never contacts any server.

**Breach history:** No breaches possible in the traditional sense -- there is no server to breach. Attack surface is the local .kdbx file.

**Open source:** Yes. Fully FOSS (GPLv2/GPLv3).

**Self-hosted:** N/A -- it is inherently local. Sync via whatever file sync you choose (Syncthing, Nextcloud, manual copy).

**CLI:** `keepassxc-cli` for command-line access to the database.

**YubiKey support:** HMAC-SHA1 Challenge-Response mode only (not FIDO2). Requires configuring a YubiKey slot for HMAC-SHA1. KeePassXC uses the database master seed as the challenge, and the response enhances the encryption key. FIDO2 hmac-secret support is proposed and in progress but not yet merged.

**Linux support:** Excellent. Native Linux application, in Fedora repos (`dnf install keepassxc`). Desktop-only -- mobile requires companion apps (KeePassDX on Android, Strongbox on iOS) that read the same .kdbx format.

**Developer features:** Minimal. No SSH agent, no secrets injection, no environment variable integration. You copy-paste from the database. Browser extension (KeePassXC-Browser) works with Chrome.

**Pricing:** Free. Completely free and open source.

**Verdict:** Maximum local control and smallest attack surface. Best for paranoid threat models. Weakest on convenience, cross-device sync, and developer workflow integration. YubiKey support is limited to challenge-response (no FIDO2 yet). No mobile app from the project itself. Works well as a secondary vault for high-value secrets that should never touch the cloud, but inconvenient as a primary daily-driver password manager.

---

#### Proton Pass

**Security model:** AES-256 with zero-knowledge architecture. Encrypts everything including metadata (URLs, usernames) -- unlike most competitors. Client-side encryption/decryption only.

**Breach history:** No vault breach ever. Clean record across 10+ years of Proton services.
- **2025:** Browser extension memory handling issue (passwords not cleared from memory after locking). Fixed.
- **2025:** DEF CON 33 disclosure of DOM-based clickjacking vulnerability in browser extensions (affected multiple password managers, not just Proton Pass). Disclosed responsibly, fixed.
- **2026 Recurity Labs audit:** 8 issues found, all fixed and verified. "Overall security posture well above par." No remote exploits, no encryption bypasses.

**Open source:** Yes. Browser extensions, iOS app, and Android app all open source under GPLv3.

**CLI:** `proton-pass-cli` with URI-based secret access (`pass://vault/item/field`). SSH agent integration (load keys into SSH agent or run as standalone agent). Environment variable injection. Personal Access Tokens (PATs) for scoped script access. CLI requires a paid plan.

**YubiKey/FIDO2:** FIDO2 and U2F for account 2FA. Passkey-based account login supported. Proton Sentinel + passkeys + YubiKey is their recommended strongest configuration.

**Linux support:** Linux GUI app exists. Visual updates in progress to match other platforms. CLI works on Linux.

**Developer features:** CLI with secret management, SSH agent integration, environment variable injection, PATs for automation. Newer and less mature than 1Password or Bitwarden CLI tooling. Proton's 2026 roadmap focuses on folders, sharing improvements, and CLI expansion.

**Pricing:** Free tier (unlimited passwords, unlimited devices, TOTP, 10 email aliases). Pass Plus: $2.99/month ($35.88/year). Pass Family: $4.99/month (6 users). Proton Unlimited bundle: $9.99/month (includes Mail, Calendar, Drive, VPN, Pass, Wallet).

**Verdict:** Strong privacy-focused option with a clean security record. Full metadata encryption is a genuine differentiator. CLI is capable but newer/less battle-tested than Bitwarden or 1Password. The Proton ecosystem (Mail, VPN, Drive) is attractive if you want to consolidate privacy tools. Not self-hostable. Open source is a plus.

---

### Head-to-Head Comparison

| Feature | Bitwarden | 1Password | KeePassXC | Proton Pass |
|---------|-----------|-----------|-----------|-------------|
| **Price (individual/yr)** | Free / $10 | $36 | Free | Free / $36 |
| **Price (family/yr)** | $40 (6 users) | $60 (5 users) | Free | $60 (6 users) |
| **Open source** | Yes | No | Yes | Yes |
| **Self-hosted** | Yes (Vaultwarden) | No | Inherently local | No |
| **Vault breach history** | None | None | N/A | None |
| **Other incidents** | npm supply chain (2026) | Okta indirect (2023) | None | Extension memory (2025) |
| **CLI** | `bw` + `bws` | `op` (best) | `keepassxc-cli` | `proton-pass-cli` |
| **SSH agent** | No (Secrets Manager only) | Yes (built-in, excellent) | No | Yes (newer) |
| **SSH key storage** | Secure Notes | Native + agent | Database entries | CLI + agent |
| **YubiKey/FIDO2 2FA** | Yes (all plans) | Yes | HMAC-SHA1 only | Yes |
| **Linux quality** | Good | Good (active Fedora support) | Excellent (in repos) | Improving |
| **Browser extension** | Chrome, Firefox, etc | Chrome, Firefox, etc | Chrome, Firefox | Chrome, Firefox |
| **Env var injection** | Yes (`bws`) | Yes (`op run`) | No | Yes (CLI) |
| **Travel Mode** | No | Yes | N/A | No |
| **Metadata encryption** | Partial | Partial | Full (local) | Full |
| **Audit frequency** | Annual | Frequent (8+ firms) | Community review | Regular (Cure53, Recurity) |

### Recommendation

**Primary password manager: Bitwarden** (self-hosted via Vaultwarden or cloud)
- Open source, self-hostable, $10/year if using cloud
- FIDO2/YubiKey 2FA on all plans
- CLI for automation
- Best value by far
- Family plan for personal use ($40/year, 6 users)

**For SSH key management and developer secrets: 1Password** (complementary, not replacement)
- If budget allows, 1Password's SSH agent and `op run` integration are the best developer workflow tools
- The Secret Key architecture is genuinely stronger
- Travel Mode is relevant for the threat model in the migration plan
- $36/year individual

**Alternative: Proton Pass** if consolidating into the Proton ecosystem (Mail, VPN, etc)
- CLI is maturing rapidly
- Clean security record
- Full metadata encryption
- Consider if already using or planning to use Proton services

**KeePassXC** works well as a secondary offline vault for the highest-value secrets (recovery codes, master passwords, cryptocurrency keys) that should never touch any cloud service.

### Migration Priority

1. Stop using LastPass immediately if not already done
2. Export LastPass vault and import into new manager
3. Rotate ALL credentials that were stored in LastPass before 2022 -- the stolen vaults are actively being cracked
4. Enable YubiKey FIDO2 2FA on the new password manager
5. Set up CLI tooling for development workflows

---

## Sources

### YubiKey
- [Yubico Store / Compare Models](https://www.yubico.com/store/compare/)
- [YubiKey 5.7 Firmware Specifics](https://docs.yubico.com/hardware/yubikey/yk-tech-manual/5.7-firmware-specifics.html)
- [YubiKey 5.7 Firmware Announcement](https://www.yubico.com/blog/now-available-for-purchase-yubikey-5-series-and-security-key-series-with-new-5-7-firmware/)
- [FIPS 140-3 Validation Announcement](https://www.yubico.com/blog/yubikey-5-fips-series-is-now-fips-140-3-validated-what-it-means-for-high-assurance-security/)
- [EUCLEAK Vulnerability Advisory (YSA-2024-03)](https://www.yubico.com/support/security-advisories/ysa-2024-03/)
- [EUCLEAK Technical Paper (NinjaLab)](https://ninjalab.io/eucleak/)
- [EUCLEAK Coverage (Help Net Security)](https://www.helpnetsecurity.com/2024/09/04/yubico-security-keys-vulnerability/)
- [Securing SSH with FIDO2 (Yubico)](https://developers.yubico.com/SSH/Securing_SSH_with_FIDO2.html)
- [Best YubiKey 2026 (CriticNest)](https://criticnest.com/best-yubikey/)

### Password Managers
- [Bitwarden, KeePassXC, 1Password 2026 Comparison (dasroot.net)](https://dasroot.net/posts/2026/03/bitwarden-keepassxc-1password-password-manager-comparison/)
- [Best Password Manager 2026 (selfhosthero.com)](https://selfhosthero.com/bitwarden-vs-1password-vs-keepassxc-password-manager-showdown/)
- [1Password vs Bitwarden 2026 (CyberInsider)](https://cyberinsider.com/password-manager/comparison/1password-vs-bitwarden/)
- [LastPass Alternatives 2026 (Comparitech)](https://www.comparitech.com/password-managers/best-lastpass-alternatives/)
- [Bitwarden CLI Supply Chain Attack (Aikido)](https://www.aikido.dev/blog/shai-hulud-npm-bitwarden-cli-compromise)
- [Bitwarden CLI Attack Analysis (Endor Labs)](https://www.endorlabs.com/learn/shai-hulud-the-third-coming----inside-the-bitwarden-cli-2026-4-0-supply-chain-attack)
- [ETH Zurich Bitwarden Cryptography Audit](https://bitwarden.com/blog/security-through-transparency-eth-zurich-audits-bitwarden-cryptography/)
- [1Password Security Assessments](https://support.1password.com/security-assessments/)
- [1Password Okta Breach Impact](https://www.cybersecuritydive.com/news/1password-okta-breach/697636/)
- [1Password SSH Agent](https://developer.1password.com/docs/ssh/agent/)
- [1Password Linux Installation](https://support.1password.com/install-linux/)
- [KeePassXC FIDO2 hmac-secret Discussion](https://github.com/keepassxreboot/keepassxc/discussions/9506)
- [Proton Pass Security Audit 2026](https://proton.me/business/blog/proton-pass-audit-2026)
- [Proton Pass CLI](https://protonpass.github.io/pass-cli/)
- [Proton Pass 2026 Roadmap](https://proton.me/blog/pass-roadmap-spring-summer-2026)
- [Proton Pass Pricing](https://proton.me/pass/pricing)
- [Vaultwarden GitHub](https://github.com/dani-garcia/vaultwarden)
- [Vaultwarden Self-Hosting Guide 2026](https://aicybr.com/blog/vaultwarden-complete-self-hosting-guide)
