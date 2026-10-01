# Dreamt — security & privacy write-up

A threat model and self-audit of **Dreamt**, an iOS dream journal I built. This repository is the security documentation, not the source.

**→ [Read the threat model](THREAT-MODEL.md)**

---

## Why this exists

A dream journal holds material the GDPR treats as special-category data under Article 9 — mental health, sexuality, religious belief, trauma, named third parties who never consented to appearing. Dreamt makes an absolute privacy promise to its users on its settings screen. I wanted to find out whether that promise was true.

It wasn't, in one place.

## The finding

Voice capture used `SFSpeechRecognizer` without setting `requiresOnDeviceRecognition`. That property defaults to `false`, so iOS was free to stream the captured audio to Apple's servers for transcription — meaning dictated dreams could leave the device, in the user's own voice, before anything was ever encrypted.

The database encryption was sound. It was protecting a copy of data that had already crossed the boundary.

Fixed by requiring on-device recognition and failing closed when the device or locale can't do it. The rejected alternative — falling back to server transcription with a warning — is covered in §5, along with why.

## What's in the document

- **§1** What's being protected, and the adversary it's built against
- **§2** Field-by-field data classification, including what's deliberately left in plaintext and what that leaks
- **§3** AES-256-GCM via CryptoKit, and the Keychain attributes doing the real work
- **§4** The trust boundary
- **§5** The finding — cause, impact, fix, and the mitigation I turned down
- **§6** Seven residual risks, accepted rather than solved

## Caveats, stated up front

- This is a self-audit. No independent review.
- The GDPR reasoning is an engineer's reading, not a lawyer's.
- The residual risks in §6 are open, not fixed. They're listed because a threat model that only covers solved problems isn't a threat model.



