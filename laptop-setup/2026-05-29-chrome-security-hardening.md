---
date: 2026-05-29T10:00:00-04:00
topic: laptop-setup
tags: [chrome, browser-security, extensions, policies, ansible, gcp, jira, github]
---

# Chrome Browser Security Hardening for Developer Workstation

Threat model: developer workstation running Claude Code with Vertex AI (work) and personal Anthropic account. Machine has active sessions to Jira, GitHub, GCP Console, container registries, and SSH keys on disk. A compromised browser extension or misconfigured setting could exfiltrate credentials to all of these services.

---

## 1. Extension Management

### Specify an allowlist and block everything else

For a machine with access to Jira, GitHub, GCP, container registries, and SSH keys, the strongest posture is an explicit allowlist with a default-deny for all other extensions.

**Recommended extensions (by Chrome Web Store ID):**

| Extension | ID | Purpose |
|-----------|----|---------|
| uBlock Origin | `cjpalhdlnbpafiamejdnhcphjbkeiagm` | Ad/tracker blocking |
| uBlock Origin Lite (MV3 fallback) | `ddkjiahejlhfcafbddmgiahcphecmpfh` | For when MV2 is deprecated |
| Bitwarden | `nngceckbapebfimnlniiiahkandclblb` | Password manager |

HTTPS Everywhere is no longer needed. Chrome enforces HTTPS-First mode natively.

### Chrome extensions are manageable via policy files on both Linux and macOS

**Linux:** Chrome reads JSON files from:
- `/etc/opt/chrome/policies/managed/` — enforced, user cannot override
- `/etc/opt/chrome/policies/recommended/` — defaults, user can override

**macOS:** Chrome reads managed preferences from:
- `/Library/Managed Preferences/com.google.Chrome.plist` — enforced
- Can also be deployed via `.mobileconfig` profiles
- Works without MDM enrollment

**Chromium** uses different paths: `/etc/chromium/policies/managed/` on Linux.

---

## 2. Chrome Enterprise Policies on a Personal Machine

**This works without enterprise enrollment, MDM, domain join, or any license.** Chrome simply reads JSON files from the policy directories if they exist. The user will see "Managed by your organization" in Chrome settings for enforced policies. This is expected and correct.

Chrome validates JSON strictly. A syntax error causes the entire policy file to be silently ignored, so validation matters. Test with `chrome://policy` after deployment.

---

## 3. Disabling Chrome's Built-In Password Manager

Chrome's password manager stores credentials in a SQLite database protected only by the OS keychain. Any extension with sufficient permissions, or malware with local access, can extract them. With Bitwarden + YubiKey for 2FA, the built-in manager adds risk with no benefit.

**Policies to set:**

| Policy | Value | Effect |
|--------|-------|--------|
| `PasswordManagerEnabled` | `false` | Disables password save/fill entirely |
| `AutofillCreditCardEnabled` | `false` | Stops offering to save credit cards |
| `AutofillAddressEnabled` | `false` | Stops offering to save addresses |
| `CredentialProviderPromoEnabled` | `false` | Suppresses "Use Chrome as your password manager" prompts |

---

## 4. Chrome Profiles — Work vs Personal

**Use separate profiles. This is one of the highest-value security measures.**

### What profile separation provides:
- Completely separate cookie jars (a compromised extension in one profile cannot read cookies from the other)
- Separate extension installations per profile
- Separate browsing history, saved passwords, autofill data
- Separate localStorage and IndexedDB
- Process-level isolation between profiles (each profile runs in its own browser process group)

### What profiles do NOT provide:
- Separate disk encryption (both profiles live on the same filesystem)
- Protection against kernel-level or OS-level compromise
- Protection against a malicious extension within the same profile

### Recommendation:
- **Work profile**: Jira, GitHub (work org), GCP Console, container registries, Vertex AI
- **Personal profile**: Personal browsing, personal GitHub, personal email

Chrome profiles cannot be created or enforced via policy. Document the setup in the Ansible playbook README and use a manual verification step.

---

## 5. Chrome Sync — What Gets Synced and Risks

### Chrome sync sends to Google's servers:
- Bookmarks
- Browsing history and open tabs
- Passwords (if built-in password manager is enabled)
- Autofill data
- **Extensions and extension settings**
- Chrome settings and preferences
- Theme

### Extension sync is a supply-chain risk

If Chrome sync is enabled and a compromised extension is installed on one device, it propagates to every device signed into that Google account. Conversely, if one device is compromised and an attacker installs a malicious extension, it syncs to all other devices.

### Recommendation: Disable sync for the Work profile

| Policy | Value | Effect |
|--------|-------|--------|
| `SyncDisabled` | `true` | Prevents all data from syncing |
| `BrowserSignin` | `1` | Allows signing in (needed for Google services) but does not sync |

---

## 6. Site Isolation / Process Isolation

### Enabled by default since Chrome 67 (2018)

Every site (scheme + eTLD+1) gets its own renderer process. `github.com`, `atlassian.net`, and `console.cloud.google.com` each run in separate processes by default.

### Additional hardening: Strict Origin Isolation

Goes further by isolating every origin (not just site), meaning `a.example.com` and `b.example.com` get separate processes.

| Policy | Value | Effect |
|--------|-------|--------|
| `SitePerProcess` | `true` | Prevents Chrome from relaxing site isolation under memory pressure |
| `IsolateOrigins` | sensitive origins listed | Adds origin-level granularity for critical sites |

Memory cost is approximately 30-100 MB per additional process. Acceptable on a developer workstation with 32+ GB RAM.

---

## 7. Chrome Flags for Security

### Via policy (stable, preferred over chrome://flags):

| Policy | Value | Effect |
|--------|-------|--------|
| `HttpsOnlyMode` | `"force_enabled"` | Auto-upgrades HTTP to HTTPS, warns on HTTP-only sites |
| `InsecurePrivateNetworkRequestsAllowed` | `false` | Blocks websites from making requests to RFC 1918 addresses (prevents CSRF against local services) |
| `DnsOverHttpsMode` | `"automatic"` | Enables DNS-over-HTTPS when the system DNS provider supports it |

### chrome://flags (experimental, may change between versions):

| Flag | Setting | Purpose |
|------|---------|---------|
| `#strict-origin-isolation` | Enabled | Origin-level process isolation (redundant if policy is set) |
| `#block-insecure-private-network-requests` | Enabled | Same as policy above, for when policy is not deployed |

### Flags to avoid:
- `#enable-quic` — QUIC/HTTP3 can bypass some network security tools; leave at default
- `#disable-web-security` — never enable; disables same-origin policy entirely

---

## 8. uBlock Origin

### Still the recommended ad/tracker blocker, with a Manifest V3 caveat

uBlock Origin (MV2) remains the most effective blocker. However, Chrome is deprecating Manifest V2 extensions. The timeline has been repeatedly delayed but MV2 will eventually stop working.

**uBlock Origin Lite** (MV3, ID `ddkjiahejlhfcafbddmgiahcphecmpfh`) is the replacement. It is less capable: blocking is based on static declarativeNetRequest rules with hard limits set by Chrome. Dynamic filtering is reduced.

**Current plan:**
- Install uBlock Origin (MV2) now; it still works as of May 2026
- Include uBlock Origin Lite in the extension allowlist as a fallback
- Monitor the MV2 deprecation timeline
- If MV2 is disabled, consider Firefox for security-sensitive browsing (Mozilla has committed to supporting MV2 indefinitely)

### Risks of uBlock Origin:
- Broad permissions are necessary for its function (reads all URLs, modifies network requests)
- Open source, heavily audited, maintained by gorhill
- Main risk is a supply-chain attack against the Chrome Web Store listing
- The ForceList policy mitigates this by pinning the extension ID

---

## 9. Chrome on RHEL CSB

### Installable, requires Google's third-party repo

```bash
# /etc/yum.repos.d/google-chrome.repo
[google-chrome]
name=google-chrome
baseurl=https://dl.google.com/linux/chrome/rpm/stable/x86_64
enabled=1
gpgcheck=1
gpgkey=https://dl.google.com/linux/linux_signing_key.pub
```

Then `dnf install google-chrome-stable`.

### CSB considerations:
- CIS benchmarks for RHEL do not address Chrome specifically (they cover OS-level controls)
- Adding a third-party repo is a deviation from minimal-software posture but standard for developer workstations
- GPG key verification (`gpgcheck=1`) ensures package integrity
- Chrome auto-updates via the repo; `dnf update` picks up new versions
- If the CSB profile restricts third-party repos, host the Chrome RPM in an internal mirror

### Alternative: Chromium from Fedora/EPEL repos
If Google's repo is not acceptable, Chromium is available from Fedora/EPEL. It lacks some proprietary codecs and does not auto-update from Google. Policy files go in `/etc/chromium/policies/` instead of `/etc/opt/chrome/policies/`.

---

## 10. What a Malicious Chrome Extension Can Access

This is the most critical question. **A single malicious extension with broad permissions can compromise every web service you are logged into.**

### Permission levels and access:

| Permission | What it grants |
|------------|----------------|
| `<all_urls>` or `*://*/*` | Read and modify content on every page. Inject scripts, read DOM, intercept form submissions |
| `cookies` | Read, modify, delete cookies for any site with host permissions. **Includes session cookies for Jira, GitHub, GCP Console** |
| `webRequest` + `webRequestBlocking` | Intercept, modify, redirect, block every HTTP request. Read Authorization headers, Bearer tokens, CSRF tokens |
| `tabs` | Read URLs and titles of all open tabs |
| `clipboardRead` | Read clipboard contents (could capture copied passwords, tokens, SSH keys) |
| `nativeMessaging` | Communicate with native applications on the host OS. Escape from browser sandbox |
| `debugger` | Attach Chrome DevTools protocol to any tab. Full access to page content, network, cookies |

### Attack scenarios specific to this workstation:

**Jira session hijack:** Extension with `cookies` + `<all_urls>` reads the Jira session cookie and exfiltrates it. Attacker gets full Jira access as you.

**GitHub token theft:** Extension reads `_gh_sess` and `user_session` cookies. Can also intercept GitHub API requests and steal OAuth tokens from Authorization headers.

**GCP credential theft:** GCP Console uses OAuth tokens. Extension with `webRequest` intercepts `Authorization: Bearer` header on requests to `*.googleapis.com` and steals the access token. Attacker gets your GCP permissions.

**Container registry tokens:** If you access a registry web UI, the session cookie/token is accessible. CLI tokens in `~/.docker/config.json` are not reachable from the browser, but if you paste a token into any web page, an extension with content script access can read it.

**SSH keys:** Extensions cannot read the filesystem directly. But if you paste an SSH private key anywhere in the browser, or a web application displays one, the extension can capture it.

**Cross-extension isolation:** Extensions are isolated from each other. One extension cannot read another extension's storage or inject code into another extension's pages. But they share cookie access for web pages.

### Key takeaway:
The extension allowlist (blocking all except known-good) is the single most important Chrome security measure for this workstation.

---

## Complete Chrome Policy JSON for Ansible Deployment

Deploy to `/etc/opt/chrome/policies/managed/security.json` on Linux.

```json
{
  "_comment": "Chrome security policy for developer workstation",

  "ExtensionInstallBlocklist": ["*"],
  "ExtensionInstallAllowlist": [
    "cjpalhdlnbpafiamejdnhcphjbkeiagm",
    "ddkjiahejlhfcafbddmgiahcphecmpfh",
    "nngceckbapebfimnlniiiahkandclblb"
  ],
  "ExtensionInstallForcelist": [
    "cjpalhdlnbpafiamejdnhcphjbkeiagm;https://clients2.google.com/service/update2/crx",
    "nngceckbapebfimnlniiiahkandclblb;https://clients2.google.com/service/update2/crx"
  ],

  "PasswordManagerEnabled": false,
  "AutofillCreditCardEnabled": false,
  "AutofillAddressEnabled": false,
  "CredentialProviderPromoEnabled": false,

  "SyncDisabled": true,
  "BrowserSignin": 1,

  "SitePerProcess": true,
  "IsolateOrigins": "github.com,*.atlassian.net,console.cloud.google.com,*.gcr.io,*.googleapis.com,*.redhat.com",

  "HttpsOnlyMode": "force_enabled",
  "InsecurePrivateNetworkRequestsAllowed": false,
  "DnsOverHttpsMode": "automatic",

  "SafeBrowsingProtectionLevel": 1,
  "SafeBrowsingExtendedReportingEnabled": false,

  "DefaultPopupsSetting": 2,
  "DefaultGeolocationSetting": 2,
  "DefaultNotificationsSetting": 2,
  "DefaultWebUsbGuardSetting": 2,
  "DefaultSerialGuardSetting": 2,

  "PromotionalTabsEnabled": false,
  "ShowAppsShortcutInBookmarkBar": false,
  "BookmarkBarEnabled": true,

  "MetricsReportingEnabled": false,
  "SpellcheckEnabled": true
}
```

### Policy notes:

- `_comment` fields are ignored by Chrome's JSON parser (it strips unknown keys)
- `SafeBrowsingProtectionLevel: 1` is standard protection (2 would be enhanced, which sends URLs to Google)
- `SafeBrowsingExtendedReportingEnabled: false` prevents sending detailed threat data to Google
- `Default*Setting: 2` blocks popups, geolocation, notifications, WebUSB, and serial port access by default (user can allow per-site)
- `MetricsReportingEnabled: false` disables sending usage statistics to Google

### Ansible task for deployment (Linux):

```yaml
- name: Create Chrome managed policies directory
  ansible.builtin.file:
    path: /etc/opt/chrome/policies/managed
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Deploy Chrome security policy
  ansible.builtin.copy:
    src: files/chrome-security-policy.json
    dest: /etc/opt/chrome/policies/managed/security.json
    owner: root
    group: root
    mode: '0644'
    validate: "python3 -c \"import json; json.load(open('%s'))\""

- name: Add Google Chrome repository
  ansible.builtin.yum_repository:
    name: google-chrome
    description: google-chrome
    baseurl: https://dl.google.com/linux/chrome/rpm/stable/x86_64
    enabled: true
    gpgcheck: true
    gpgkey: https://dl.google.com/linux/linux_signing_key.pub

- name: Install Google Chrome
  ansible.builtin.dnf:
    name: google-chrome-stable
    state: present
```

### Verification after deployment:

1. Open Chrome and navigate to `chrome://policy`
2. All policies from the JSON file should appear with status "OK" and source "Platform"
3. Navigate to `chrome://extensions` — only allowlisted extensions should be installable
4. Navigate to Settings > Passwords — password manager should show as disabled by policy
5. Navigate to Settings > Sync — sync should show as disabled by policy

---

## Summary of Priorities (Highest Impact First)

1. **Extension allowlist with default-deny** — blocks the primary attack vector for credential theft
2. **Separate Chrome profiles for work and personal** — isolates cookie jars and sessions
3. **Disable Chrome password manager** — use Bitwarden with YubiKey 2FA instead
4. **Disable Chrome sync** — prevents extension/credential propagation to other devices
5. **Enforce HTTPS-only mode** — prevents downgrade attacks
6. **Site isolation hardening** — defense-in-depth for process isolation
7. **Block private network requests** — prevents CSRF against local services
8. **Disable telemetry** — reduces data sent to Google
