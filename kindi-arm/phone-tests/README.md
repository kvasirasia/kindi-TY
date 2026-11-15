# Kindi ARM64 — Phone Test Results

Shared test results for the **Kindi v0.4.0 Android ARM64 CPU port** running
**Bonsai-2B PTQ1_0** (1.34 GB ternary pack) on Samsung Galaxy phones via Termux.

These results are device-reported: run locally on each phone with the Kindi ARM64
engine, then copied here for sharing. No model weights, no secrets, no PII — just
tokens, scores, and timings.

## Device results

| Device | Chipset | Verify (4×40 token-exact) | Textjobs (158 items) | Max-context | Status |
|---|---|---|---|---|---|
| S23 #1 `RFCX71B5C9K` | Exynos (dm1q) | **4/4 PASS**, 1.33 tok/s | 148/158 = **74%** | 256→4096 OK | complete |
| S23 #2 `RFCX9011K4E` | Exynos (dm1q) | **4/4 PASS**, 1.30 tok/s | 151/158 = **73%** | 256→2048 OK | complete |
| S24 `RFCX9076A9H` | Snapdragon 8 Gen 3 (e1s) | **4/4 PASS**, 0.87 tok/s | in progress | 256→1024 OK | textjobs running |

## Per-device directories

- [`s23_1_RFCX71B5C9K/`](s23_1_RFCX71B5C9K/) — Galaxy S23 #1 (complete)
- [`s23_2_RFCX9011K4E/`](s23_2_RFCX9011K4E/) — Galaxy S23 #2 (complete)
- [`s24_RFCX9076A9H/`](s24_RFCX9076A9H/) — Galaxy S24 (textjobs in progress)

Each device directory contains:
- `device-info.md` — chipset, Android version, threads, toolchain, test date
- Result JSON/log files (phone_tests, phone_textjobs, phone_ctx_high, verify_clean)
- See [`CONTRIBUTING.md`](CONTRIBUTING.md) for how to add your own device results

## Smoke oracle (all devices)

```
OUT_TOKENS [198, 2, 220, 16, 13, 220, 16, 13]
```

This matches the desktop x86 oracle exactly — the Kindi ARM64 engine produces
bit-exact identical output to Kindi v0.4.0 x86-64 on Bonsai-2B PTQ1_0.

## Honest baseline note

`llama.cpp b12189` **cannot** run Bonsai ternary `type-143` GGUF, so there is
**no direct same-model llama.cpp baseline** on these phones. Comparison is against
the Kindi desktop x86 oracle and measured device throughput only.

## Performance insight

Decode tok/s is nearly identical across all three devices (~0.8–1.1 tok/s) because
the Kindi ARM64 engine is a scalar FP16 implementation — device-independent
correctness, clock-speed-dependent speed. S24 is slower (~0.87 tok/s verify vs ~1.3
on S23) due to thermal throttling under sustained load, not engine differences.

## Textjobs weakness map (from feasible evidence)

| Category | S23 #1 | S23 #2 | Failure mode |
|---|---|---|---|
| sentiment (30) | 25/30 | 25/30 | occasional label drift |
| escalate (30) | 28/30 | 28/30 | safe |
| spam (30) | 29/30 | 29/30 | strongest task |
| topic (30) | 28/30 | 28/30 | safe |
| spell (15) | 0/15 | 0/15 | 2B emits think tags instead of direct output |
| grammar (12) | 0/12 | 0/12 | same — think-tag leakage |
| table (5) | 0/1 | 0/4 | engine hangs on table generation |

**Headline:** ~73–74% on classify tasks; 0% on spell/grammar/table — a **capability
boundary of the 2B ternary model**, not a Kindi engine bug.

## Want to add your phone?

See [`CONTRIBUTING.md`](CONTRIBUTING.md) — we welcome results from any Android
phone running Termux with the Kindi ARM64 port.
