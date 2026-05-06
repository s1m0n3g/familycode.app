<div align="center">

# FamilyCode 🔐

**Verify who's really calling you.**

A zero-server, zero-signup OTP webapp to defeat vishing scams and audio deepfakes.

[![License: PolyForm Noncommercial](https://img.shields.io/badge/License-PolyForm%20NC%201.0.0-blueviolet)](LICENSE)
[![Single File](https://img.shields.io/badge/Architecture-Single%20HTML-brightgreen)]()
[![No Server](https://img.shields.io/badge/Server-None-important)]()
[![Live Demo](https://img.shields.io/badge/Demo-familycode.netlify.app-7c6af7)](https://familycode.netlify.app)

</div>

---

## 🌐 Live Demo

**Try it now →** [familycode.netlify.app](https://familycode.netlify.app)

No installation needed. Works on any device with a modern browser.

---

## The problem

Vishing scams and AI-generated voice deepfakes are on the rise. A fraudster calls pretending to be your son, your mother, your bank — and the voice sounds **real**. Existing solutions are B2B-only, complex, or require accounts and subscriptions.

**FamilyCode is different:** it's a single HTML file your whole family can use in 30 seconds.

---

## How it works

Each family shares a **secret 128-bit key embedded in a URL**. Everyone who opens the link computes the same **6-digit OTP**, rotating every **5 minutes** — similar to Google Authenticator, but with zero infrastructure.

```
OTP = hash(secret_128bit + PIN + time_interval)
```

When you get a suspicious call from "your son":

1. Open FamilyCode
2. See the current 6-digit code
3. Ask the caller: *"tell me the code"*
4. If they can't → **it's a scammer**

No database. No server. No account. **The URL is the key.**

---

## Features

| | Feature | Description |
|---|---|---|
| 🔑 | **128-bit secret** | Generated via `crypto.getRandomValues()`, cryptographically secure |
| 🔗 | **Secret in URL `#fragment`** | Never sent to any server, ever |
| 🔐 | **4-digit PIN** | Modifies OTP calculation; share it separately from the link |
| ⏱ | **5-minute rotating OTP** | Same code on all devices sharing the same link + PIN |
| 👨‍👩‍👧 | **Multiple families** | Manage as many groups as you need |
| 🗑 | **Delete families** | With safety warning and restore instructions |
| 🔄 | **Restore via link** | Re-open the shared link or scan the QR to restore a deleted family |
| 🖨 | **Print QR + PIN** | Generate a printable sheet for elderly family members |
| ✏️ | **Rename families** | Update the display name anytime |
| 🌍 | **5 languages** | Italian, English, Spanish, German, French (auto-detected) |
| 🌙 | **Dark / Light theme** | Switchable |
| 📖 | **Built-in guide** | Illustrated step-by-step tutorial inside the app |
| 📦 | **Single HTML file** | No npm, no build step, no dependencies |
| 📱 | **PWA installable** | Add to home screen on Android and iOS, works offline |
| 💾 | **localStorage cache** | The URL remains the true source of truth |

---

## Security model

| Threat | Without PIN | With PIN |
|---|---|---|
| Attacker intercepts the link (e.g. WhatsApp theft) | ❌ vulnerable | ✅ protected |
| Attacker has physical access to unlocked device | ❌ vulnerable | ✅ protected (PIN not saved) |
| Malware on device | ❌ | ❌ |
| Social engineering | ❌ | ❌ |

### Why the PIN is unbreakable without a server

The app has no concept of "correct" or "wrong" PIN. It simply computes:

```javascript
OTP = hash(secret + "::" + PIN + time)
```

An attacker who brute-forces the PIN gets a different OTP for each attempt — but has **no way to know which one is correct** without calling you and asking. Each attempt requires a real phone call. It's computationally and practically impossible.

The PIN is **never saved** — not in localStorage, not in the URL, not anywhere. It lives only in your memory and is required on every session.

### The `#fragment` privacy guarantee

The URL fragment (everything after `#`) is a browser-only construct. It is **never transmitted to the server** in HTTP requests. This means even if FamilyCode is hosted on a public server, the operator cannot see your secret key in server logs.

---

## Quick start

### Try it live (recommended)

Open **[familycode.netlify.app](https://familycode.netlify.app)** — ready to use, nothing to install.

### Host it for free (recommended)

Drag and drop `familycode.html` on **[netlify.com/drop](https://netlify.com/drop)**.
You get a public URL in under 30 seconds, for free.

### Self-host

Any static file server works:

```bash
# Python
python3 -m http.server 8080

# Node
npx serve .
```

---

## Sharing flow

```
Creator
 │
 ├─ Creates family → app generates 128-bit secret
 │
 ├─ Sets a 4-digit PIN → shared verbally or via separate channel
 │
 └─ Shares the link → familycode.app/#s=3f9a...&n=Famiglia+Rossi
     │                  │
     │                  └─ family name (cosmetic)
     └─ 128-bit secret (never hits server)

Recipient
 │
 ├─ Opens link (or scans QR) → family restored automatically
 │
 └─ Enters 4-digit PIN → sees the same OTP
```

---

## Delete & restore flow

When you delete a family from the home screen:

1. A **warning modal** explains the consequences
2. The family is removed from localStorage
3. **The link still works** — opening it again (or scanning the printed QR) restores the family
4. The PIN is still required and was never stored

> 💡 *Share the link via WhatsApp or print the QR before deleting. The link is the only way to restore access.*

---

## Technical notes

### OTP algorithm

FamilyCode uses a **FNV-1a hash with avalanche mixing** and a **5-minute window**:

```javascript
function otp(secret, pin) {
  const base = secret + '::' + pin;
  const t = Math.floor(Date.now() / (300 * 1000)); // 5-min window
  const s = base + ':' + t;
  let h = 0x811c9dc5; // FNV-1a offset basis
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i);
    h = Math.imul(h, 0x01000193); // FNV prime
  }
  // Avalanche mixing
  h = Math.imul(h ^ (h >>> 16), 0x45d9f3b);
  h = Math.imul(h ^ (h >>> 13), 0x45d9f3b);
  h = (h ^ (h >>> 16)) >>> 0;
  return String(h % 1000000).padStart(6, '0');
}
```

The FNV-1a hash ensures strong avalanche properties: a single-bit change in the input (e.g. time advancing by one interval) produces a completely different 6-digit code. All devices sharing the same secret and PIN compute identical results because the only inputs are the shared secret and the current time interval — both identical across devices.

### Secret generation

```javascript
const a = new Uint8Array(16); // 128 bits
crypto.getRandomValues(a);    // CSPRNG — same entropy source as TLS
```

With 2¹²⁸ possible secrets, brute force is computationally infeasible.

### Storage

| Data | Where |
|---|---|
| Secret key | URL `#fragment` (source of truth) |
| Family name | URL `&n=` parameter |
| PIN (4 digits) | **Nowhere** — session memory only |
| Family list | `localStorage` (convenience cache) |

---

## FAQ

**What if I lose the link?**
The link is the only source of truth. Save it as a bookmark, in a password manager, or print the QR. If lost, create a new family and reshare.

**What if I forget the PIN?**
It cannot be recovered — it was never saved. Create a new family with a new PIN and reshare.

**What if I delete a family by mistake?**
If you still have the link (or the printed QR), just open it again. The family will be restored. You'll need to re-enter the PIN.

**Can I rename the family?**
Yes. The name in the URL updates when you rename and copy the new link. The old link still works with the old name.

**Does FamilyCode work offline?**
Yes, once loaded the app works fully offline. OTP computation requires no network.

**Where is FamilyCode hosted?**
The official instance is live at [familycode.netlify.app](https://familycode.netlify.app). You can also self-host it — it's a single HTML file.

**Is this TOTP-compliant (RFC 6238)?**
No. FamilyCode uses a custom hash for simplicity and to avoid requiring HMAC-SHA1 in vanilla JS without dependencies. It is not compatible with standard authenticator apps by design.

---

## Roadmap

- [x] PWA / installable on home screen
- [ ] Optional HMAC-SHA1 TOTP for RFC 6238 compliance
- [ ] Notification: "someone just checked the code" (requires minimal backend)
- [ ] Share via native share sheet (Web Share API)

---

## License

This project is licensed under the **[PolyForm Noncommercial License 1.0.0](LICENSE)**.

| | |
|---|---|
| ✅ | Personal use |
| ✅ | Study, research, experimentation |
| ✅ | Non-profit / educational use |
| ❌ | **Commercial use — not permitted** |

See [LICENSE](LICENSE) for the full legal text.

---

## Contributing

Issues and PRs welcome. Keep it simple — the single-file constraint is intentional.

By contributing, you agree that your contributions will be licensed under the same PolyForm Noncommercial License 1.0.0.

---

<div align="center">
<sub>Built to protect families from AI-powered phone fraud.<br/>No ads. No tracking. No server.</sub>
</div>
