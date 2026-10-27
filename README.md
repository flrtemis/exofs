# EXOFS — start here

Four files. Read in this order.

| File | What it is |
|---|---|
| **`EXOFS-playbook.md`** | The doctrine. Four fabrics, five patterns, the MOLT 60-minute protocol, hybrid-fabric field guide, failure analysis, 30-day roadmap. **Read this first.** |
| **`architecture.svg`** | One-page diagram of the Ghost Machine — client, four fabric layers, hot/warm/cold storage, failure paths, the MOLT timeline. |
| **`ghostdrive.html`** | A working disk made of URLs, in three steps: **write** (drop a file → 4 MiB content-addressed bricks → AES-GCM in the browser → manifest), **custody** (pushes bricks + manifest into a real GitHub repo via the Git Data API — blobs → tree → commit), **restore** (pulls them back over HTTPS, verifies every hash, decrypts, reassembles). Verified end-to-end: byte-identical round trip, deterministic ciphertext so git dedupes, wrong passphrase fails cleanly, a 10 MB file indexed by 432 bytes of manifest. |
| **`mirror.yml`** | Drop into `.github/workflows/`. Verifies every brick's hash, mirrors to R2 + Hugging Face, runs a restore drill from the mirror, and commits an integrity report — every 6 hours, free, on public-repo runners. |

## The 5-minute version

1. **Your computer was never the disk** — it was the thing that talked to the disk. Keep the protocol, move the hashes.
2. **Nothing survives locally, so every tool must be a URL.** The post-T−0 toolchain is a bookmarks file.
3. **A save is a round-trip.** If nothing acknowledged it, it did not happen.
4. **Don't copy the disk, copy the irreplaceable.** Text before media: keys → identities → documents → code. That's usually < 5 GB and ~90% of what matters.
5. **Never store a file, store bricks.** Chunk to 4 MiB, sha256 each, encrypt, upload; the manifest — a few KB of JSON — is the whole index and it lives in git.
6. **Three custodians, different companies, different failure modes.** GitHub + Cloudflare R2 + Hugging Face/Internet Archive. Never three repos under one login.
7. **A backup you have not restored is a hypothesis.** Schedule the drill, read the report.

## The honest limit

You cannot move all local data to free tiers — the volumes don't exist and it would be ToS abuse that gets accounts terminated exactly when they're needed. The defensible claim is the one in the playbook: **all irreplaceable files, provably restorable, plus regeneration plans for the rest.** That survives contact with reality; "everything" doesn't.

---

*Free-tier numbers throughout were verified against provider documentation in October 2026 and will drift. Treat them as shape, not gospel.*
