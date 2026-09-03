# References: WhatsApp end-to-end encrypted backups (HSM Backup Key Vault)

Keepers for the 2026-09-03 teardown. Primary sources first.

## Primary (WhatsApp / Meta)
- WhatsApp, "Security of End-To-End Encrypted Backups" (official whitepaper): https://www.whatsapp.com/security/WhatsApp_Security_Encrypted_Backups_Whitepaper.pdf
  - The canonical description of the Backup Key Vault, OPAQUE, the 64-digit-key-vs-password choice, the guess counter, and the audit / transparency steps.
- Engineering at Meta, "How WhatsApp is enabling end-to-end encrypted backups" (2021-09-10): https://engineering.fb.com/2021/09/10/security/whatsapp-e2ee-backups/
- Engineering at Meta, "How Meta Is Strengthening End-to-End Encrypted Backups" (2026-05-01): https://engineering.fb.com/2026/05/01/security/meta-strengthening-end-to-end-encrypted-backups/
  - The 2026 hardening: Cloudflare-signed + Meta-counter-signed validation bundles, over-the-air fleet keys for Messenger, the open-source "mbt" (Meta Binary Transparency) tool.

## Independent analysis
- "Security Analysis of the WhatsApp End-to-End Encrypted Backup Protocol," CRYPTO 2023 (IACR eprint 2023/843): https://eprint.iacr.org/2023/843.pdf
  - Formal UC-framework analysis of the WhatsApp Backup Protocol (WBP). Confirms 2HashDH OPRF, 3DH, OPAQUE_K derivation.
- NCC Group, "End-to-End Encrypted Backups Security Assessment: WhatsApp" (2021-10-27): https://www.nccgroup.com/media/fzwdxklh/_ncc_group_whatsapp_e001000m_report_2021-10-27_v12.pdf

## Protocol background
- IETF CFRG OPAQUE draft (asymmetric PAKE): https://datatracker.ietf.org/doc/draft-irtf-cfrg-opaque/

## Press / secondary (for framing and dates)
- ESET WeLiveSecurity (2021-09-14): https://www.welivesecurity.com/2021/09/14/whatsapp-announces-end-to-end-encrypted-backups/
- Help Net Security (2026-05-05), on the proof-based 2026 update: https://www.helpnetsecurity.com/2026/05/05/meta-whatsapp-messenger-encrypted-backups-update/

## Confirmed facts used
- Backup encrypted with a random 256-bit key on-device before upload.
- Two recovery options: user-held 64-digit key, or password backed by the HSM Vault.
- Password path uses OPAQUE (2HashDH OPRF + 3DH); password never sent to the Vault.
- HSM guess counter initialized at 10; key rendered permanently inaccessible after the limit.
- Vault is a geographically distributed fleet across 5 datacenter sites with majority-consensus replication.
- Encrypted backup blob stored on Google Drive (Android) / iCloud (iOS); the key is not there.

## Inference (labeled in the report, not public)
- Exact consensus algorithm (Paxos vs Raft vs custom), precise sharding scheme, HSM vendor/model.
