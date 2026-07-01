# References: WhatsApp end-to-end encryption (Signal Protocol)

Saved 2026-07-01 for the WhatsApp E2E encryption teardown.

## Primary specifications and white papers
- WhatsApp Encryption Overview, Technical White Paper (v9, Feb 2026): https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf
- Signal, X3DH Key Agreement Protocol (Marlinspike, Perrin): https://signal.org/docs/specifications/x3dh/
- Signal, The Double Ratchet Algorithm (Perrin, Marlinspike): https://signal.org/docs/specifications/doubleratchet/

## Reference encyclopedias
- Double Ratchet Algorithm (Wikipedia): https://en.wikipedia.org/wiki/Double_Ratchet_Algorithm
- Sender Keys (Wikipedia): https://en.wikipedia.org/wiki/Sender_Keys
- Signal Protocol (Wikipedia): https://en.wikipedia.org/wiki/Signal_Protocol
- Open Whisper Systems (Wikipedia): https://en.wikipedia.org/wiki/Open_Whisper_Systems

## Academic analysis (peer-reviewed / preprint)
- "WhatsUpp with Sender Keys? Analysis, Improvements and Security Proofs" (IACR ePrint 2023/1385): https://eprint.iacr.org/2023/1385.pdf
- "Analysis and Improvements of the Sender Keys Protocol for Group Messaging" (arXiv 2301.07045): https://arxiv.org/pdf/2301.07045
- "Prekey Pogo: Security and Privacy Issues in WhatsApp's Handshake Mechanism" (arXiv 2504.07323): https://arxiv.org/pdf/2504.07323
- "A Dive into WhatsApp's End-to-End Encryption" (arXiv 2209.11198): https://arxiv.org/pdf/2209.11198

## History and scale
- Signal blog: OWS partners with WhatsApp: https://signal.org/blog/whatsapp/
- TechCrunch: encryption rollout complete, 5 Apr 2016: https://techcrunch.com/2016/04/05/whatsapp-completes-end-to-end-encryption-rollout/
- TechCrunch: ~100 billion messages/day, 29 Oct 2020: https://techcrunch.com/2020/10/29/whatsapp-is-now-delivering-roughly-100-billion-messages-a-day/

## Key facts pinned
- Primitives: Curve25519 (key agreement/signatures), AES-256-CBC (message body), HMAC-SHA256 (auth).
- Key types: Identity Key (long-term), Signed Pre Key (medium-term), One-Time Pre Keys (batch of ~100, single use), Sender Key (Chain Key + Signature Key pair) for groups.
- X3DH: SK = KDF(DH1||DH2||DH3||DH4); one-time prekey DH is the strongest forward-secrecy ingredient; server can be contacted while recipient is offline (asynchronous).
- Double Ratchet: three KDF chains (root, sending, receiving); symmetric ratchet = one-way KDF chain (forward secrecy); DH ratchet re-injects fresh DH per round (self-heal / post-compromise recovery); header carries ratchet pubkey + N + PN; skipped keys stored in a dict; MAX_SKIP caps DoS-by-arithmetic.
- Sender Keys: distribute once pairwise via SenderKeyDistributionMessage (O(N), rare); each group message encrypted once, signed, server fans out (O(1) sender + fan-out); resample on member leave.
- Multi-device: each linked device is its own cryptographic identity in the fan-out.
- Rollout finished 5 April 2016; protocol by Trevor Perrin and Moxie Marlinspike (Open Whisper Systems).
