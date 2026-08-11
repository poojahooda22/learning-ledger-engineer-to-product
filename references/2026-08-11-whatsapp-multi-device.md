# References: WhatsApp Multi-Device

Saved for the 2026-08-11 teardown (companion devices, client-fanout, App State Sync, LTHash).

## Primary sources

- Engineering at Meta, "How WhatsApp enables multi-device capability" (2021-07-14).
  The client-fanout model, per-device Signal identity, Automatic Device Verification.
  https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/

- WhatsApp Security Whitepaper (Encryption Overview). Account Signature / Device
  Signature, app state sync, sender keys, prekeys.
  https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf

## Protocol reimplementations (open source, document the wire format)

- whatsmeow appstate package. App State Sync collections (critical_block,
  critical_unblock_low, regular_low, regular_high, regular), patch encode/decode.
  https://pkg.go.dev/go.mau.fi/whatsmeow/appstate

- LTHash package. SubtractThenAdd, the "WhatsApp Patch Integrity" HKDF label,
  1024-bit (128-byte) LtHash16 values, homomorphic summation hash.
  https://pkg.go.dev/github.com/testovoleg/whatsapp/appstate/lthash

- whatsmeow issue #651, "mismatching LTHash". Real-world proof the integrity
  check fires and forces a full app-state resync when a patch chain diverges.
  https://github.com/tulir/whatsmeow/issues/651

## Background

- InfoQ, "WhatsApp Adopts the Signal Protocol for Secure Multi-Device
  Communication" (2021). Summary of the client-fanout + per-device identity design.
  https://www.infoq.com/news/2021/07/WhatsApp-signal-protocol/

- Signal Sender Keys (group messaging scheme reused per-device in groups).
  https://en.wikipedia.org/wiki/Sender_Keys

## Key facts captured

- Up to 4 companion devices + phone = 5 surfaces. Phone can be offline; 14-day
  idle logout of companions (also a stolen-device kill switch and GC valve).
- 1:1 = client-fanout, one ciphertext per device over a distinct pairwise Double
  Ratchet session. Your own other devices are in the recipient set (sent-message
  mirroring).
- Groups = per-device Sender Keys: distribute the key pairwise once (O devices,
  rare), then each message encrypted once and server-fanned (O(1) hot path).
- Automatic Device Verification: Account Signature (phone signs companion's
  identity key) + Device Signature (companion signs phone's). Others reject any
  device lacking a phone-signed Account Signature the server cannot forge.
- App State Sync: encrypted op-log of Mutations bundled into Patches, per-
  Collection; server stores Snapshots + patches it cannot read.
- LtHash16: homomorphic hash, 1024-bit sum of per-item vectors, updated by
  SubtractThenAdd, verify cost O(changed) not O(whole collection). Plus per-
  mutation value MACs and a patch HMAC so the server cannot forge/flip/drop.
- Scale: 2.5B users, 100B+ messages/day. Client-fanout multiplies delivery by
  device count; per-device store-and-forward queues shard horizontally; prekey
  pools drain faster (devices x users) and can exhaust into a reduced-security
  X3DH fallback.

## Builds on earlier ledger teardowns

- 2026-06-15 WhatsApp delivery receipts (store-and-forward post office).
- 2026-07-01 WhatsApp end-to-end encryption (X3DH, Double Ratchet, Sender Keys).
