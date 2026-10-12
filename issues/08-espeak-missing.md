# Voicemail TTS reports "eSpeak not found" at runtime

## Symptom

The headless server starts and serves on <https://127.0.0.1:8000>, but every
time an identity context starts, it logs:

```
INFO:meshchatx.rns:[...] [Error]    Voicemail: eSpeak not found
```

## Impact

This does not affect messaging or the web UI. Only voicemail text-to-speech
(spoken greetings) is unavailable.

## Fix

Install eSpeak NG from the system package manager:

```bash
sudo apt install espeak-ng
```

Then restart MeshChatX. The session did not do this; it is listed here for
completeness.

## Other runtime notes from the first run

- On first start, the app generated a new Reticulum identity in
  `storage/identity`, a self-signed TLS certificate in `storage/ssl/cert.pem`,
  and ran database migrations from schema 0 to 59.
- Landlock (ABI 8) and seccomp-BPF turned themselves on automatically.
- "Hot reloading Reticulum stack" appeared twice in the first few minutes.
  The likely cause is settings changes made in the UI, but the log does not
  show what triggered them.
