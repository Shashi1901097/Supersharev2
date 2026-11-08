# Super Share – security design and test report

Everything below was run against the real sender/receiver code. Attacks were *simulated* on one computer
(loopback network). That is a strong check of the transfer engine, but it is not a replacement for testing on real
phones or for a professional penetration test.

## How transfers are protected
- **Random pairing code**: 10 characters from a 32-letter alphabet (50 bits), new for every Receive session.
- **Private Wi-Fi Direct group**: its name and password are derived from the code (PBKDF2-HMAC-SHA256, 120,000 rounds),
  so only a phone that knows the code can join; the radio link is WPA2-encrypted.
- **Every connection is authenticated** with HMAC-SHA256 over a fresh random challenge (the code is never sent,
  answers cannot be replayed or reused for another request type).
- **Lock-out** after 8 wrong attempts; unauthenticated connections are dropped after 5 s; at most 32 connections.
- **Untrusted input is validated**: file count, total size claimed, ids, chunk numbers; names are reduced to a plain
  file name (no folders, no `..`, no control characters). Sender also refuses oversized replies.
- **Storage safety**: files that do not fit into the free space (minus a 100 MB reserve) are refused and reported;
  space is never over-committed.
- **Privacy**: no accounts, analytics, ads or servers. Backups disabled. Files are saved only in Downloads/SuperShare.
  Only the permissions needed for Wi-Fi Direct are requested.

## Results: 20 automated attack/behaviour tests (all pass; the whole suite ran 8 more times in a row without a failure)
| Test | What it proves |
|---|---|
| names / nameFuzz | 40,000 random hostile names (separators, `..`, control chars, RTL tricks, huge) always become safe plain names |
| keys / keyQuality | 5,000 codes unique and uniformly random; one changed character flips ~half the key bits; derivation is slow on purpose |
| transfer | 5 files (0 B to 45 MB, parallel streams) arrive byte-identical (SHA-256) |
| resume | receiver killed mid-transfer; second run continues from saved progress; file identical |
| wrongKey / lockout | wrong code stores nothing; after too many attempts even the right code is refused |
| replayAndTypeConfusion | a recorded valid answer cannot be replayed; a QUERY answer cannot be used for DATA |
| idleAndConnectionFlood | idle peers dropped after the 5 s limit; flooding with 42 connections is capped; receiver recovers and still works |
| memoryExhaustionGuards | announcing 2,000 files of 4 TB (to exhaust memory) or too many files is rejected instantly |
| fuzzReceiver | 400 rounds of random and mutated messages: no crash, no stray files, receiver still correct afterwards |
| insufficientStorage / manyFilesNeverOverCommitSpace | exact shortfall reported; 200 files into space for 80 stores exactly 80 |
| alreadyHave | files already received are not sent again; a deleted file is sent again |
| pathTraversalName / malformedInput | hostile names and headers cannot write outside the folder or break the receiver |
| concurrentSenders | 4 phones sending at once: every file intact |
| maliciousReceiver | a fake receiver answering with a gigantic bitmap is refused without allocating memory |
| cancel | Stop ends the transfer promptly and publishes no half file |

## Static scan: 27/27 checks pass (`python3 tools/static_security_scan.py`)
Manifest (one exported activity, no backup, no cleartext, minimum permissions, no restricted permissions), build
(API 36 target, minified release, no signing secrets), and source (no shell execution, WebView, TLS overrides, logging,
reflection, plain URLs, weak hashes/randomness, hard-coded secrets; constant-time MAC check; SecureRandom; PBKDF2).

## Brute-force estimate for the pairing code
One guess costs about 100 ms on a single CPU core (measured). Cracking a 50-bit code needs up to 1.1 x 10^15 guesses.
Even an attacker able to try **1,000,000 guesses per second** (far beyond normal) would need about **36 years**, and the code
stops working when the Receive session ends.

## Fixes made because of these tests
- Unauthenticated connections now time out after 5 s (was 20 s).
- An announcement can claim at most ~8 TB in total (memory-exhaustion guard).
- Read buffers reduced to 1 MB per connection (low memory on small phones).
- Progress files are written atomically; stopping the receiver waits for progress to be saved.
- Stop/Cancel reacts immediately.

## What has NOT been tested – please do before publishing
- The Android layer (Wi-Fi Direct joining, MediaStore saving, Open/Delete, screens) could not be run on a phone here.
  **Test on at least two real phones** (different brands if possible).
- No third-party penetration test or code audit.
- Data integrity relies on TCP and WPA2 checksums; there is no extra per-file hash (keeps speed high).
- Files that live only in the cloud (for example Google Drive files not downloaded) cannot be sent; download them first.
- Speed depends on both phones' Wi-Fi hardware and storage; 1 Gbps is not guaranteed.
