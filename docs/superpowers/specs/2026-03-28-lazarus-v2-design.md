# Lazarus v2 - Deterministic TOTP Recovery Tool

## Context

Lazarus v1 is a Shamir 2-of-3 secret sharing recovery portal hosted on GitHub Pages. It works but violates KISS: three hosting providers with shifted shares, dual PINs, GF(256) Lagrange interpolation, separate JSON share files. The cognitive load under stress is high.

The core problem it solves: recovering access to a Vaultwarden password vault when you have no phone, no hardware security keys, and are away from home. The only thing you have is your brain and a borrowed device.

Lazarus v2 replaces the Shamir approach with deterministic TOTP derivation. One memorised master password + a site name deterministically generates a TOTP code that serves as the 2FA factor for Vaultwarden login. Combined with the memorised vault master password, this provides full vault access from any browser anywhere.

## Scope

- Rebuild the Lazarus `index.html` as a deterministic TOTP generator
- Remove all Shamir/share/PIN/essentials/lifeboat code
- Single HTML file, zero external dependencies
- Host on GitHub Pages (thoroslives.github.io/lazarus/)

Out of scope (separate projects):
- Vaultwarden SSO via Authentik
- VPS Vaultwarden replica
- Updating recovery documentation in the brain vault

## Crypto Specification

These parameters are immutable once deployed. Changing any value produces a different seed, breaking registered TOTP secrets.

```
Version:      v1 (embedded in salt prefix)
Algorithm:    PBKDF2-HMAC-SHA256
Password:     UTF-8 bytes of master password (NFC normalized, case-sensitive, no trimming)
Salt:         "v1" + uint32_be(len(name)) + name + uint32_be(len(site)) + site
              where name and site are NFC normalized, lowercased, trimmed UTF-8 bytes
              and len() is byte length, not character count
Iterations:   1,000,000
Key length:   20 bytes (160 bits)
Encoding:     Base32 (RFC 4648, uppercase, no padding)
TOTP:         HMAC-SHA1, 6 digits, 30-second period, T0=0 (RFC 6238 defaults)
```

The salt uses length-prefixed fields (Spectre/MasterPassword pattern) to prevent boundary
ambiguity attacks. The version prefix enables future algorithm upgrades without silent breakage.

All inputs are Unicode NFC normalized before processing to ensure cross-platform consistency.

All crypto uses the browser-native WebCrypto API (`crypto.subtle`). No external libraries.

## UI Design

### Layout

Minimal, clean, no branding. Dark or neutral theme. Mobile-friendly.

```
+------------------------------------------+
|  Recovery                                |
|  All crypto runs locally.                |
|  Nothing uploads.                        |
|                                          |
|  Master password: [__________________]   |
|  Service:         [__________________]   |
|                                          |
|  [ Derive ]                              |
|                                          |
|  +------------------------------------+  |
|  |        123 456         (24s)       |  |
|  +------------------------------------+  |
|                                          |
|  [Show seed]                             |
|                                          |
+------------------------------------------+
```

### Elements

- **Master password field**: `type="password"`, no placeholder, no autocomplete
- **Identicon**: deterministic visual hash of master password, updates on input, for typo detection
- **Name field**: `type="text"`, no placeholder, no autocomplete (user's full name, anti-precomputation salt)
- **Site field**: `type="text"`, no placeholder, no autocomplete, no default value (bare domain)
- **Derive button**: triggers KDF + TOTP generation
- **TOTP code display**: large monospace font, space-separated groups of 3 (e.g. "123 456"), hidden until derived
- **Countdown**: seconds remaining in current 30s window, displayed next to the code
- **Show seed toggle**: reveals the base32 seed string below the code, copyable. Hidden by default.
- **Spec footer**: small text showing the crypto parameters (algorithm, iterations, key length) so the derivation is reproducible even without the page

### Behaviour

1. User enters master password and service name
2. Clicks "Derive" (or presses Enter)
3. Page runs PBKDF2 (takes ~2-3 seconds), shows a spinner/progress indicator
4. Displays the 6-digit TOTP code with countdown timer
5. Code auto-refreshes every 30 seconds
6. "Show seed" toggle reveals the base32 seed for initial registration

### What the page does NOT do

- No state persistence (no cookies, no localStorage, no sessionStorage)
- No clipboard API (user manually selects/copies)
- No analytics, no tracking, no external requests
- No hints about what services might be used (no placeholders, no defaults, no suggestions)

## Security

### Content Security Policy

```
default-src 'self';
script-src 'self' 'unsafe-inline';
style-src 'self' 'unsafe-inline';
img-src 'self' data:;
connect-src 'none';
base-uri 'none';
form-action 'self';
```

Note: `connect-src 'none'` - the page makes zero network requests after loading. This is stricter than v1 which allowed HTTPS fetches for share files.

### Additional Measures

- Incognito window recommendation displayed prominently
- Password field value not readable via DOM inspection after derivation (clear the input after use)
- Page clears all derived data on visibility change (tab switch/minimise) or after 5 minutes of inactivity
- The crypto spec is displayed on the page so the tool is reproducible without the source code

### Threat Model

- **Attacker knows the page exists**: they still need the master password and the exact service name
- **Attacker has the master password**: they still need the Vaultwarden master password (different secret) to decrypt the vault
- **Shoulder surfing**: the TOTP code is visible for 30 seconds. Same risk as any authenticator app.
- **Borrowed device is compromised**: keylogger captures master password. Mitigated by incognito recommendation and the fact that TOTP alone doesn't grant vault access.

## Registration Workflow (One-Time Per Service)

1. Open Lazarus v2 from any browser
2. Enter master password + service name (e.g. "vaultwarden")
3. Click "Show seed" to reveal the base32 string
4. In Vaultwarden: Settings -> Security -> Two-step login -> Authenticator app
5. Choose "Enter key manually", paste the derived seed
6. Verify by entering the current TOTP code from Lazarus
7. Save. The derived seed is now registered as a TOTP method alongside YubiKey.

## Recovery Workflow

Lost phone, no YubiKey, anywhere in the world:

1. Open any browser, navigate to thoroslives.github.io/lazarus/
2. Enter master password + "vaultwarden" (or whatever service name was registered)
3. Copy the displayed TOTP code
4. Navigate to northwarden.north.cx
5. Enter vault email + master password
6. When prompted for 2FA, choose authenticator and enter the TOTP code
7. Vault unlocked. Retrieve any other TOTP codes needed to cascade into other services.

## Files Changed

- `/workspace/repos/lazarus/index.html` - complete rewrite
- `/workspace/repos/lazarus/essentials.enc.json` - delete
- `/workspace/repos/lazarus/share2.enc.json` - delete
- `/workspace/repos/lazarus/README.md` - update (keep vague, no details about what the tool does)

## Testing

1. **Determinism test**: derive seed for same inputs twice, verify identical output
2. **Cross-browser test**: verify same inputs produce same TOTP code in Chrome, Firefox, Safari
3. **TOTP validation**: register derived seed in Vaultwarden, verify codes match
4. **Offline test**: save page locally, disconnect internet, verify it still works
5. **Mobile test**: verify the page works on a mobile browser (borrowed phone scenario)
6. **Timer test**: verify code refreshes correctly at 30-second boundaries
7. **Edge cases**: empty inputs, very long inputs, unicode characters, leading/trailing spaces (should be trimmed)
