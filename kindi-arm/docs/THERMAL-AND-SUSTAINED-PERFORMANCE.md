# Thermal throttling & sustained performance (phone inference)

**Dev-log note — 2026-11-17.** Captures what we learned about heat while benchmarking Bonsai-2 on
Galaxy S23 / S24, so nobody re-derives it. Companion to `ARM-FINALIZATION-2026-11-17.md` and
`RELEASE-arm-v3-int16-lut.md`.

## The observation that prompted this
The 4B decode measured **5.59 tok/s on S23** but **7.20 tok/s on S24**. The difference was **heat, not
the kernel**: S23 had just run a long benchmark (battery 37.9°C, big cores throttled to **1.2 GHz vs
2.8 GHz max**), while S24 started cool (25.4°C). The cool-start number is the real one.

## What the device's thermal config actually shows (S23, read from sysfs)
- CPU thermal zones' **hard trip point = 115°C** (`/sys/class/thermal/thermal_zone*/trip_point_0_temp
  = 115000`). That is the silicon's *emergency shutdown*, **not** the throttle threshold.
- Throttling is done by the **thermal governor** much earlier, through two cooling-device families:
  - `cpufreq-cpu0` / `cpufreq-cpu3` — drop CPU frequency states.
  - **`thermal-pause-*` / `pause-cpu*`** — literally *pause (offline) cores*. This is the affinity drift
    we logged (`Cpus_allowed_list 3-7 → 3-4,6-7`): core 5 was thermally paused mid-run.
- The governor triggers on **skin / battery temperature** (~37–40°C battery in our runs), **not** the
  115°C silicon limit. The chip can take 115°C; the **chassis and battery** cannot.

## "Would a fridge (10–20°C) stop the throttling?"
Cooler ambient lets the phone sustain boost longer (heat escapes faster) — in principle yes. **But a
fridge is a bad idea: condensation.** A phone below the dew point of the surrounding air condenses water
on/in it → short circuit. Pulling a cold phone into humid room air is the classic hardware killer. The
safe version is **active cooling at room temperature** (stay above dew point), not a fridge.

## "Can we just let it run to 45–50°C?"
Blocked by **safety, not the silicon**:
- **Skin** comfort/safety limit ≈ 43–45°C — the governor throttles to stay under it.
- **Battery** is the binding constraint: Li-ion sustained >45°C degrades quickly; >50°C is unsafe.
  Our 37.9°C battery was already near the trigger.
- Overriding the governor needs **root** (these phones aren't rooted); the PMIC still force-shuts at
  ~115°C regardless; and you would cook the battery. **Not recommended.**

## Ways to push sustained performance (ranked)
1. **Active cooling — the real fix.** A clip-on Peltier/semiconductor phone cooler (~$15–30) or a small
   fan on the back keeps skin/battery cool so the SoC *sustains* boost. This is why gaming phones
   (RedMagic/ROG) ship with fans. It would make **g256 ≈ g64** (remove the warm-up decay).
2. **Remove the case + a cool (~20°C AC) room** — free, modest help.
3. **Performance/game mode** (e.g. Samsung Game Booster) — relaxes throttling a little, still respects
   skin/battery safety.
4. Root + thermal override — risky, battery danger, **don't**.

## The wall cooling CANNOT break: memory bandwidth
Decode is **bandwidth-bound**, evidenced by our own scaling: the 4B has **~2.2× the ternary weights** of
the 2B (bcodes 1.05 GB vs 470 MB) and decodes at **~half** the rate (15 → 7 tok/s). That is the
bandwidth signature — the token rate is capped by how fast weights stream from LPDDR5X (tens of GB/s),
not by CPU frequency. **Cooling lets you *sustain* the peak; it does not *raise* it.** Our cool-start
numbers (2B ~15, 4B ~7 tok/s) are already near that ceiling.

## Benchmark-methodology implications (do this going forward)
- **Report cool-start vs sustained separately.** A single number without thermal state is misleading.
- **Log battery temp before/after** and the **start/end `Cpus_allowed_list` + thread count** (the runner
  already records these). A run that ends throttled/paused is not a clean peak measurement.
- **Let the device cool between runs** (screen off, idle) or use active cooling for a sustained figure.
- The **g64 vs g256** pair is a built-in thermal probe: if g256 < g64, the device warmed during the run.

## Fleet implication
This is the same reason the fleet doctrine treats phones as **low-duty / duty-cycled workers**, not
continuous inference servers: sustained load needs cooling or rest periods. A phone left to run flat-out
will throttle to a fraction of its cool-start rate (we saw ~1.2/2.8 GHz ≈ 40% frequency).

## Open (not yet measured)
- A **duty-cycle sweep**: tok/s decay over a 5–10 min continuous run, and how much rest intervals or a
  cooler recover. Would quantify exactly what cooling/pacing buys, without buying anything.