# Security Audit — Kodergamesense/Sc

**Date:** 2026-06-23  
**Scope:** Full repository scan (binary analysis, git history, repo hygiene)

---

## Executive Summary

This repository contains a single compiled Windows binary (`SoundCloudDesktop.exe`) — a WebView2-based SoundCloud desktop client with Discord Rich Presence integration. No source code is present. The audit was performed via static binary analysis (string extraction, PE header inspection, import table review, git history analysis).

**Critical findings: 3 | High: 3 | Medium: 3 | Low: 2**

---

## Findings

### CRITICAL

#### 1. PDB Path Leaks Developer Identity

The binary embeds the full PDB debug symbol path:

```
C:\Users\Crown\Documents\discover\build\x64\Release\SoundCloudDesktop.pdb
```

This reveals:
- Windows username: `Crown`
- Project directory structure: `Documents\discover\`
- Build configuration: `x64\Release`

**Risk:** Information disclosure. Attackers can use this for social engineering or targeted attacks.  
**Fix:** Strip PDB paths from release builds (`/PDBALTPATH:%_PDB%` linker flag or post-build strip).

---

#### 2. Binary Not Code-Signed

The PE certificate table is empty (`RVA=0x0, Size=0`). The binary has no Authenticode signature.

**Risk:** Users cannot verify the binary's authenticity or integrity. Windows SmartScreen will flag it. Supply-chain attacks are trivial — anyone can replace the `.exe` in the repo.  
**Fix:** Sign with a code-signing certificate before distribution.

---

#### 3. No Source Code — Binary-Only Distribution

The repository distributes only a compiled `.exe`. There is no way for users to:
- Audit the actual application logic
- Verify the binary matches any claimed source
- Build from source independently

**Risk:** Trust issue. Binary-only repos are a known vector for malware distribution.  
**Fix:** Publish the source code alongside the binary, or link to a source repo.

---

### HIGH

#### 4. URLDownloadToFileW — Unverified Downloads

The binary imports `URLDownloadToFileW` (from `urlmon.dll`) and downloads:
- `MicrosoftEdgeWebview2Setup.exe` — WebView2 runtime installer
- Favicon from `cdn.prod.website-files.com`
- Emoji assets from `cdn.jsdelivr.net`
- Profile avatars and track artwork from SoundCloud CDN

**Risk:** If any CDN is compromised, the app downloads and potentially executes malicious content (especially the WebView2 installer). No certificate pinning or hash verification was detected.  
**Fix:** Verify checksums/signatures of downloaded executables. Pin expected CDN domains.

---

#### 5. ShellExecuteW with `runas` — Silent Privilege Escalation

The binary uses `ShellExecuteExW` with `runas` verb and a `/silent /install` pattern, likely to install the WebView2 runtime with elevated privileges.

**Risk:** If the downloaded installer is tampered with, it executes with admin rights.  
**Fix:** Verify the WebView2 installer's Authenticode signature before executing it with elevation.

---

#### 6. Registry Autostart Persistence

The binary writes to:

```
Software\Microsoft\Windows\CurrentVersion\Run
```

This registers the app to launch automatically on Windows startup.

**Risk:** Autostart entries are a common persistence mechanism for malware. While this appears to be a legitimate feature (controlled by `autostart_enabled` config), it modifies the user's system without clear consent.  
**Fix:** Ensure autostart is opt-in (not default-on), and clearly disclose this behavior to users.

---

### MEDIUM

#### 7. JavaScript Injection into WebView2

The binary embeds substantial JavaScript that gets injected into the loaded SoundCloud page to:
- Scrape playback state (title, artist, artwork, play/pause status, timestamps)
- Extract user profile data (avatar URL, username) from `localStorage`, `window.__sc_hydration`, and DOM
- Scan all `window` keys matching `/^(__sc|SC|soundcloud|redux|store|user|session|account)/i`

**Risk:** This JavaScript has broad access to the loaded page's DOM and storage. If SoundCloud's page structure changes or if a malicious page is loaded, this could leak data.  
**Fix:** Restrict JavaScript injection to SoundCloud domains only (verify in `NavigationStarting` handler). Minimize data extraction scope.

---

#### 8. CryptProtectData / CryptUnprotectData Usage

The binary imports Windows DPAPI functions. This is used to encrypt/decrypt locally stored data (likely the config or cached credentials).

**Risk:** DPAPI is machine/user-scoped — any process running as the same Windows user can decrypt the data. This is not a strong protection for sensitive data.  
**Fix:** Document what data is encrypted. Consider additional encryption layers for sensitive data.

---

#### 9. Control Flow Guard (CFG) Disabled

PE DllCharacteristics: `0x8160`  
- ASLR: ENABLED  
- DEP: ENABLED  
- **CFG: DISABLED**

**Risk:** Without CFG, the binary is more vulnerable to control-flow hijacking exploits (ROP, JOP).  
**Fix:** Enable `/guard:cf` in the MSVC compiler/linker flags.

---

### LOW

#### 10. Old Binary Versions in Git History

Git history contains 3 different versions of the binary (143KB, 126KB, 312KB). Even though files were "deleted" between uploads, all versions remain in git's object store.

**Risk:** If any older version contained different secrets or vulnerabilities, they are still accessible via `git show <commit>:SoundCloudDesktop.exe`.  
**Fix:** If older versions contained sensitive data, use `git filter-repo` to permanently remove them.

---

#### 11. No `.gitignore` or Repository Hygiene

The repository has:
- No `.gitignore`
- No `README.md`
- No `LICENSE`
- No `SECURITY.md` (security policy)

**Risk:** Accidental commits of sensitive files (config files, debug builds, secrets) are likely without a `.gitignore`.  
**Fix:** Add `.gitignore`, `README.md`, and `SECURITY.md`.

---

## Checklist Summary

| Category                        | Status |
|---------------------------------|--------|
| Hardcoded API keys/secrets      | `SOUNDCLOUD_DISCORD_CLIENT_ID` read from env/config — OK design, but config file may store it in plaintext |
| SQL injection                   | N/A — no database code detected |
| Unvalidated user input          | JavaScript injection does not sanitize scraped DOM content before sending to native code |
| Insecure dependencies           | WebView2 runtime downloaded without integrity check |
| Overly permissive CORS          | N/A — desktop app, not a web server |
| Exposed debug endpoints         | PDB path leaks developer info |
| Missing authentication checks   | N/A — no server-side auth |

---

## Recommendations (Priority Order)

1. **Publish source code** or link to source repo
2. **Code-sign the binary** with a valid certificate
3. **Strip PDB paths** from release builds
4. **Enable CFG** (`/guard:cf`) in compiler settings
5. **Verify downloaded executables** before running them (check Authenticode signature)
6. **Add `.gitignore`** to prevent accidental commits
7. **Make autostart opt-in** with clear user disclosure
8. **Restrict JS injection** to SoundCloud domains only
9. **Clean git history** of old binary versions if they differ in security posture
