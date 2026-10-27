# EXOFS
### The Disk Outside the Machine — a 60-minute doctrine for a world without local storage

**Status:** design doctrine + runbook · **Clock:** T−60 minutes · **Constraint set:** free-tier, public-internet, zero new software installs (installs cannot persist), servers unaffected, disks dying globally.

> **Read this first.** This is written as an operating doctrine, not a prediction. Every number below was checked against provider documentation in **October 2026** — free tiers move constantly, so treat the tables as *shape*, not gospel, and re-verify anything you're about to bet a file on. The most useful thing here is the structure: **four fabrics, five patterns, one protocol.** If reality mutates, the structure still holds.

---

## 0. What actually breaks (and what doesn't)

In the next hour, one capability is removed from every personal computer on Earth: **persistence**. Not computation, not RAM, not networking. Writes to local disk stop being durable — anything you save dies with the process, the sleep, the crash.

That single deletion cascades hard, and it's worth being precise about why:

| Capability | Assumed to survive? | Reality after T−0 |
|---|---|---|
| Compute | yes | **Yes**, but only as *ephemeral* compute. A laptop becomes a very expensive calculator with a browser. |
| RAM / open applications | yes | **Yes, until reboot.** Every reboot is a factory reset. Battery life is now your session lifetime. |
| Network | yes | **Yes.** This is the only surviving wire to reality. |
| Servers / the cloud | yes | **Yes, completely unaffected.** This is the entire solution space. |
| Installing software | yes | **No.** `apt install` writes to disk. Any tool you "install" evaporates. **Every tool must be a URL.** |
| Local caches, build dirs, `.git` | yes | **No.** Your working copy of everything is now volatile memory. |
| "My Documents" | yes | **No.** It is already gone at T−0 unless it left the machine before then. |

Three consequences fall out immediately, and they drive everything else:

1. **Tools must be URLs.** The only software anyone can run after T−0 is software served over HTTPS and executed in a browser tab or on someone else's server. The post-T−0 toolchain is a bookmarks file — which is why the bookmarks themselves must be off-machine *now*.
2. **Every action must be a transaction.** A save is not a disk write; a save is a **network round-trip that someone else acknowledges**. If the round-trip didn't happen, the work didn't happen. Users must be taught one reflex: *"Did it go green?"*
3. **The bottleneck is not storage, it's the write path.** You have one hour of upload bandwidth for a lifetime of files. **Triage is the whole game.** The winning strategy is not "copy the disk," it's "copy the irreplaceable, then make the replaceable re-derivable."

---

## 1. The four fabrics

Stop thinking "a cloud drive." Think four independent fabrics, because a fabric that stores can't compute, and a fabric that computes can't be trusted to retain. EXOFS binds them with a contract between each layer.

```
              ┌──────────────────────────────────────────────────────┐
  L4  IDENTITY│ passkeys · OAuth · Shamir-sharded keys · 2FA        │  who you are
              ├──────────────────────────────────────────────────────┤
  L3  STATE   │ CRDTs · append-only logs · catalogs · manifests      │  what is true
              ├──────────────────────────────────────────────────────┤
  L2  COMPUTE │ browser WASM → edge workers → CI runners → GPU       │  what runs
              ├──────────────────────────────────────────────────────┤
  L1  STORAGE │ git · object stores · vaults · cold archives         │  what persists
              ├──────────────────────────────────────────────────────┤
  L0  TRANSPORT│ HTTPS · CDN · jsDelivr · Pages · WebRTC/BitTorrent  │  how bits move
              └──────────────────────────────────────────────────────┘
```

### L1 — Storage fabric (the surviving "disk")

| Fabric | Free reality (Oct 2026) | What it's genuinely good at | Trap |
|---|---|---|---|
| **GitHub repos** | Unlimited public/private repos. **100 MiB hard per-file block** (25 MB via browser). Repo recommended < 1 GB, practically degrades past ~5 GB. Full version history. | Source code, text, configs, small binaries, **and versioning every write for free**. Content-addressed by SHA. | Git is a *history*, not a drive. Big binaries bloat clone time forever. |
| **GitHub Releases** | **2 GiB per asset**, no published cap on total release size or bandwidth. Served from a CDN. | Bulk binaries, datasets, archives, release-grade artifacts. | Upload is CLI/API-driven; you'll drive it from CI, not your dying laptop. |
| **Git LFS** | 10 GiB storage + 10 GiB bandwidth/mo (Free). | Medium binaries that must live in-tree. | Bandwidth quota is small; clones bill it. |
| **Hugging Face Hub** | **Public storage: free, best-effort, effectively very large.** Private: 100 GB. Per-file up to 200 GB (50 GB recommended), < 100k files/repo. | Datasets, models, media, anything big and public. Fast CDN downloads. | Best-effort public storage = don't be abusive, and don't rely on a private vault being free forever. |
| **Cloudflare R2** | 10 GB-month, 1M Class A + 10M Class B ops/mo, **egress free forever**. | Hot object store for an app's live data. S3 API from anywhere. | 10 GB is a *working set*, not an archive. |
| **Google Drive** | 15 GB shared across Drive/Gmail/Photos. | Private vault of last resort, human-browsable, trivial sharing. | No content addressing, no cheap API at scale, no versioning guarantees. |
| **Internet Archive** | Free uploads of public material, effectively unbounded, permanent by mission. | **Cold tier of last resort.** "Permanent" is literally the institution's purpose. | Not a private drive; slow; public; requires policy-appropriate content. |
| **npm / PyPI registries** | Free publishing of packages. | Sneaky-good immutable blob store (a package version is immutable *forever* and CDN-mirrored). | Registry ToS: publishing data as packages is abuse. Don't. |
| **Telegram / Discord attachments** | ~2 GB per file (Telegram), unlimited-ish retention; Discord CDN links expire-hinged and are volatile. | Emergency short-term relay *between humans* when a proper API is down. | **Not storage.** No integrity guarantees, no SLA, ToS-hostile for archiving. Use for transport only. |

### L2 — Compute fabric (the surviving "CPU")

| Tier | Fabric | Free reality | Use |
|---|---|---|---|
| **T0** | **Browser + WASM** (Pyodide, WebContainers, FFmpeg.wasm) | Free, local, instant, private. Runs *in the tab*, needs no disk. | Editing, hashing, encryption, transcoding, diffing — the 90% of "local" work that turns out to be CPU-only. |
| **T0.5** | **Live coding shells** (perchance, StackBlitz, Replit, Observable, val.town, Deno Playground) | Free tiers; editable in-browser; some host the result. | The "IDE" that survives, because the IDE is a URL. perchance's generator/agent model is unusually good here: many tiny generators that each do one thing, plus a real-time editor, composed into one app by wiring URLs and embeds together. |
| **T1** | **Edge functions** (Cloudflare Workers, Deno Deploy, Vercel/Netlify functions) | Workers Free: 100k req/day, 10 ms CPU, 128 MB. | The API layer: auth, routing, signed URL minting, metadata, small writes. |
| **T2** | **CI runners as a cloud computer** (GitHub Actions) | **Standard runners on public repos: free and unlimited.** 4 vCPU / 16 GB / 14 GB SSD on public repos. 6 h max per job, 35-day max workflow run, 256-job matrices. | The workhorse. Every heavy job: mirroring, chunking, encryption, builds, scheduled sweeps, restore drills. **This is the machine you no longer own.** |
| **T3** | **Interactive VMs / notebooks** (Codespaces 120 core-hrs/mo, Colab, Kaggle, Gitpod, HF Spaces) | Free tiers with sleep policies. Codespaces gives you a real Linux box with a terminal in a tab. | Interactive debugging, GUI-ish work, GPU jobs, "I need a shell for ten minutes." |

### L3 — State fabric

| Fabric | Free reality | Use |
|---|---|---|
| **Cloudflare D1 / KV** | D1: 500 MB/DB, 5 GB/account, 100k row-writes/day. KV: 1 GB, 100k reads/day, **1,000 writes/day** (beware). | Catalogs, manifests, indexes, feature flags, auth sessions. |
| **Supabase / Neon / Turso / Upstash** | Small-but-real Postgres/SQLite/Redis free tiers. | Relational metadata, full-text search, queues. |
| **Google Sheets / Notion / Airtable APIs** | Free, human-editable, versioned-ish. | **A database a human can read without a client.** Genuinely the best emergency catalog format: survives the loss of every tool except a browser. |
| **Git itself** | Free, content-addressed, merkle-verified, replicated by every clone. | The canonical state log. Append-only by construction. |

### L4 — Identity fabric

The dangerous new single point of failure. If disks die, **accounts become the only container of your existence** — and an account is a thing that can be locked, phished, banned, or forgotten.

- **Passkeys + hardware keys first**, passwords second, SMS never.
- **Two independent recovery identities** (two email providers, two phones, printed recovery codes in two physical places).
- **Key sharding:** the encryption key is split 3-of-5 (Shamir) and each share lives in a *different* provider's private storage + one offline copy. Any 3 reconstruct; 2 shares are useless.
- **Rule:** never let one provider hold both ciphertext and key share. Never let one provider hold your only account recovery path.

---

## 2. The five structural patterns

These are the inventions. Everything else is plumbing.

### Pattern 1 — **Git-as-Disk** (content-addressed persistence)

Git is the closest thing the internet has to a free, universally-replicated, tamper-evident, versioned filesystem. EXOFS treats a repository as a *disk volume*:

- **A commit is a write.** Message = operation, tree = new state, SHA = integrity check.
- **History is free snapshots.** "Undo" works across years. Time travel is `git checkout`.
- **Every clone is a full replica.** The redundancy is a side effect of normal use.
- **Every artifact is a URL:** `https://raw.githubusercontent.com/u/r/<sha>/path` (uncached, 100 MB cap) or `https://cdn.jsdelivr.net/gh/u/r@<sha>/path` (**CDN, permanently cached once served, 20 MB/file on `/gh/`, 50 MB/package** — note HTML is served as `text/plain`, so serve apps as `.html` via Pages, not jsDelivr).

The `@<sha>` form is the important trick: pin to a commit SHA, not a branch, and the URL becomes **immutable and cache-proof** — a permanent, tamper-evident pointer to a file, servable from a global CDN for free, forever.

### Pattern 2 — **BRICK: chunked, hashed, manifest-first storage**

Git's 100 MiB wall and jsDelivr's 20 MB wall both lose to one idea: **never store a file, store bricks.**

```
source file ──► split into 4 MiB bricks ──► sha256 each ──► encrypt each
                                                              │
                    manifest.json (a few KB, lives in git)  ◄──┘
                    { "name": "wedding.mov", "size": 41_231_904_112,
                      "bricks": ["9f2c…","a71b…","3e0d…", …] }
```

Why this is the load-bearing idea:

- **No size limits.** Each brick is 4 MiB. Every lane in the fabric accepts 4 MiB.
- **Dedupe is built in.** Identical bricks dedupe *across every file you own* — a 300 GB photo library is often ~40% duplicate block-level. Hash before upload; skip what exists.
- **Verification is trivial.** Restore = fetch bricks, check sha256, concatenate. Corruption is detected, never silent.
- **Partial restore is possible.** Lose one brick, lose 4 MiB. Re-fetch it from any mirror.
- **Manifests are tiny**, so the *catalog* lives in git (diffable, reviewable, cheap) while the *bulk* lives in Releases / R2 / HF.
- **Repair is cheap:** a scheduled CI job walks manifests, HEADs each brick across mirrors, and re-uploads stragglers. Self-healing by cron.

This is what turns "free online services" from a pile of websites into a filesystem.

### Pattern 3 — **The Mirror Triad**

No artifact has fewer than **three custodians** from **different legal entities and failure modes**:

| Copy | Custodian | Failure mode it survives |
|---|---|---|
| Primary | GitHub (Microsoft, US) | A Cloudflare outage |
| Secondary | Cloudflare R2 + HF Hub | A GitHub account ban / outage / billing dispute |
| Tertiary (cold) | Internet Archive / a second account, different org | Policy change, acquisition, mass takedown |

Rules:
- Mirrors are **cross-org and cross-account** — never three repos under one login. One compromised login must not equal one lost planet.
- **Different formats where possible** (git tree vs object store vs tarball), so one parser bug can't eat all copies.
- **Mirroring runs on a schedule in CI**, not by human willpower. `mirror.yml`, every 6 hours.
- A mirror is not real until a **restore drill has passed** — see Pattern 5/MOLT step 7.

### Pattern 4 — **Diskless execution**

A program needs three things: its bytes, its memory, and a place to put output. Bytes come from a URL, memory is RAM, output goes back over the network. Concretely, in descending order of how much of your old laptop each replaces:

1. **Browser-WASM replaces the OS.** Pyodide for Python, FFmpeg.wasm for media, sql.js for databases, WebContainers for Node. Same CPU, no disk, no install — a tab *is* the process. This is the answer for interactive work.
2. **CI runners replace the server.** A 6-hour, 16 GB Linux box on demand, triggered by a commit, a cron, or a webhook. Non-interactive but enormously capable, and **free and unlimited on public repos**.
3. **Codespaces/Colab replace the dev workstation** when you need a real shell.
4. **'Round-trip state':** since local RAM dies at reboot, long jobs must be *checkpointed to the fabric* every N minutes and made **resumable** — write output bricks as they're produced, update the manifest, commit. A crash should cost you ≤ 1 brick of work. Design every job as a state machine whose state lives in git.

### Pattern 5 — **Encrypt-then-shard (public storage is not private storage)**

Free tiers are mostly public repositories and shared infrastructure. Two non-negotiables:

- **Encrypt before upload**, client-side (age, libsodium secretstream, or openssl from WASM). The provider sees noise. Manifest stores plaintext name + size + brick hashes **only if** that's acceptable; otherwise encrypt the manifest too and keep the name index in a private vault.
- **Key custody ≠ data custody.** Shamir 3-of-5 shares across five independent custodians (two private cloud vaults, one password manager, two offline). **Write the recovery procedure down in plain language and store it with the shares** — the most common disaster in a system like this is not data loss, it's *key* loss, and it's caused by cleverness.
- **Threat model update:** in a world where every file is a URL, *capability to compute becomes capability to leak*. Assume any link you paste is public forever; prefer expiring signed URLs minted by an edge function.

---

## 3. MOLT — the 60-minute exfiltration protocol

The hardest part of the whole event is that it must be executed **while disks still work**, by people who don't believe it's happening yet, over home upload bandwidth, with no time to install anything. So MOLT is ordered by **value density per megabyte**, not by importance-to-you or size.

**Governing rule: bandwidth is the scarcest resource on Earth in this hour. Spend every megabyte on the thing that cannot be recreated.**

### T−60 → T−40 · TRIAGE (do not open any big file)

1. **Create accounts / confirm logins** for: GitHub, Cloudflare, Hugging Face, Google, Internet Archive. Do it from a password manager that syncs remotely (Bitwarden/1Password) — **never** a locally-stored password list.
2. **Enable 2FA and passkeys**, and print/download recovery codes to your phone's *remote* notes, not a local file.
3. **Write the Triage List** — one page, in a cloud doc, answering: *what cannot be recreated?*
   - Identity & legal: IDs, passports, deeds, wills, contracts, tax records, insurance.
   - Keys: SSH/GPG keys, API tokens, password vault export, 2FA seeds.
   - Money: statements, account numbers.
   - Human record: photos/videos of people who have died; children's milestones.
   - Work in flight: uncommitted code, drafts, open documents, half-finished designs.
   - Credentials for everything above.
4. **Freeze new work.** Anything created in the next hour on a local disk is at risk of dying mid-write. Move to a cloud-native tool *now* (Google Docs, GitHub web editor, StackBlitz, perchance) so new work is born on the far side.

### T−40 → T−20 · LIFT THE IRREPLACEABLE (text layer first)

5. **Text beats media, 1000:1.** Push in this order: password vault export → keys → documents → source code → spreadsheets → email archives. This is often < 5 GB and is ~90% of what actually matters.
   - Code: `git remote add exofs … && git push --all` (or upload as a zip through the GitHub web UI, which survives at 25 MB/chunk).
   - Docs: upload to Drive **and** commit an encrypted copy to a repo. Two custodians from the first minute.
6. **Export the browser** — bookmarks, saved passwords, open tabs. **Your bookmarks are now your operating system.** Mirror them to a gist and to Drive.
7. **Log out of nothing and close nothing** — open documents in RAM are still alive. Screenshot/save them to the cloud *before* any reboot. Rebooting now is a data-loss event.

### T−20 → T−5 · LIFT THE BULK (chunked, parallel, deduped)

8. **Chunk and hash first, upload second.** Run the BRICK chunker (locally, these are the last minutes of disk) or let CI do it: zip → split 4 MiB → sha256 → dedupe. Uploading 30% fewer bytes is 18 minutes saved.
9. **Fan out in parallel across three lanes**, because per-connection throughput is the limit, not total bandwidth: lane A → GitHub Releases (biggest assets), lane B → HF Hub or R2 (media/datasets), lane C → Drive (misc private). Saturating three lanes beats queueing in one.
10. **Write manifests as you go** and commit them. **A manifest committed at T−10 is worth more than a file uploaded at T−0.5**, because the manifest tells the future you *what existed* and *how to reassemble it* — even if a brick is missing, you'll know exactly which one and where to hunt for it.
11. **Abandon gracefully.** Anything not lifted by T−5 gets a **regeneration note** in the manifest: `"2024-taxes.pdf — reconstruct from bank portal, 10 min"`. A catalog of what's missing is a plan; silence is a loss.

### T−5 → T+0 · MAKE IT SURVIVABLE WITHOUT YOU

12. Commit and push everything. **Verify the round trip actually happened** — open the repo in a *different* tab and confirm the files render. (Trust the browser, not the app's own "saved" toast.)
13. **Kill anything written to local disk** — stop local builds, close IDEs, end local database writes. Better to lose 4 minutes than to lose a half-written file that corrupts a whole day's recovery.
14. **Screenshot your three URLs** (repo, vault, catalog). Write them on paper. Paper is the only remaining offline storage medium on Earth.

### T+0 → T+24h · BECOME OPERATIONAL

15. **Boot the Ghost Machine:** a static shell on GitHub Pages or Cloudflare Pages + an edge function + object storage. (Reference implementation: `ghostdrive.html` in this folder — a browser-native disk that treats a repo as its volume.)
16. **Run a restore drill from a different machine/account.** Restore one small file, one medium file, and one *brick-sharded* file. **A backup you have not restored is a hypothesis.**
17. **Automate the triad:** turn on `mirror.yml` (see appendix) so every push replicates to R2/HF/archive on a schedule, and a weekly cron verifies every brick's hash and reports gaps.
18. **Publish the catalog as a static site** — a plain HTML index of everything you own, generated from manifests. Human-navigable, no client required, works on a phone.
19. **Teach the reflex:** save = round-trip; green = real. And re-triage weekly — the fabric needs curating, or it fills with dedupe-able noise.

---

## 4. The Ghost Machine (reference architecture)

A complete, free, no-server-of-your-own system: a "computer" assembled entirely from other people's computers.

```
 ┌────────────┐   HTTPS    ┌──────────────────────────────────────────────────────┐
 │  Browser   │ ─────────► │  SHELL      static app · GH Pages / CF Pages         │
 │  (your     │            │             perchance / StackBlitz for live editing  │
 │   new      │            ├──────────────────────────────────────────────────────┤
 │   "disk")  │ ─────────► │  API        edge function · CF Worker / Deno Deploy  │
 │            │            │             auth · routing · signed URLs · metadata  │
 │  WASM:     │            ├──────────────────────────────────────────────────────┤
 │  Pyodide   │            │  STATE      D1/KV · Supabase · Sheets catalog         │
 │  FFmpeg    │            ├──────────────────────────────────────────────────────┤
 │  libsodium │            │  HOT        R2 bucket (10 GB, free egress)           │
 └────────────┘            ├──────────────────────────────────────────────────────┤
        ▲                  │  WARM       git repos (manifests, code, small files)  │
        │  jsDelivr CDN    │             + Releases (2 GiB assets)                 │
        └───────────────── ├──────────────────────────────────────────────────────┤
                           │  COLD       HF Hub · Internet Archive · 2nd account   │
                           ├──────────────────────────────────────────────────────┤
                           │  CPU        GitHub Actions (free, unlimited, public)  │
                           │             scheduled mirroring · chunking · drills   │
                           └──────────────────────────────────────────────────────┘
```

**Read path:** browser → CDN (bricks, cached at edge) → verify sha256 in WASM → render/execute in tab. No server involved for static content, so cost and latency are near zero.

**Write path:** browser → Worker (auth + signed URL) → R2 (brick) → commit manifest to git via API. Three custodians confirmed afterward by CI.

**Failure paths:** GitHub down → jsDelivr still serves cached bricks. GitHub account banned → R2 + HF still hold bricks, manifests re-derivable from a second account. CF down → raw.githubusercontent serves. Everything down → you have manifests, hashes, and the location list, which is a rebuild plan.

**The uncomfortable property, stated plainly:** this system has *no master copy*. It has a *hash* that every copy must match. That's stronger than a master copy — but only if the manifests are mirrored as aggressively as the bricks, and only if someone occasionally *proves* it.

---

## 5. Hybrid fabric: what each site is secretly good at

The user's instinct is correct — the win comes from composing services that were built for something else. A field guide to non-obvious uses:

| Service | Designed for | Secretly useful as | Robustness |
|---|---|---|---|
| **perchance.org** | Random generators | A **no-install app platform**: generators as microservices/plugins, live-editable in-browser, linkable, embeddable. Multiple generators *composed* into one app = an app whose source is a handful of URLs. | ★★★☆ |
| **GitHub** | Code hosting | Immutable content-addressed store + free unlimited CI + a public audit log of every change | ★★★★★ |
| **GitHub Releases** | Shipping binaries | A free CDN for 2 GiB blobs with no bandwidth cap | ★★★★☆ |
| **jsDelivr** | npm/Git CDN | A free global CDN that **permanently caches** what it serves — including as a mirror of last resort for files deleted upstream | ★★★★ |
| **Hugging Face** | ML models | The cheapest large public storage with real download speed; plus free-ish app hosting | ★★★★ |
| **Cloudflare** | CDN | 10 GB of hot object storage with free egress, plus edge compute and SQL | ★★★★★ |
| **Internet Archive** | Web preservation | The cold tier with an institutional *mission* to keep things | ★★★★★ |
| **Google Drive/Sheets** | Office | A private vault; a human-readable database that needs no client | ★★★☆ |
| **Telegram** | Messaging | A rapid human relay of ~2 GB files when APIs are down | ★★ |
| **YouTube/Vimeo** | Video | Cold archive for large media (survives everything, loses fidelity/metadata) | ★★ |
| **npm / PyPI** | Package registry | Immutable, CDN-mirrored blob hosting — *only* for legitimate packages | ★★★ |
| **Notion / Airtable** | Notes | Structured catalogs with built-in UI for non-technical users | ★★★☆ |
| **DNS (TXT / SVCB)** | Name resolution | A tiny, absurdly resilient message bus for pointers and tiny messages — survives when web apps don't | ★★★ |
| **BitTorrent / WebTorrent** | File sharing | Bandwidth *amplification*: N recipients share the cost of delivery. The only way to move internet-scale data for free | ★★★★ |

**Strangeness ladder** — the further down, the more it's a research project and the less you should depend on it tonight: chunk-in-git → CDN-pinned SHAs → attachment relay → data-in-DNS → files-as-video (fountain-coded into a video stream, recoverable from a screen recording) → air-gapped QR/fountain transfer between two phones. All of it works. None of it is where you start at T−60.

---

## 6. What kills this (failure analysis, honestly)

| Risk | Why it's serious | Mitigation |
|---|---|---|
| **Account as single point of failure** | Your entire existence is one login. Bans are automated and appeals are slow. | N-version accounts across orgs; triads; never one login for all copies. |
| **Free-tier abuse detection** | Mass file-hosting on a CI platform or registry is a **ToS violation** and gets accounts terminated — probably *en masse* during a global event. | Store legitimate artifacts, encrypt private data, spread load, pay where you can, read the ToS *before* T−0. **The fabric dies if everyone cheats it.** |
| **Rate limits / quotas** | 100k Worker req/day, 1,000 KV writes/day, 10 GB R2. A million people arriving at once breaks all of it. | Cache aggressively at the CDN; read-mostly design; per-user sharding across providers. |
| **Bandwidth & latency** | The network is now the memory bus. A 4k video edit over WAN is not an edit. | BRICK locality, proxy/thumbnail tiers, WASM local processing of downloaded bricks, expect first-byte pain. |
| **"Free" isn't free** | Free tiers are a customer-acquisition strategy. A global dependency will get re-priced or capped. | Assume any tier can vanish; keep cold copies; keep a paid or self-hosted escape hatch. |
| **Legal/compliance** | Encrypted blobs across jurisdictions, GDPR erasure requests, export control, copyright. | Know what you're storing and where; classify data; keep sensitive material in one jurisdiction you can name. |
| **Key loss** | Total, silent, unrecoverable. | Shamir 3-of-5; written procedure; annual key ceremony; *test* a reconstruction. |
| **Nobody ever restores** | The most common real-world death of backup systems. | Scheduled drills in CI, with a pass/fail report emailed. Make restoration a *test*, not a hope. |

---

## 7. Quota math: how much is "enough"

A single careful identity, without abuse, comfortably holds **tens of GB** (R2 10 GB + Drive 15 GB + HF private 100 GB + GitHub repos/releases in the low GBs + Archive cold). Note the real ceiling isn't aggregate capacity — it's **the labor of curation and the cost of verification**. 20 GB you can restore in an hour beats 2 TB you can't find anything in.

- **Individual:** 10–100 GB is achievable and honest. Prioritize *irreplaceable* over *large*.
- **Household / team:** multiply by identities, then add a shared org account with the triad. Add BitTorrent for fan-out to members.
- **Why not "all of it":** the global corpus of personal data is ~zettabytes; global free tiers are not. Any doctrine that promises "all files" by volume is lying. **The honest promise is: all *irreplaceable* files, provably restorable, plus regeneration plans for the rest.** That is the real solution, and it's the one that survives contact with reality.

---

## 8. 30-day hardening roadmap

| Week | Goal | Deliverable |
|---|---|---|
| 1 | Provable restore | Triad live, one full restore drill passed from a clean machine, key ceremony complete |
| 2 | Automation | `mirror.yml` on schedule, weekly hash sweep, catalog site auto-generated |
| 3 | Usability | Browser-native shell deployed; "save = round-trip" UX; phone-accessible catalog |
| 4 | Stress | Chaos test: revoke one custodian, delete a brick, rotate a key, fail a drill on purpose. Fix what breaks. |

---

## Appendix A — `mirror.yml` (the triad, automated)

```yaml
name: exofs-mirror
on:
  push: { branches: [main] }
  schedule: [{ cron: "0 */6 * * *" }]
  workflow_dispatch:

concurrency: { group: mirror, cancel-in-progress: false }

jobs:
  mirror:
    runs-on: ubuntu-latest        # public repo => free & unlimited
    timeout-minutes: 350          # 6h job cap; stay under it
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }

      - name: Verify every brick (self-healing sweep)
        run: |
          python3 - <<'PY'
          import json,hashlib,os,sys
          bad=0
          for root,_,files in os.walk('manifests'):
            for f in files:
              m=json.load(open(os.path.join(root,f)))
              for b in m.get('bricks',[]):
                p=os.path.join('bricks',b[:2],b)
                if not os.path.exists(p): bad+=1; print('MISSING',b)
                else:
                  h=hashlib.sha256(open(p,'rb').read()).hexdigest()
                  if h!=b: bad+=1; print('CORRUPT',b)
          print(f'gaps={bad}')
          sys.exit(1 if bad else 0)
          PY

      - name: Mirror to R2 (second custodian)
        env: { AWS_ACCESS_KEY_ID: ${{ secrets.R2_KEY }}, AWS_SECRET_ACCESS_KEY: ${{ secrets.R2_SECRET }} }
        run: |
          aws s3 sync bricks s3://exofs-bricks --endpoint-url "${{ secrets.R2_ENDPOINT }}" --size-only

      - name: Mirror manifests to Hugging Face (third custodian)
        env: { HF_TOKEN: ${{ secrets.HF_TOKEN }} }
        run: |
          pip install -q huggingface_hub
          python3 -c "
          from huggingface_hub import HfApi; import os
          HfApi().upload_folder(folder_path='.', repo_id=os.environ['HF_REPO'],
                                repo_type='dataset', path_in_repo='exofs')"

      - name: Restore drill (fetch 3 bricks from a mirror, verify, report)
        run: |
          echo "drill: re-download sampled bricks from R2 and re-hash -> report to catalog"
```

## Appendix B — URL recipes

| Want | Use |
|---|---|
| Immutable file from a repo | `https://raw.githubusercontent.com/<u>/<r>/<sha>/<path>` |
| CDN'd, permanently cached | `https://cdn.jsdelivr.net/gh/<u>/<r>@<sha>/<path>` (≤20 MB/file, ≤50 MB/pkg) |
| Big blob, CDN, no cap | GitHub Release asset URL (`…/releases/download/<tag>/<file>`) |
| Public dataset with real speed | `https://huggingface.co/datasets/<u>/<r>/resolve/main/<path>` |
| Hot object + free egress | R2 custom domain or S3 API (`s3://`) |
| Human-browsable app | GitHub Pages or Cloudflare Pages (1 GB site, 100 GB/mo soft) |
| Ephemeral cloud computer | `gh workflow run …` on a public repo |
| Ephemeral cloud shell | Codespaces (120 core-hrs/mo) |

## Appendix C — browser-only toolbox (no install, ever)

Code: StackBlitz · GitHub web editor · CodeSandbox · val.town · Replit
Compute/notebooks: Pyodide · Observable · Colab · Kaggle
Media: Photopea (images) · Squoosh · CloudConvert · FFmpeg.wasm · Clipchamp
Docs: Google Docs/Sheets · Notion · Excalidraw · tldraw
Files/hash/crypto: CyberChef · any SHA-256 in-browser tool · age/libsodium WASM
Apps in a URL: perchance · Glitch · HF Static Spaces
Version control: github.dev (press `.` on any repo — a full editor in a tab)

---

### The one-sentence version

**Your computer was never the disk. It was the thing that talked to the disk.** Everything that mattered about it was always a protocol, an identity, and a set of hashes — so when the shell fails, keep the protocol, move the hashes into other people's computers, prove you can get them back, and write the recovery procedure down before you need to read it.
