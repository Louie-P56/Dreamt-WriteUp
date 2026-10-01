# Encrypting a dream journal

**Threat model & privacy architecture — Dreamt (iOS)**

Dreamt is an iOS dream journal. Its contents are close to the most sensitive category of personal data there is — and the app makes an absolute privacy claim to its users. This document sets out what that claim is worth, where it held, and the one place it turned out to be false.

| | |
|---|---|
| **System** | Dreamt — iOS 16+ |
| **Scope** | Data at rest & in transit |
| **Storage** | On-device only |
| **Backend** | None |
| **Review** | Self-audit |

---

## §1 · What is being protected

A dream journal is not a notes app. Free-text dream descriptions routinely contain material that the GDPR treats as **special-category data under Article 9**: mental health, sexuality, religious belief, trauma, named third parties who never consented to being written about. The user is recording it half-awake, with no editorial filter, in the most unguarded thirty seconds of their day.

So the design assumption is that *every* entry is special-category until proven otherwise, and that anything *derived* from an entry inherits its sensitivity. Tags such as `emotion:anxious` or `theme:falling` are not metadata — they are a structured summary of the same protected content, and in some ways a more legible one. They are encrypted alongside the text, not beside it.

The adversary this is built against is ordinary and realistic: someone who obtains the device or its filesystem — a partner, a thief, a repair shop, a backup extracted to a laptop.

---

## §2 · Data classification

| Data | Classification | Reasoning |
|---|---|---|
| Dream transcript | Encrypted | Assumed Art. 9 throughout. |
| Derived tags | Encrypted | A structured summary of the same content. |
| Interpretation | Encrypted | Derived from, and quotes, the transcript. |
| Timestamp | Plaintext | Needed to sort and to compute streaks without decrypting every row. |
| Lucid flag | Plaintext | Same — drives counts and streaks on the home screen. |
| Capture duration | Plaintext | Used for capture-friction measurement. |
| Consent record | Plaintext | Two timestamps. A compliance artifact, not a secret. |
| Data export | Leaves device | Plaintext JSON, by explicit user action — the GDPR portability right. |
| Voice audio | Leaves device | It did, silently, until §5. Now it does not. |

The four plaintext rows are a deliberate trade, and they are a real leak. An attacker who reaches the database without the key still learns when you sleep, how often you remember dreaming, which nights were lucid and how long you spoke — a behavioural profile assembled without recovering a single word. That was judged an acceptable price for rendering a list without decrypting the whole journal on every scroll. It is a judgement, not a non-issue.

---

## §3 · Cryptography and key handling

The sensitive fields are serialised into a single JSON payload and sealed with **AES-256-GCM** via CryptoKit. GCM is authenticated, so a modified row fails to open rather than decrypting into something plausible. The stored blob is the `.combined` form — nonce, ciphertext and tag together — so nonce management is not hand-rolled.

The 256-bit key is generated on first use and held in the Keychain as a generic password item, with two attributes doing the real work:

- `kSecAttrSynchronizable = false` — the key is never promoted to iCloud Keychain. It cannot follow the account onto another device.
- `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` — unreadable while the device is locked, and excluded from encrypted backups, so restoring a backup elsewhere yields ciphertext and no key.

The storage layer has no networking code of any kind. That is a structural property rather than a policy: the type has no way to reach a network, so no future change to it can quietly start exfiltrating rows.

---

## §4 · Trust boundary

Everything inside the boundary stays on the handset:

```
┌─ DEVICE ─────────────────────────────────────────────┐
│                                                      │
│  Capture (voice / text)    Tagging (NaturalLanguage) │
│  Interpretation            Pattern detection         │
│  (on-device templates)     (dream signs)             │
│                                                      │
│  SQLite                    Key                       │
│  (AES-256-GCM)             (Keychain, this device)   │
│                                                      │
│  Sleep data (HealthKit, read-only)                   │
│                                                      │
│  ──▶ Apple speech servers   ← the only unintended    │
│      [REMOVED — see §5]       crossing               │
└──────────────────────────────────────────────────────┘
```

Interpretation runs on-device from templates rather than calling a hosted LLM. That costs real quality — the reflections are flatter than a frontier model would produce — and it was chosen anyway, because sending dream text to a third-party inference endpoint would move the most sensitive asset in the system across the boundary this entire design exists to hold.

---

## §5 · Finding: the guarantee was false

The app told users this, verbatim, on its settings screen:

> "Dreams are encrypted and stored only on this device. Tagging and reflection run on-device. Nothing is shared with anyone unless you export it yourself."

### F-01 — Dictated dreams were transcribed off-device

Voice capture used `SFSpeechRecognizer` without setting `requiresOnDeviceRecognition`. That property defaults to `false`, which leaves iOS free to stream the captured audio to Apple for transcription. Every dictated dream — raw audio, in the user's own voice, describing exactly the material §1 classifies as Article 9 — could leave the device before a single byte was ever encrypted.

Encryption at rest is irrelevant against this. The plaintext had already crossed the boundary at the moment of capture; the database was busy protecting a copy of something that had already left.

| | |
|---|---|
| **Impact** | The app's central privacy claim was false for every voice-logged entry. |
| **Cause** | An unsafe default, silently inherited. Nothing in the code said "send this to Apple." |
| **Fix** | Require on-device recognition, and refuse to record at all when the device or locale cannot do it — the app now fails closed, tells the user plainly, and points them at typing. |
| **Rejected** | Falling back to server transcription with a warning. A privacy guarantee that degrades quietly under conditions the user cannot evaluate is not a guarantee; it is marketing. |

### Why this is the interesting part

Nothing here was exotic. The encryption was sound, the key handling was deliberate, the architecture had no server to breach — and the guarantee still broke, because a framework default disagreed with the product's promise and never said so. The failure was not in the cryptography. It was in the gap between what the interface claimed and what the platform actually did, which is where most privacy failures in shipped software actually live.

---

## §6 · Residual risk

Known and accepted, rather than solved:

**R-01 — Metadata discloses behaviour.** Timestamps, lucid flags and capture durations sit in the clear (§2). Sleep timing and dream frequency are recoverable without the key.

**R-02 — No biometric gate on the key.** Device unlock is the only barrier. Adding `kSecAccessControl` with `.biometryCurrentSet` would require Face ID per session; it was not done, so an unlocked compromised device yields the key.

**R-03 — No key rotation.** The key is generated once and never rotated. There is no re-key path and no way to recover from suspected key compromise short of erasing the journal.

**R-04 — Deletion is logical, not physical.** Erasure issues SQL `DELETE` without `PRAGMA secure_delete`, so ciphertext may persist in freed pages. Harmless without the key; compounding alongside R-02.

**R-05 — Export is the widest exposure.** Portability writes the entire journal as plaintext JSON to the share sheet. Intended and user-initiated, but it is the one routine path by which everything leaves at once.

**R-06 — Consent record is not tamper-evident.** Stored unencrypted in `UserDefaults` with no integrity protection. Adequate as a UX state flag; inadequate if it ever needs to serve as legal evidence of consent.

**R-07 — No legal review.** Article 9 handling here reflects an engineer's reading of the GDPR, not a lawyer's. It has not been reviewed by anyone qualified to sign it off.

Explicitly out of scope: a jailbroken or compromised OS, forensic extraction with a known passcode, and an adversary with sustained physical access to an unlocked device. None of these are defended against, and claiming otherwise would repeat the mistake in §5.

---

Dreamt is a personal project — a Swift package plus a SwiftUI app, no backend, no analytics, no third-party SDKs. This document covers its security and privacy design only.

Written after the self-audit that produced F-01.
