# Hardware Measurement Plan — ML-Based Edge Data Reduction (STM32MP257F-DK)

## 1. Purpose & scope
Produces the on-target evidence that answers **RQ1** (does *mapping* or *model choice* dominate latency / energy / accuracy-under-budget) and **RQ2** (how NPU benefit tracks structural match, and the cost of recovering mismatched models on the programmable cores).

Operating principle, three separable jobs that are **not** measured through one another:
1. **Reduction faithfulness** — validated over the edge→PC MQTT link (does the deployed encoder reduce/reconstruct as the off-target harness predicted).
2. **Compute cost** — latency, power, energy, memory measured **locally on each compute unit**, never inferred from MQTT round-trips.
3. **Composition** — the two are combined analytically into energy-per-sample (§6).

**Platform:** 2× Cortex-A35 @ 1.5 GHz · Cortex-M33 @ 400 MHz (FPU+DSP) · Neural-ART NPU 1.35 TOPS (INT8) · 3D GPU (shares its shader array with the NPU).

---

## 2. Placements under test

| Placement | Runtime / stack | Timing source | Energy source |
|---|---|---|---|
| A35 scalar | OpenSTLinux, TFLite (1 thread) | `CLOCK_MONOTONIC_RAW` | external monitor on board rail |
| A35 + NEON | TFLite + XNNPACK, multi-thread | `CLOCK_MONOTONIC_RAW` | external monitor on board rail |
| M33 + CMSIS-NN | bare-metal / RTOS via STM32Cube | DWT cycle counter + GPIO toggle | board rail, GPIO-delimited |
| NPU (Neural-ART) | X-LINUX-AI runtime | end-to-end incl. DMA | board rail, GPIO-delimited |
| GPU (OpenCL) | OpenCL/OpenVX | incl. buffer transfer | board rail, GPIO-delimited |

External monitor = Otii Arc or Joulescope on the board supply, following the MLPerf-Tiny energy-harness pattern. GPU and NPU cannot be exercised concurrently (shared silicon).

---

## 3. Metric catalogue

### A. Compute cost (the RQ1/RQ2 headline axis)
| Metric | Definition | Unit | Source | RQ |
|---|---|---|---|---|
| Inference latency | one forward pass, reported as a **distribution** (median, p95, p99 — not mean) | ms | timing source per placement | RQ1, RQ2 |
| End-to-end per-sample time | preprocess + inference + dequantize/output, incl. data movement on/off the unit | ms | wall timing around the full step | RQ1, RQ2 |
| Energy per inference | rail energy over a GPIO-bracketed inference, **idle-subtracted** | mJ | external monitor | RQ1, RQ2 |
| Average power | mean power during inference | mW | external monitor | context |
| Peak power | max instantaneous power during inference (supply sizing for battery/solar) | mW | external monitor | context |

### B. Memory (three genuinely different numbers)
| Metric | Definition | Unit | Source | RQ |
|---|---|---|---|---|
| Peak runtime RAM | working set / arena high-water during inference | KB | RSS (A35) / `.map` + arena (M33) | RQ1, RQ2 |
| Model footprint | weights in flash/storage | KB | artifact size | RQ1 |
| Runtime/library overhead | X-LINUX-AI / CMSIS-NN stack resident cost | KB | RSS delta / map | RQ2 (feasibility) |

### C. System / composite (closes the prior paper's open question)
| Metric | Definition | Unit | Source | RQ |
|---|---|---|---|---|
| Transmit probability `p_tx` | fraction of samples that trigger a send | — | Stage-3 harness, per (model, ε, regime) | RQ1 |
| Energy per transmission `E_tx` | energy to publish one payload on the **real radio** | mJ | radio measurement or datasheet model | RQ1 |
| **Energy per sample** | `E_infer + p_tx · E_tx` — the project's decisive metric | mJ | composed | RQ1 |
| Bytes per sample | payload bytes × `p_tx` | B | harness | RQ1 |

### D. RQ2 mechanism (direct, not inferred)
| Metric | Definition | Unit | Source | RQ |
|---|---|---|---|---|
| Operator-mapping coverage | share of the graph that ran on Neural-ART vs fell back to CPU | % ops / % MACs | ST Edge AI deployment report | **RQ2** |
| Fallback op count | number/type of operators not accelerated | count | deployment report | RQ2 |

### E. Correctness (the bridge between the two project halves)
| Metric | Definition | Unit | Source | RQ |
|---|---|---|---|---|
| On-device accuracy | TAD / MAD / MD recomputed with the **on-device INT8** model in the reduction loop | ppm | harness with board in loop | RQ1, RQ2 |
| Float→INT8 delta | on-device accuracy minus host-float accuracy | ppm / % | comparison | RQ2 |
| Reconstruction accuracy under loss | TAD / MAD / MD (and plume-window MD) vs observed packet-loss rate, **per reconstruction rule** (hold-last vs mirror) | ppm | harness over WLAN, loss-rate logged | robustness |

### F. Context & controls (logged, not headline)
| Metric | Why | Source |
|---|---|---|
| Load / init time | one-off model load; can dominate for sporadic (every-few-seconds) inference if not kept resident | timing |
| Latency jitter / determinism | matters more than the mean for the real-time-core deployment | distribution spread |
| Packet-loss rate | per-run WLAN loss/retransmission; an **uncontrolled variable** that must be recorded alongside every reduction/accuracy result so it isn't mistaken for a model/ε effect | broker counters vs publish log |
| Thermal / throttling | sustained-run drift that scatters every other metric | on-die temp, governor logs |
| DVFS / governor state | must be fixed or recorded to make runs comparable | sysfs |

---

## 4. Instrumentation per placement

**A35 scalar & A35 + NEON (Linux).** Time with `CLOCK_MONOTONIC_RAW` around the inference region. Pin to an isolated core (`isolcpus` + `taskset`) and fix the cpufreq governor to `performance` (record the frequency); note SMP behaviour for the multi-thread NEON case. Memory from RSS high-water and the TFLite arena. Confirm NEON/XNNPACK kernels are actually engaged (not a scalar fallback) before trusting the NEON column.

**M33 + CMSIS-NN (bare-metal coprocessor).** Time with the DWT cycle counter; cross-check with a GPIO toggle read on a logic analyzer/scope (the Linux side cannot time the M33). Flash and RAM from the linker `.map` plus the CMSIS-NN buffer arena — this column is where "✗ too large" cells are confirmed against the M33's SRAM. Energy is rail-level and shared with the rest of the SoC, so bracket with GPIO and subtract a measured idle baseline; acknowledge the isolation floor.

**NPU (Neural-ART via X-LINUX-AI).** Time **end-to-end including DMA on/off the accelerator** — the invocation and setup overhead lives outside the kernel and is exactly what makes the NPU lose on tiny models. Pull operator-mapping coverage from the ST Edge AI compilation report (this is the RQ2 evidence). GPIO-bracket for energy; capture the cold first-call cost explicitly.

**GPU (OpenCL).** Time including buffer transfer; energy GPIO-bracketed on the rail. Run as a secondary comparison only, and never alongside the NPU (shared shaders). Expected to be where recurrent models (GRU) may beat the NPU.

---

## 5. Measurement protocol

- **Warm-up & repetitions:** discard the first N cold iterations for the steady-state figure; run ≥ M repetitions (M large enough to support the RQ1 variance decomposition); report the full distribution, not a mean.
- **Two duty-cycle regimes, both reported:**
  1. *Looped / steady-state* — back-to-back inference, warm. The conventional benchmark number.
  2. *Sporadic / realistic* — single-shot at the true sampling cadence (every few seconds), cold. The deployed reality for a data-reduction workload.
  The **gap between them** is a result in itself: on NPU/GPU the accelerator may never reach steady state in deployment, so the realistic figure is overhead-dominated.
- **Energy method:** GPIO marker brackets the measured region → integrate power from the monitor → subtract a separately measured idle baseline → report per-inference and per-sample. Keep monitor timestamps aligned to the GPIO markers.
- **Controls:** fix and log governor/clock; log on-die temperature; include a sustained-run check for throttling.

---

## 6. Composite: energy per sample
The decisive number the prior paper lacked:

> **E_sample = E_infer + p_tx · E_tx**

`E_infer` from §3A (per placement), `p_tx` from the Stage-3 harness (per model, ε, regime), `E_tx` from the **real** transmit path. Because the validation LAN is lossless and unmetered, `E_tx` must come from the actual radio module (cellular/LoRa) or a per-byte datasheet model — state which. Whether reduction is a *net* energy win then depends on how one inference compares to one transmission, evaluated per placement: a model that is cheap to run but poorly mapped (GRU on NPU) may still lose to the same model on A35+NEON once `E_infer` is included.

---

## 7. Operator-mapping coverage (RQ2)
From the ST Edge AI / Neural-ART compilation report, per model: percentage of operators and of MACs executed on the NPU versus offloaded to CPU, plus the list of unsupported operators. This is the mechanistic explanation behind any NPU latency/energy penalty — expected to show the GRU's recurrent operators falling back — and is stronger evidence than inferring mismatch from timing alone.

---

## 8. On-device accuracy after INT8
Re-run the reduction harness with the on-device INT8 model in the loop and recompute TAD/MAD/MD; compare to the host-float results per placement. Early host data suggests the INT8 penalty is small, but it is a deployment cost to *measure*, not assume, and it is where the data-science accuracy meets the hardware energy.

---

## 9. Data recording schema
One tidy long-format row per measurement, to support per-factor analysis and the RQ1 variance decomposition:

`model · config · placement · regime · duty_regime · metric · value · unit · n_reps · median · p95 · run_id · timestamp · governor_state · die_temp · link_type · packet_loss_rate`

---

## 10. Validity & caveats
- **WLAN is a middle-ground link, not the real one.** Validation runs over WLAN — lossier and more variable than Ethernet, far less so than the cellular/LoRa link the harbor node will use. It sits usefully between the two: lossy enough to exercise real gap-fill and reconstruction behaviour, but not representative of the deployment radio. Consequences:
  - *Correctness transfers; energy does not.* WLAN faithfully validates reduction and reconstruction (a dropped publish exercises exactly the decoder paths the real link will). It says nothing about `E_tx` — WLAN's per-transmission energy and RF profile are unlike cellular/LoRa, so the energy-per-sample composite (§6) still sources `E_tx` from the real radio or a datasheet model.
  - *Loss is an uncontrolled variable — log it.* WLAN loss is bursty and interference-dependent, so a run can look clean or terrible by the hour. Packet-loss rate (§3F) is recorded with every result; otherwise it contaminates accuracy numbers attributed to model and ε.
  - *Separate headline from robustness.* Run the headline reduction/accuracy results on the lossless path so model and ε are the only moving factors; treat WLAN as a distinct, characterized robustness condition with loss rate measured per run.
  - *Free robustness result.* WLAN gives a real loss channel to compare reconstruction rules under loss (§3E): hold-last freezes at a stale value when a transmission is lost, while mirror keeps predicting — and a miss during a plume event is the worst case for both. Quantifying that gap on a real link is stronger than injected-loss alone.
- **Shared SoC rail.** Isolating M33/NPU/GPU energy relies on idle subtraction; report the baseline-noise floor.
- **GPU/NPU contention.** Shared shaders — measured separately, never concurrently.
- **Thermal scatter & DVFS.** Fixed/logged; otherwise they dominate variance.
- **INT8-only NPU.** Some operators may not map; coverage (§7) records this.
- **External validity.** Single board, single site, single workload class — scope claims accordingly.

---

## 11. Test matrix (reference)
Models × placements and the deliberate size sweep are defined in Part 3 of the project design. In brief: a Tier-1 spine of {1D-CNN, GRU, TCN, MLP} across {A35 scalar, A35+NEON, M33, NPU}; baselines (persistence, moving-average, Ridge) on {A35, M33} as the energy floor; one neural representative per family on GPU; and a **size sweep** (deployed model + ~10× + ~100×) for the conv and recurrent families across {A35+NEON, NPU, GPU} to locate the accelerator break-even and show where the deployed CO₂ models sit relative to it.
