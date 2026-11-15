# Kindi v0.4.0 — Android ARM64 CPU Port (Bonsai-2B PTQ1_0)

**Release tag:** `v0.4.0-android-arm64`
**Base:** `ahtx/kindi` v0.4.0 (tag `32bc703`)
**Port date:** 2026-11-11 → 2026-11-15
**Scope:** ARM64 (aarch64) CPU backend for Android Termux; x86-64 path preserved unchanged.

**Released by:** Tom Makela (Nickname: TomYam & The Bots) — http://Kvasir.Asia

**Profiles:** https://www.facebook.com/makela.tom · https://www.linkedin.com/in/tommakela/ · https://www.facebook.com/groups/open.bots/

---

## What this release adds

A clean, self-contained ARM64 CPU port of the Kindi 0.4.0 engine that runs the ternary **Bonsai-2B PTQ1_0** model (1.34 GB, type-143 GGUF) on Android phones via Termux — completely offline, no GPU required.

**New files:**
- `kindi/kcpu_arm.c` (443 lines) — scalar ARM64 FP16 kernel (`__fp16`), OpenMP-optional, same layouts/`kc_*` symbols as `kcpu.c`, bit-exact scalar `rows8` first.
- `kindi/cpu.py` — modified to auto-build/load `libkcpu_arm.so` on aarch64 or when `KINDI_ARM=1`; keeps x86 `kcpu.c` + `libkcpu.so` path untouched.
- `docs/ANDROID_ARM64_SETUP.md` — end-to-end offline USB/Termux setup guide.
- `docs/DEVELOPMENT_STATISTICS.md` — how this port was made (time, steps, measurements).
- `docs/ISSUES_AND_RESOLUTIONS.md` — every problem hit and how it was solved.
- `RELEASE-v0.4.0-android-arm64.md` — this file.

**What did not change:**
- `kindi/kcpu.c` (x86-64) — untouched, byte-identical to upstream.
- All other Kindi source files — unchanged.

---

## Acceptance oracle (verified on-device)

| Check | Result |
|---|---|
| Smoke oracle tokens | `OUT_TOKENS [198, 2, 220, 16, 13, 220, 16, 13]` — matches desktop x86 oracle exactly |
| `kindi/reference.json` verify | **4/4 PASS, 40/40 token-exact** on all three devices |
| Max context (incremental) | **256→2048 all OK** on S23 #1 & #2; **3072 & 4096 OK** on S23 #1; CTX 256→1024 OK on S24 |

**Honest baseline note:** `llama.cpp b12189` **cannot** run Bonsai ternary `type-143` GGUF, so there is **no direct same-model llama.cpp baseline** on these phones. Comparison is against the Kindi desktop x86 oracle and measured device throughput only.

---

## Measured performance

| Metric | S23 #1 (SM-S911B) | S23 #2 (SM-S911B) | S24 (SM-S921B) |
|---|---|---|---|
| Smoke load (ctx 512) | ~21.6s | ~21.6s | 39.4s |
| Verify (ctx 1024, 4×40) | **ALL_PASS 1.33 tok/s** | **ALL_PASS 1.30 tok/s** | **ALL_PASS 0.87 tok/s** |
| Prefill 32→1024 tok/s | 1.442 / 1.431 / 1.426 / 1.394 / 1.368 / 1.326 | 1.661 / 1.633 / 1.580 / 1.543 / 1.503 / 1.456 | 1.09–1.10 |
| Decode g16→g256 tok/s | 1.021 / 0.988 / 1.001 / 0.980 / 0.973 | 1.021 / 1.032 / 1.071 / 1.097 / 1.076 | 0.82–0.91 |
| Write decode tok/s | sentence 0.991 / para 0.971 | sentence 1.074 / para 1.077 | 0.82–0.83 |
| Max-ctx ≤2048 | all OK | all OK | CTX 256→1024 OK |

## Full 158-item kbench textjobs (checkpointed, resumed after kills)

| Device | Completed | Pass rate | Wall time | Notes |
|---|---|---|---|---|
| S23 #1 | 148/158 | 110/148 = **74%** | ~2.56 h | killed on final table items |
| S23 #2 | 151/158 | 110/151 = **73%** | ~2.57 h | killed on final table items |
| S24 | in progress | — | — | launched 2026-11-15 |

Per-category (S23 #1/#2): sentiment 25/30, escalate 28/30, spam 29/30, topic 28/30 → classify strong (83–97%). spell 0/15, grammar 0/12 → open-generation correction fails (2B emits think tags instead of direct corrected sentence). table 0/1–0/4 → table items hang.

**Headline:** ~73–74% on classify tasks; 0% on spell/grammar/table — a **capability boundary of the 2B ternary model**, not a Kindi engine bug (token-exact verify passes 4/4; classify is strong).

---

## Files in this release

```
kindi/kcpu_arm.c      # ARM64 CPU kernel (new)
kindi/cpu.py          # auto-build/load libkcpu_arm.so (modified)
docs/*.md             # setup, statistics, issues (new)
RELEASE-v0.4.0-android-arm64.md  # this file
```

---

## License

Kindi v0.4.0 upstream license applies to the port. See `LICENSE`, `NOTICE`, `COMMERCIAL-LICENSE.md`.