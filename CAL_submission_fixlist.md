# CANavigator → IEEE CAL: Pre-Submission Fix List

**Repo:** `Vikhyat-Chauhan/CANavigator`
**Paper:** `CA_IEEE_CAL_Paper__FULL__.pdf`
**Target:** IEEE Computer Architecture Letters (CAL)

**Bottom line:** Feasibility analysis and the time result are publishable. One scientific blocker (compute-energy framing) plus stats-wording and emulated-latency transparency must be fixed before submission. Everything else is code↔paper consistency and artifact hygiene.

---

## Venue constraints (hard)

- Max **4 pages incl. figures, tables, references**. Over-length returned without review. Current draft ~3 pp → ~1 pp headroom; all fixes must fit inside it.
- First decision almost always **Revise&Resubmit** (= major revision), 6-week window. Pre-submission quality of P0 items determines R&R-vs-reject.
- Verify **blinding requirement** on CAL author-info page. PDF currently exposes author names + VT affiliation.
- **IEEE DataPort** dataset upload encouraged — do only AFTER P3 artifact cleanup.
- Scope explicitly includes "power-aware computing" → reviewers are energy specialists. Assume the energy claim gets scrutinized.

---

## P0 — Blockers (fix or risk reject)

### P0.1 Compute-energy claim is not independent of time
- **Paper:** Title; Abstract ("compute energy by 34.52%"); §V-B; Table III "Compute energy 148.92→97.51"; Conclusion; contribution #4.
- **Code:** `ca_navigator/tools/orin_nx_cycle_model.py:225–251` (`latency_to_energy_j`): `E = p_idle*wall_s + (TDP−p_idle)*active_s/N_cores`.
- **Problem:** Decomposition on paper's own numbers:
  - APE1: 148.94 J = idle 148.90 J (2.5 W × 59.56 s) + active **0.044 J** (0.03%).
  - CA:   97.66 J  = idle  97.45 J (2.5 W × 38.98 s) + active **0.205 J** (0.21%).
  - −34.52% (energy) and −34.55% (time) are the same quantity. The 0.03% gap is the tell, and it is in the abstract.
  - Active-only energy REVERSES the claim: CA 0.205 J > APE1 0.044 J (~4.6×, CA runs 3 planners). The result depends entirely on counting idle power, which makes it equal to the time result.
- **Fix (choose one):**
  - (A) Reframe + retitle. Claim becomes: "CA cuts mission time ~34%; compute energy is idle-bound on this platform, so parallel-arbitration marginal cost is sub-joule → planning depth is ~free on compute energy." Drop "Simultaneously … Compute Energy" from title; drop "in contrast to the usual trade of performance against power."
  - (B) Replace modeled latency with measured Orin NX latency so the active term is non-trivial and the dual claim is genuine (large lift; = your "hardware deployment" future work).
- **Recommended:** (A) for this submission.

### P0.2 Latencies are modeled + injected, presented as measured
- **Paper:** Table I ("Native latency @ 2.0 GHz 523/1343/2035 µs"); §IV-C.
- **Code:** APEs are `time.sleep(budget_ms/1000)` — `nav_algorithm_T.py:557,566,577`. Budgets `nav_algorithm_T.py:114–116`. Source model `orin_nx_cycle_model.py:102–112` (`_L`), `:195` (`_latency_us`), `:201–208` (`APE_LATENCY_US`), `:214` (`DEADLINE_SCALE`).
- **Problem:** Reads as profiled hardware; is a static op-count model injected as delay. `_L` per-op table is unit-mislabeled (see P1.3).
- **Fix:** State plainly that APE latencies are analytically modeled and injected. Lean on §IV-C invariance argument (result needs only APE1<APE2<APE3 and APE1 budget < min deadline < … < max deadline ≥ APE3 budget). This is the strongest defense — make it explicit, not a footnote.

### P0.3 Statistical independence overstated
- **Paper:** §IV-A ("1,000 runs per strategy", "10 replications"); Tables II–III p-values.
- **Code:** Seed cycle `main.py`: `seed = world_gen_seed_offset + ((attempt_idx−1) % 100) + 1` → ≤100 distinct worlds. Determinism: event seed 42, `physics_seed=42`, `event_deterministic=True` → repeated worlds are bit-identical, add zero statistical power.
- **Problem:** ≤100 independent worlds, not 1,000. p<0.0001 computed on inflated N.
- **Fix:** (a) State comparison is **paired per identical world**; (b) report significance on the independent unit (paired test, N = distinct worlds); (c) either generate 1,000 genuinely distinct seeds or stop calling them 1,000 independent runs.

---

## P1 — Should-fix (code↔paper mismatches; visible if artifact released)

### P1.1 APE3 over-described
- **Paper:** §III APE3 bullet ("Scores candidate headings across a sweep range … arc-safety … NFZ proximity penalties"); Fig. 1 "APE3: corridor navigator".
- **Code:** `nav_algorithm_T.py:576–592` (`_evt_plan_ape3`) = confidence-scaled sidestep, same `sidestep_deg`/`_side_bias` as APE2 (`:583`). Sweep planner `_choose_heading` is `nav_algorithm_T.py:510`, called only in base loop `:958`, never by an APE.
- **Fix:** Describe APE3 as it runs (forward-clearance/gap/arc-safety confidence → scaled sidestep), OR wire `_choose_heading` into `_evt_plan_ape3` (cost model already charges for it at `orin_nx_cycle_model.py` `_CONFIDENCE`/`_APE3_UNIQUE`).

### P1.2 APE1 baseline fairness
- **Paper:** §III APE1; Table III (APE1-vs-CA is the only valid comparison).
- **Code:** `_evt_plan_ape1` `nav_algorithm_T.py:559` `slow_frac=0.05` (5% speed reflex).
- **Problem:** Only-feasible-competitor runs at 5% speed → looks strawmanned; CA-beats-its-own-fallback.
- **Fix:** Justify 5% as realistic or add sensitivity to `slow_frac`. State explicitly the comparison is CA vs its APE1 fallback.

### P1.3 Table I cycles↔latency inconsistent ~1000×
- **Paper:** Table I pairs "~1,046/2,686/4,070 cycles" with "523/1343/2035 µs".
- **Code:** `APE_LATENCY_US` = 523.48/1342.55/2034.81; implies ~1,046,960/2,685,100/4,069,620 cycles at 2.0 GHz, not ~1,046. `_L` (`orin_nx_cycle_model.py:102–112`) labeled µs but values read as ~ns/cycle costs; unit drifts ns→µs→ms through to `ape1_budget_ms=523`.
- **Fix:** Make Table I self-consistent (either ~1.05M cycles @ 523 µs, or ~1,046 cycles @ 0.523 µs + explicit budget-scaling step). Pick one unit chain, label end-to-end.

### P1.4 Inter-arrival vs deadline range conflated
- **Paper:** §IV-B ("log-uniform inter-arrival over [0.60, 3.50]s").
- **Code:** inter-arrival log-uniform [0.90, 4.0]s (`event_emitter.py:25,29`; `_draw_dt :172–174`); [0.60, 3.50]s is the DEADLINE clamp (`:44–45`; `_deadline_from_dt :176–180`, `deadline = clamp(0.85·Δt, [0.60,3.50])`).
- **Fix:** "Inter-arrivals log-uniform [0.90, 4.0]s; deadline = clamp(0.85·Δt, [0.60, 3.50])s."

### P1.5 CA-specific APE3 threshold undocumented
- **Paper:** §III/Table I imply APE3 selected at 2035 budget.
- **Code:** `main.py:64` CA-only `ape3_select_threshold_ms=2589` (+554 ms margin) vs default 2035 (`nav_algorithm_T.py:124`). Shifts tier split (code comment `main.py:58–64`: APE1≈61/APE2≈23/APE3≈16%) away from paper's ~30% APE3.
- **Fix:** Document the 2589 ms margin + resulting split, or revert to 2035.

---

## P2 — Mechanism wording (low risk, tighten if space)

### P2.1 Selector is pre-decided, not raced
- **Paper:** Abstract/§III ("commits whichever candidate is ready when … deadline forces selection").
- **Code:** Winner fixed at intake from deadline alone — `_evt_winner_for_deadline` `nav_algorithm_T.py:605–614`; loop then waits for that APE (comment `:901–902` "no fallback logic"). Outcome-identical because budgets are deterministic.
- **Fix:** Note selection is computed at intake from the deadline (equivalent to racing under deterministic budgets), or leave — cosmetic only.

---

## P3 — Artifact hygiene (before DataPort/arXiv upload)

- `config.py:8` `simulation_runs = 10` → set to paper value (1000) for the release run.
- `config.py:96–98` `deadline_min_s=0.70` is DEAD: `main.py:179–182` builds `EventCfg(seed=42, event_deterministic=True)` instead of `EventCfg.from_teleop_cfg(cfg)`, so live floor = 0.60 (`event_emitter.py:44`). Delete the config field or call `from_teleop_cfg`; reconcile 0.60 vs 0.70 (paper uses 0.60 — correct).
- Hardcoded path `/home/vikhyat/Documents/Hydra/worlds/...` in `config.py` → make relative/parameterized.
- Empty `PX4-Autopilot/` dir → remove or populate as submodule.
- Ensure released config + seeds reproduce Tables II–III exactly.

---

## Pre-submission checklist

- [ ] P0.1 retitle + reframe §V/abstract/contribution #4; Table III split idle vs active compute
- [ ] P0.2 state latency is modeled+injected; foreground §IV-C invariance
- [ ] P0.3 paired-by-world stats; correct N; fix "1,000 independent runs" wording
- [ ] P1.1 APE3 prose matches `_evt_plan_ape3` (or wire in `_choose_heading`)
- [ ] P1.2 justify/sensitivity `slow_frac=0.05`; state CA-vs-fallback
- [ ] P1.3 Table I cycles↔latency consistent, single unit chain
- [ ] P1.4 fix inter-arrival vs deadline ranges
- [ ] P1.5 document or revert 2589 ms CA threshold
- [ ] P2.1 (optional) selector wording
- [ ] P3 config: runs=1000, dead deadline_min, hardcoded path, empty PX4
- [ ] ≤ 4 pages incl. refs after edits
- [ ] blinding requirement verified
- [ ] DataPort dataset prepared post-cleanup
