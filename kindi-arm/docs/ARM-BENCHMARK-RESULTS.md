# Kindi ARM64 — benchmark results, notes & workarounds

**Consolidated test record — 2026-11-17/18.** All numbers measured on-device with the v3 int16-LUT kernel
(`-DKINDI_I16_ROWS8`) unless noted. Companion to `RELEASE-arm-v3-int16-lut.md` (the share doc),
`ARM-FINALIZATION-2026-11-17.md`, and `THERMAL-AND-SUSTAINED-PERFORMANCE.md`. Raw JSON in
`../phone-tests/artifacts/`.

**Controlled conditions:** big cores only (taskset), 4 OpenMP threads, screen awake + `svc power stayon true`,
Termux `run-as`. Decode = pure generation (g64/g256 = 64/256 generated tokens); prefill = prompt ingestion
(n256/n1024). Verify = token-exact vs the per-size x86 oracle (`reference.json` / `-4b` / `-9b`).

---

## 1. The optimization ladder (Bonsai-2-2B, S23 #1 big cores)
| Variant | Flag | decode g64/g256 | correctness | status |
|---|---|---|---|---|
| v0 scalar FP16 | *(none)* | 3.29 / 3.17 | bit-exact | original port |
| v1 branchless | *(none, default)* | 8.06 / 7.77 | bit-exact | safe fallback |
| v2 NEON fp32 | `-DKINDI_NEON_ROWS8` | 6.45 / 6.21 | bit-exact | shelved (shuffle-bound) |
| **v3 int16-LUT** | **`-DKINDI_I16_ROWS8`** | **15.33 / 14.67** | token-exact* | **FINAL (4.66×)** |
| v3.1 streamed | `-DKINDI_I16_STREAM` | 6.05 / 5.83 | token-exact* | shelved (slower than v3) |

*v3/v3.1 are token-exact on the 2B; on 4B/9B they flip one int8 near-tie (see §3).

## 2. Fleet results by model size (v3)
### Bonsai-2-2B
| Device | SoC | decode g64/g256 | prefill | verify |
|---|---|---|---|---|
| S23 #1 | SD 8 Gen 2 | 15.33 / 14.67 | 19.3 / 18.3 | 4/4 |
| S24 | Exynos 2400 | 14.19 / 16.81 | 21.6 / 20.0 | 4/4 |
| A32 | Dimensity 720 | 6.52 / 6.40 | — | 4/4 |

**2B textjobs (158 items) == fp32 v0:** sentiment 25/30, escalate 28/30, spam 29/30, topic 28/30;
spell/grammar/table/notes/news 0 (the 2B capability ceiling, identical in v0 — not an engine defect).

### Bonsai-2-4B
| Device | decode g64/g256 | prefill | verify | note |
|---|---|---|---|---|
| S24 | 7.20 / 6.65 | 7.42 / 6.18 | 39/40 | cool start 25°C, stable affinity 4–9 |
| S23 #1 | 5.59 / 5.35 | 6.02 / 5.65 | 39/40 | **thermally throttled** (37°C, ~1.2/2.8 GHz) |

Load ~50 s. Verify: prompt 0 = 39/40 (one int8 near-tie at pos 24, re-converges); prompts 1/2/3 = 40/40.

### Bonsai-2-9B
| Device | decode g64/g256 | prefill | verify | note |
|---|---|---|---|---|
| S24 | 2.62 / 2.56 | 2.93 / 2.88 | 39/40* | warm 35→41°C; **load 156 s**; fits 8 GB w/ freed RAM |

*9B verify per prompt: 0=40/40, 1=**39/40** (near-tie), 2=**13/13** (expects only 13 tokens), 3=40/40.
fp32 v1 fully exact (40/40/13/13/40/40). Cool-start decode ≈ 3.4 expected.

### Scaling note
Decode is **bandwidth-bound**: 2B→4B→9B weights ≈ 0.5→1.1→2.4 GB, decode ≈ 15→7→2.6 tok/s (~½ per step).
Cooling *sustains* the peak, doesn't raise it. 27B (~7 GB weights) is too large for phones.

---

## 3. Correctness notes (read before trusting a number)
- **v3 is not bit-exact** — it quantizes activations to int8 (NMSE ~1.6e-4). On the 2B it is token-exact
  everywhere tested; on 4B/9B it flips **one deterministic near-tie token** in the verify set (39/40). The LUT
  math itself is exact; the only approximation is the int8 activations. **Use v1 (no flag) when you need bit-exact.**
- **Verify prompts have different expected lengths.** 9B prompt 2 expects 13 tokens, not 40 — a "13" match is
  13/13 EXACT, not a divergence. **Always check `len(expected)` per prompt before calling a short match a failure.**
- **A tok/s without thermal state is misleading.** Report cool-start vs sustained; `g256 < g64` means the device
  warmed mid-run. Log battery temp + start/end `Cpus_allowed_list` (the runner records these).

---

## 4. Workarounds & gotchas (the practical knowledge)
| Problem | Workaround |
|---|---|
| **SIGBUS on 4B/9B/27B** | Fixed in kernel: `gemv` scale arrays sized to `G0=C/128` (was fixed `[64]`; 4B ffn-down G0=72 overflowed). If you fork an older `kcpu_arm.c`, apply the VLA fix. |
| **Android shell confined to mid/little cores** | `taskset` the big-core mask: **S23 `f8`** (cores 3–7), **S24 `3f0`** (4–9), **A32 `c0`** (6–7). Screen awake + `svc power stayon true` so the app cpuset stays on big cores. |
| **Knox work profile / Secure Folder hold RAM** | adb **cannot** force-stop user 10/150 ("Shell does not have permission to access user 10"). **Manually close** those apps on the phone; a reboot + close gets ~3.6 GB free on the 8 GB S24. |
| **9B needs RAM headroom** | Weights = **2.41 GB RAM** (pack's 4.2 GB disk includes source `model.gguf`, not loaded). Free **~3 GB+** before loading; load takes 156 s. |
| **Thermal throttling** | Cool the device between runs (screen off, idle) or use a clip-on Peltier cooler (~$15–30). **No fridge** (condensation). Don't override the governor (cooks the battery). See `THERMAL-AND-SUSTAINED-PERFORMANCE.md`. |
| **Need bit-exact output** | Build with **no** `-DKINDI_I16_ROWS8` (v1 branchless default). |
| **Struct mismatch / `kc_sizeof_model` assert** | Deploy `cpu.py` + `kcpu.c` + `kcpu_arm.c` **as a set** (the `s01` field changed the Model struct). |
| **Stale kernel after changing flags** | `rm -f $KINDI_CPU_CACHE/libkcpu_arm.so` to force a rebuild. |
| **Wrong-size oracle** | Set `KINDI_BENCH_REF` to the per-size reference (`reference-4b.json`, `reference-9b.json`). |
| **`run-as` fails after reboot** | Launch Termux once (`am start -n com.termux/.app.TermuxActivity`) before `run-as`. |
| **OpenMP link** | Termux needs `libomp` (`pkg install libomp`; static `libomp.a`). |

---

## 5. Reproduce (quick ref)
```sh
# inside run-as com.termux, big cores, v3:
export KINDI_DEVICE=cpu PYTHONPATH=$HOME/kindi-android-test KINDI_CPU_CACHE=$HOME/.cache/kindi
export OMP_NUM_THREADS=4 CC=clang
export KINDI_CPU_CFLAGS="-O3 -march=armv8.2-a -fopenmp -fno-math-errno -ffp-contract=off -DKINDI_I16_ROWS8"
export KINDI_BENCH_REF=$HOME/kindi-android-test/kindi/reference-9b.json   # per size
rm -f "$KINDI_CPU_CACHE/libkcpu_arm.so"
taskset 3f0 python3 runner_kindi.py "$HOME/kindi-pack-9b" v3-9b out.json   # mask per device
```
Wrappers: `../run_variant.sh` (2B), `../run_4b.sh`, `../run_9b.sh` (on-device). Full steps in the release note.

## 6. Open (not yet measured)
- **4B/9B textjobs** (only verify done; the 2B textjobs gate is the fully-validated one).
- **Multilingual / creative prose** domain (textjobs is technical/classification).
- **Repeated-run variance bound** on the 4.66× (single runs so far).
- **Duty-cycle thermal decay sweep** (5–10 min continuous, cool vs rest).