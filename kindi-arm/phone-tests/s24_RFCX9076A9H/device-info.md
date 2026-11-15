# Device: Galaxy S24 — `SM-S921B` (serial `RFCX9076A9H`)

**Chipset:** Snapdragon 8 Gen 3 for Galaxy (`e1s`)  
**Android:** 17, debuggable Termux (arm64-v8a)  
**Engine:** Kindi v0.4.0 ARM64 CPU port (`kcpu_arm.c` + `cpu.py`)  
**Model:** Bonsai-2B PTQ1_0 (1.34 GB ternary pack, type-143 GGUF)  
**Threads:** 4 (`OMP_NUM_THREADS=4`)  
**CLANG:** 21.1.8  

## Results included

| File | Description |
|---|---|
| `verify_clean.txt` | Smoke + clean verify (`reference.json`) |
| `textjobs-in-progress.txt` | Status note — full 158-item textjobs still running |

## Key numbers

- **Verify `reference.json`:** 4/4 PASS, 40/40 token-exact, `ALL_PASS 0.87 tok/s` (ctx 1024)
- **Smoke oracle:** `OUT_TOKENS [198, 2, 220, 16, 13, 220, 16, 13]` — matches desktop x86 oracle exactly
- **Prefill tok/s:** ~1.09–1.10 (32–1024)
- **Decode tok/s:** ~0.82–0.91 (g16–256)
- **Write decode tok/s:** ~0.82–0.83 (sentence/paragraph)
- **Max-context (≤1024):** CTX 256/512/768/1024 all OK (226/453/679/922s)
- **Max-context (>1024):** CTX 1536/2048 still running at release time (thermal throttling on sustained load)
- **Textjobs (158 items):** launched 2026-11-15, still running at release time

## Test date

2026-11-11 (provisioned/verified); 2026-11-15 (textjobs launched)
