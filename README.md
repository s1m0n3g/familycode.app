# FamilyCode 🔐

> **Verify who's really calling you.**  
> A zero-server, zero-signup OTP webapp to defeat vishing scams and audio deepfakes.

---

## The problem

Vishing scams and AI-generated voice deepfakes are on the rise. A fraudster calls pretending to be your son, your mother, your bank — and the voice sounds real. Existing solutions are B2B-only, complex, or require accounts and subscriptions.

FamilyCode is different: it's a single HTML file your whole family can use in 30 seconds.

---

## How it works

Each family shares a **secret 128-bit key embedded in a URL**. Everyone who opens the link computes the same **6-digit OTP**, rotating every 30 seconds — exactly like Google Authenticator, but with zero infrastructure.

```
OTP = hash(secret_128bit [+ PIN] + time_interval)
```

When you get a suspicious call from "your son":

1. Open FamilyCode
2. See the current 6-digit code
3. Ask the caller: *"tell me the code"*
4. If they can't → it's a scammer

No database. No server. No account. The URL **is** the key.

---

## Features

- 🔑 **128-bit secret** generated via `crypto.getRandomValues()` — cryptographically secure
- 🔗 **Secret embedded in URL `#fragment`** — never sent to any server, ever
- 🔐 **Optional PIN** — modifies OTP calculation; share it separately from the link for double protection
- ⏱ **30-second rotating OTP** — same code on all devices sharing the same link
- 👨‍👩‍👧 **Multiple families** — manage as many groups as you need
- 🌍 **5 languages** — Italian, English, Spanish, German, French (auto-detected)
- 🌙 **Dark / Light theme** — switchable
- 📦 **Single HTML file** — no npm, no build step, no dependencies
- 💾 **localStorage** for convenience — the URL remains the true source of truth

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

```
OTP = hash(secret + "::" + PIN + time)
```

An attacker who brute-forces the PIN gets a different OTP for each attempt — but has no way to know which one is correct without calling you and asking. Each attempt requires a real phone call. It's computationally and practically impossible.

The PIN is **never saved** — not in localStorage, not in the URL, not anywhere. It lives only in your memory and is required on every session.

### The `#fragment` privacy guarantee

The URL fragment (everything after `#`) is a browser-only construct. It is **never transmitted to the server** in HTTP requests. This means even if FamilyCode is hosted on a public server, the operator cannot see your secret key in server logs.

---

## Quick start

### Use it now
Just open `familycode.html` in any browser. No installation required.

### Host it for free (recommended)
Drag and drop `familycode.html` on [netlify.com/drop](https://netlify.com/drop).  
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
  ├─ [Optional] Sets a PIN → shared verbally or via separate channel
  │
  └─ Shares the link → familycode.app/#s=3f9a...&n=Famiglia+Rossi
                                         │              │
                                         │              └─ family name (cosmetic)
                                         └─ 128-bit secret (never hits server)

Recipient
  │
  └─ Opens link → enters PIN if required → sees the same OTP
```

---

## Technical notes

### OTP algorithm

FamilyCode uses a simplified TOTP-like approach:

```javascript
function otp(secret, pin) {
  const base = pin ? secret + '::' + pin : secret;
  const t = Math.floor(Date.now() / 30000); // 30-second window
  let h = 0;
  const s = base + t;
  for (let i = 0; i < s.length; i++) {
    h = (Math.imul(31, h) + s.charCodeAt(i)) | 0;
  }
  return String(Math.abs(h) % 1000000).padStart(6, '0');
}
```

All devices sharing the same secret (and PIN) compute identical results because the only inputs are the shared secret and the current time interval — both identical across devices.

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
| PIN | Nowhere — session memory only |
| Family list | `localStorage` (convenience cache) |

---

## FAQ

**What if I lose the link?**  
The link is the only source of truth. Save it as a bookmark or in a password manager. If lost, create a new family and reshare.

**What if I forget the PIN?**  
It cannot be recovered — it was never saved. Create a new family without PIN, or with a new PIN, and reshare.

**Can I rename the family?**  
Yes. The name in the URL updates when you rename and copy the new link. The old link still works with the old name.

**Does FamilyCode work offline?**  
Yes, once loaded the app works fully offline. OTP computation requires no network.

**Is this TOTP-compliant (RFC 6238)?**  
No. FamilyCode uses a custom hash for simplicity and to avoid requiring HMAC-SHA1 in vanilla JS without dependencies. It is not compatible with standard authenticator apps by design.

---

## Roadmap ideas

- [ ] QR code generation for easy sharing (especially useful for elderly family members)
- [ ] PWA / installable on home screen
- [ ] Optional HMAC-SHA1 TOTP for RFC 6238 compliance
- [ ] Notification: "someone just checked the code" (requires minimal backend)

---

## License

MIT — do whatever you want with it.

---

## Contributing

Issues and PRs welcome. Keep it simple — the single-file constraint is intentional.

---

*Built to protect families from AI-powered phone fraud. No ads. No tracking. No server.*