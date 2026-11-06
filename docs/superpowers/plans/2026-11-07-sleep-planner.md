# Sleep Planner (ช่วงสอบ) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extend `Sleep_Planner.ipynb` so a student enters several coffees in one day and gets the time they can sleep, graph A/B, a Sequence & Series section, a Probability (Monte Carlo) section, and 3 runnable test cases.

**Architecture:** Everything stays in the one notebook (it is the graded deliverable). Pure functions in their own cells (model → series → probability → plotting), then I/O (`get_inputs`, `show_result`, `main`), then markdown explanation, then 3 test-case cells with `assert`, and the interactive `main()` call as the **last** cell, tagged `demo`. Tests live in the notebook because the rubric requires tests "shown in the Notebook".

**Tech Stack:** Python 3.9 (Anaconda Jupyter), `math`, `datetime`, `numpy`, `matplotlib` only.

**Spec:** `docs/superpowers/specs/2026-11-07-sleep-planner-design.md`

## Global Constraints

- Deliverable file: `Sleep_Planner.ipynb`. Do not modify `Formula.ipynb` or `GUI.ipynb`.
- Imports allowed: `math`, `datetime`, `numpy`, `matplotlib` — nothing else.
- Keep the existing model: Vd = 0.6·W; CL = 0.085·W (0.06·W if age ≥ 65); t½ = ln2·Vd/CL; kₑ = ln2/t½. Keep `COFFEES`, `convert_time`, `calc_pk`, `calc_vd`, `calc_cl`, `calc_half_life`, `calc_ke`, `calc_c0`.
- Thresholds: `SLEEP_MG = 50` (main), `STRICT_MG = 30` (dashed faint line + secondary time only).
- Monte Carlo: `N = 10_000`, seed `42`, half-life log-normal with median = model t½, σ(ln t½) = 0.35, clipped to [1.5, 9.5] h; per-cup dose ~ Uniform(low, high).
- Graph text in English; `print` and markdown in Thai.
- Times are float hours since 00:00 of the drinking day; values ≥ 24 mean next day. Sleep times round **up** to the whole minute.
- "Run All" must reach every test cell without calling `input()`; the `main()` call is the last cell and carries cell tag `demo`.
- Existing disclaimer lines in `show_result` are kept.

## Review Focus

1. Cup times typed out of order (e.g. 17:00 then 08:00) → cups are sorted by time, result identical to ordered entry. (Task 2 test)
2. Intended bedtime earlier than the last cup (e.g. bed 01:00, last cup 17:00) → treated as next day (25.0 h), not as before the coffee. (Task 2 test)
3. Caffeine already ≤ threshold right after last cup (Decaf) → `find_sleep_time` returns `None`; output says "นอนได้ทุกเมื่อ"; graph A still spans ≥ 24 h. (Task 1 + Task 4 tests)
4. A cup typed as 00:30 → **decision: all cups are on one day, 00:00–23:59, sorted**; 00:30 is the earliest cup, and the prompt text says "วันเดียวกัน". (Task 2 test: 0.5 sorts first)
5. Number-of-cups input 0, negative, or > 10 → rejected and asked again, like other invalid input. (Task 2 test via `validate_cup_count`)

## Verification command (used by every task)

Runs every code cell except those tagged `demo`, headless:

```bash
cd /Users/park/Projects/Math2 && MPLBACKEND=Agg python3 -c "
import json
nb = json.load(open('Sleep_Planner.ipynb'))
g = {}
for i, c in enumerate(nb['cells']):
    if c['cell_type'] != 'code' or 'demo' in c.get('metadata', {}).get('tags', []):
        continue
    exec(compile(''.join(c['source']), f'cell{i}', 'exec'), g)
print('ALL CELLS OK')
"
```

Expected final line: `ALL CELLS OK`. An `AssertionError`/`NameError` = FAIL.

Edit the notebook with the NotebookEdit tool (or a JSON script). Keep `outputs: []` on new cells; clear outputs before commit.

---

### Task 1: Multi-dose model + sleep time

**Files:**
- Modify: `Sleep_Planner.ipynb` — new code cell after the existing `calc_alerts` cell (cell 4); new test cell placed in a "การทดสอบ" area at the end, before the `demo` cell.

**Interfaces:**
- Consumes: `calc_pk(weight, age, dose) -> dict` (`"t_half"`, `"ke"`).
- Produces:
  - `SLEEP_MG = 50`, `STRICT_MG = 30`
  - `caffeine_at(t: float, doses: list[tuple[float, float]], ke: float) -> float` — `doses` = `[(hour, mg), ...]`; sum of `mg·e^(−ke(t−hour))` for `hour ≤ t`.
  - `find_sleep_time(doses, ke, threshold) -> float | None` — first whole minute after the last cup where `caffeine_at ≤ threshold`; `None` if already ≤ threshold at the last cup. Closed form: `t_last + ln(A(t_last)/threshold)/ke`, then ceil to minute (`math.ceil(h*60)/60`).
  - `hours_to_label(h: float) -> str` — `"03:01 (วันถัดไป)"`, `"21:30 (วันเดียวกัน)"`, uses existing `day_label`.

- [ ] **Step 1: Write the failing test cell** (`# Unit: multi-dose model`)

```python
ke20 = calc_pk(60, 20, 95)["ke"]
exam = [(8.0, 95), (13.0, 95), (17.0, 126)]
assert abs(caffeine_at(8.0, exam, ke20) - 95) < 1e-9
assert caffeine_at(7.99, exam, ke20) == 0
assert hours_to_label(find_sleep_time(exam, ke20, SLEEP_MG)).startswith("03:01")
assert hours_to_label(find_sleep_time(exam, ke20, STRICT_MG)).startswith("06:37")
assert find_sleep_time([(9.0, 5)], ke20, SLEEP_MG) is None   # Decaf
assert caffeine_at(find_sleep_time(exam, ke20, SLEEP_MG), exam, ke20) <= SLEEP_MG
```

- [ ] **Step 2: Run verification command** — Expected: FAIL (`NameError: caffeine_at`).
- [ ] **Step 3: Implement the four items above** in the new model cell.
- [ ] **Step 4: Run verification command** — Expected: `ALL CELLS OK`. (Existing cell 7 defines `main` **and** calls `main()` — in this step split it: keep the `def main` in place, move the bare `main()` call into a new last cell with metadata `{"tags": ["demo"]}`.)
- [ ] **Step 5: Commit** — `git add Sleep_Planner.ipynb && git commit -m "feat: multi-dose caffeine model and sleep time"`

---

### Task 2: Multi-cup input

**Files:**
- Modify: `Sleep_Planner.ipynb` — replace `get_inputs` cell (cell 6).

**Interfaces:**
- Consumes: `COFFEES`, `convert_time`, `show_coffee_menu`.
- Produces:
  - `Cup = (name: str, hour: float, mg: float, low: float, high: float)` (plain tuple)
  - `validate_cup_count(n: int) -> int` — raises `ValueError` unless `1 ≤ n ≤ 10`.
  - `normalize_schedule(cups: list[Cup], bed_hour: float) -> tuple[list[Cup], float]` — sorts cups by hour; if `bed_hour <= last cup hour`, returns `bed_hour + 24`.
  - `get_inputs() -> tuple[float, int, list[Cup], float]` — `(weight, age, cups, bed_hour)`. Asks: weight, age, number of cups, then per cup: coffee number, low/high choice if range (existing prompt), time ("เวลาที่ดื่ม แก้วที่ i (00:00–23:59 วันเดียวกัน)"); finally "เวลาที่ตั้งใจจะนอน". Re-asks the whole form on `ValueError` (existing pattern).
  - `to_doses(cups) -> list[tuple[float, float]]` — `[(hour, mg)]`.

- [ ] **Step 1: Write the failing test cell** (`# Unit: input helpers`)

```python
c = lambda h, mg: ("x", h, mg, mg, mg)
cups, bed = normalize_schedule([c(17, 126), c(8, 95), c(13, 95)], 23.0)
assert [x[1] for x in cups] == [8, 13, 17] and bed == 23.0
assert normalize_schedule([c(8, 95), c(17, 126)], 1.0)[1] == 25.0
assert normalize_schedule([c(8, 95), c(0.5, 63)], 23.0)[0][0][1] == 0.5
assert to_doses(cups) == [(8, 95), (13, 95), (17, 126)]
for bad in (0, -1, 11):
    try: validate_cup_count(bad); raise AssertionError(bad)
    except ValueError: pass
assert validate_cup_count(3) == 3
```

- [ ] **Step 2: Run verification command** — Expected: FAIL (`NameError: normalize_schedule`).
- [ ] **Step 3: Implement** the helpers and new `get_inputs`; hour from `convert_time(...)` as `dt.hour + dt.minute/60`.
- [ ] **Step 4: Run verification command** — Expected: `ALL CELLS OK`.
- [ ] **Step 5: Commit** — `git commit -am "feat: multi-cup input with bedtime"`

---

### Task 3: Sequence & Series

**Files:**
- Modify: `Sleep_Planner.ipynb` — new code cell after the model cell; markdown cell above it explaining aₙ = D·(½)ⁿ, Sₙ = B(1−rⁿ)/(1−r), S∞ = B/(1−r), |r| < 1 (Thai, cite Lec7/Lab4).

**Interfaces:**
- Consumes: `caffeine_at`, `calc_pk`.
- Produces:
  - `half_life_sequence(dose: float, t_half: float, n_max: int = 6) -> list[tuple[int, float, float]]` — `(n, hours_after = n·t½, mg = dose·0.5**n)`.
  - `multi_day_series(B: float, ke: float, days: int = 7) -> dict` — keys `"r"` (= e^(−24·ke)), `"rows"` = list of `(day, loop_sum, formula_sum)` where `loop_sum` = Σ_{k=0}^{n−1} B·r^k by `for` loop and `formula_sum` = B(1−rⁿ)/(1−r), `"limit"` = B/(1−r).
  - `show_series(cups, ke, t_half, bed_hour) -> None` — prints the sequence table for the largest cup and the 7-day table with `B = caffeine_at(bed_hour, ...)`, plus the Thai one-line conclusion (≈ how many % above day 1).

- [ ] **Step 1: Write the failing test cell** (`# Unit: series`)

```python
seq = half_life_sequence(96, 4.89, 6)
assert seq[0] == (0, 0.0, 96) and abs(seq[3][2] - 12) < 1e-9
s = multi_day_series(88.0, ke20, 7)
assert abs(s["r"] - math.exp(-24 * ke20)) < 1e-12 and 0.030 < s["r"] < 0.036
assert all(abs(a - b) < 1e-9 for _, a, b in s["rows"])
assert s["rows"][0][1] == 88.0 and s["rows"][-1][1] < s["limit"]
assert abs(s["limit"] - 88.0 / (1 - s["r"])) < 1e-9
ke65 = calc_pk(60, 70, 95)["ke"]
assert multi_day_series(88.0, ke65)["r"] > s["r"]   # elderly accumulate more
```

- [ ] **Step 2: Run verification command** — Expected: FAIL.
- [ ] **Step 3: Implement** the three functions.
- [ ] **Step 4: Run verification command** — Expected: `ALL CELLS OK`.
- [ ] **Step 5: Commit** — `git commit -am "feat: sequence and series section"`

---

### Task 4: Graphs A and B

**Files:**
- Modify: `Sleep_Planner.ipynb` — new plotting cell after the series cell.

**Interfaces:**
- Consumes: `caffeine_at`, `find_sleep_time`, `SLEEP_MG`, `STRICT_MG`.
- Produces:
  - `plot_total(doses, ke, sleep50: float | None, sleep30: float | None) -> matplotlib.figure.Figure` — graph A: total line; dashed 50 mg line; faint dashed 30 mg line; vertical line + annotation `"Sleep OK HH:MM"` at `sleep50` (if `None`, annotation `"Below 50 mg from the start"`). X range: first cup − 1 h to `max(sleep50 or 0, first cup + 24) + 1` h; x tick labels as `HH:00` modulo 24.
  - `plot_stacked(doses, ke, labels: list[str]) -> Figure` — graph B: `stackplot` of each cup's remaining mg, legend with `labels`, same x range rule (without sleep marker), plus the total line.
  - Neither calls `plt.show()`; caller does.

- [ ] **Step 1: Write the failing test cell** (`# Unit: plots`)

```python
import matplotlib
f = plot_total(exam, ke20, find_sleep_time(exam, ke20, SLEEP_MG), find_sleep_time(exam, ke20, STRICT_MG))
ax = f.axes[0]
assert any("03:01" in t.get_text() for t in ax.texts)
lo, hi = ax.get_xlim(); assert hi - lo >= 24
f2 = plot_total([(9.0, 5)], ke20, None, None)
assert f2.axes[0].get_xlim()[1] - f2.axes[0].get_xlim()[0] >= 24
f3 = plot_stacked(exam, ke20, ["Drip 08:00", "Drip 13:00", "Americano 17:00"])
assert len(f3.axes[0].collections) >= 3
matplotlib.pyplot.close("all")
```

- [ ] **Step 2: Run verification command** — Expected: FAIL.
- [ ] **Step 3: Implement** both functions (style close to `.superpowers/brainstorm/*/content/graph-style.html` A and B: orange total line, green threshold).
- [ ] **Step 4: Run verification command** — Expected: `ALL CELLS OK`.
- [ ] **Step 5: Commit** — `git commit -am "feat: caffeine graphs A and B"`

---

### Task 5: Probability (Monte Carlo)

**Files:**
- Modify: `Sleep_Planner.ipynb` — new code cell + markdown cell above (Thai: empirical probability = successes/N, why half-life dominates, NCBI 1.5–9.5 h, cite Lec6).

**Interfaces:**
- Consumes: `cups` (`Cup` tuples with `low`, `high`), `t_half` from `calc_pk`, `find_sleep_time`.
- Produces:
  - `monte_carlo_sleep(cups, t_half: float, bed_hour: float, threshold: float = SLEEP_MG, n: int = 10_000, seed: int = 42, sigma: float = 0.35) -> dict` — keys `"prob"` (float 0–1: share of runs with caffeine at `bed_hour` ≤ threshold), `"sleep_hours"` (np.ndarray, length n; runs that are already below → last cup hour), `"t_halves"` (np.ndarray). Use `np.random.default_rng(seed)`; vectorise over runs with numpy.
  - `plot_sleep_histogram(result, bed_hour) -> Figure` — histogram of `sleep_hours` with a vertical line at `bed_hour`, title shows `P = xx.x%`.

- [ ] **Step 1: Write the failing test cell** (`# Unit: probability`)

```python
exam_cups = [("Drip", 8.0, 95, 95, 96), ("Drip", 13.0, 95, 95, 96), ("Americano 2", 17.0, 126, 126, 126)]
t20 = calc_pk(60, 20, 95)["t_half"]; t70 = calc_pk(60, 70, 95)["t_half"]
r1 = monte_carlo_sleep(exam_cups, t20, 23.0); r2 = monte_carlo_sleep(exam_cups, t20, 23.0)
assert r1["prob"] == r2["prob"] and 0 <= r1["prob"] <= 1
assert r1["t_halves"].min() >= 1.5 and r1["t_halves"].max() <= 9.5
assert abs(np.median(r1["t_halves"]) - t20) < 0.2
assert monte_carlo_sleep(exam_cups, t70, 23.0)["prob"] < r1["prob"]
assert monte_carlo_sleep(exam_cups, t20, 27.5)["prob"] > r1["prob"]
assert len(r1["sleep_hours"]) == 10_000
```

- [ ] **Step 2: Run verification command** — Expected: FAIL.
- [ ] **Step 3: Implement** both functions.
- [ ] **Step 4: Run verification command** — Expected: `ALL CELLS OK`.
- [ ] **Step 5: Commit** — `git commit -am "feat: monte carlo sleep probability"`

---

### Task 6: Wire up output, explanation, 3 test cases

**Files:**
- Modify: `Sleep_Planner.ipynb` — `show_result`, `main`, markdown cells, test-case cells, final `demo` cell.
- Modify: `.gitignore` — add `.superpowers/`.

**Interfaces:**
- Consumes: everything above.
- Produces:
  - `run_plan(weight, age, cups, bed_hour, show_plots: bool = True) -> dict` — computes everything with no `input()`; returns `{"pk", "sleep50", "sleep30", "series", "mc"}`; prints in the spec order: cups table → PK values → sleep time 50 mg (or "คาเฟอีนต่ำกว่าเกณฑ์ตั้งแต่ดื่ม — นอนได้ทุกเมื่อ") and 30 mg → graph A → graph B → series tables → probability % + histogram → existing disclaimers. `pk` from `calc_pk(weight, age, total mg)`.
  - `main()` = `run_plan(*get_inputs())`.

- [ ] **Step 1: Write the 3 test-case cells** (markdown header "ทดสอบ 3 กรณี" above each, Thai one-line description):

```python
# กรณีที่ 1: วันปกติ — Drip 1 แก้ว 08:00, นอน 23:00
r = run_plan(60, 20, [("Drip", 8.0, 95, 95, 96)], 23.0)
assert r["sleep50"] is not None and r["sleep50"] < 23.0 and r["mc"]["prob"] > 0.9

# กรณีที่ 2: ช่วงสอบ — 3 แก้ว
r = run_plan(60, 20, exam_cups, 23.0)
assert hours_to_label(r["sleep50"]).startswith("03:01")
assert hours_to_label(r["sleep30"]).startswith("06:37")

# กรณีที่ 3: อายุ 70 ดื่ม Cold Brew 200 mg 10:00 vs อายุ 20
cb = [("Cold Brew", 10.0, 200, 100, 200)]
old, young = run_plan(60, 70, cb, 23.0), run_plan(60, 20, cb, 23.0)
assert old["sleep50"] > young["sleep50"] and old["mc"]["prob"] < young["mc"]["prob"]
```

  Fill case 1's exact expected sleep time into its assert after first run (print it, then pin with `startswith`).
- [ ] **Step 2: Run verification command** — Expected: FAIL (`NameError: run_plan`).
- [ ] **Step 3: Implement** `run_plan`, rewrite `show_result` to the new output order, `main()`; add markdown cells: title + problem (Thai, the student exam scenario + Drake 2013), "แบบจำลองคณิตศาสตร์" (PK formulas + A(t) sum), threshold rationale (Gardiner 2023/2024), references list from spec §7. Order of cells: title/problem md → imports → data → model → series → probability → plots → I/O → `run_plan`/`main` → unit-test cells → 3 test-case cells → `demo` cell `main()`.
- [ ] **Step 4: Run verification command** — Expected: `ALL CELLS OK`. Then open in Jupyter (or ask the user to) and run the `demo` cell once by hand with the exam inputs to confirm prompts read well.
- [ ] **Step 5: Clear outputs and commit** — `git add Sleep_Planner.ipynb .gitignore && git commit -m "feat: full sleep planner flow with tests"`
