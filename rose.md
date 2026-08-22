# ROSE — Project Context & Webpage Refactor Brief (for Claude Code)

> Paste this whole file into Claude Code, or drop it in the repo and tell Claude Code
> to read it first. It contains everything needed to understand the ROSE research
> project and to rebuild / refactor the existing results webpage. Sections 1–4 are the
> "why" (research framing). Section 5 is the exact data to reproduce. Sections 6–7 are
> the existing design system & stack. Section 8 is where YOU write what you want changed.

---

## 0. TL;DR of the task

There is an existing single-file HTML results page for a research project called **ROSE**
(`Chart.js` + vanilla JS, ~800px max-width, light academic style). The goal is to
**refactor / rebuild** this page. Preserve all the data and the core narrative; improve
structure, maintainability, and presentation per the goals in Section 8.

---

## 1. What ROSE is

**ROSE = "Roadside Supervision Enables Learning at the Edge of Reasoning Capabilities."**

It is a **cold-start / warm-up SFT method for RLVR** (reinforcement learning with
verifiable rewards). The motivation: on hard problems, a small student model rarely or
never samples a correct rollout within budget, so on-policy RL gets zero reward → zero
advantage → no gradient → training stalls. Meanwhile naive full SFT on clean teacher
traces forces the student to imitate tokens far outside its own distribution, which
distorts its reasoning style and transfers poorly on small models.

ROSE sits **between** off-policy SFT and on-policy distillation. It frames reasoning
rollouts as a search process and applies **"roadside supervision":**

1. The **student** generates an on-policy **prefix** (its own natural reasoning trace).
2. The trajectory is **cut at a stitch point** before the student drifts.
3. The **teacher** (Qwen3-4B-Thinking-2507) **continues** from the student prefix to a
   correct answer (off-policy suffix).
4. **SFT loss is masked**: computed **only on the teacher continuation (suffix)**. The
   student prefix is preserved, never imitated.

This "stitched / prefixed" supervision keeps the student near its own on-policy
distribution while still injecting a correct completion.

## 2. Core claims / framing (use these as the page's narrative spine)

- **Smooth knowledge transfer**: ROSE bridges the gap between student and teacher
  distributions instead of forcing imitation, so transfer is smoother.
- **Small Model Learnability Gap**: stitched / noisy supervision *outperforms* clean
  oracle SFT on small models (Qwen3-1.7B) but the advantage shrinks / disappears on
  larger models — a **scale-dependent interaction effect** between data quality and
  model size.
- **Robustness to noisy supervision**: ROSE uses *unfiltered* data (mixed
  correct/incorrect) yet beats methods that get the same noisy data, and even beats
  them at 2× the data budget.
- **Stronger initialization for online learning**: a ROSE warm-up gives a better cold
  start for subsequent online GRPO and on-policy distillation (OPD) than a
  Verified-SFT warm-up.
- **Distributional evidence (NLL)**: after ROSE, the student's outputs sit closer to
  both the student's own policy (low NLL on the training set) and the teacher's policy
  (low NLL on AIME25 under the teacher).

## 3. Baselines / comparison methods

- **Plain Model** — base student, no training.
- **Full SFT / Verified SFT** — SFT on full teacher trajectories that are verified
  correct (loss on every token).
- **Teacher Prefix Supervision (TPS)** — teacher provides the prefix instead of the
  student (the inverse of ROSE).
- **Mixed** — SFT on unfiltered (noisy) student/teacher data, no stitching.
- **POPE** — a baseline appearing only at the RLVR step in the Polaris experiment.
- **OPD** — on-policy distillation (used both as an online-learning target in Fig. 3
  and as an eval checkpoint in the Polaris experiment).
- (Other baselines discussed in the paper but not all plotted here: LUFFY,
  CRITIQUE-CODER.)

## 4. Experimental scope

- **Models**: student Qwen3-1.7B (logic/RLVE experiments) and Qwen3-4B (Polaris MATH);
  teacher Qwen3-4B-Thinking-2507.
- **Domains / datasets**: SynLogic-style puzzles (**Kukurasu, Minesweeper**, Sudoku),
  **RLVE** multi-task environments (10 and 20 envs), and **Polaris MATH**.
- **Eval**: Avg@K and Pass@K; AIME 2025 and in-distribution held-out sets;
  also LiveCodeBench in the broader project.
- **Status**: ongoing work. Primary submission target **ARR August 2026** (EACL cycle);
  near-term a **CoLM workshop** for early feedback.

---

## 5. EXACT DATA TO REPRODUCE

> Reproduce every chart with these numbers. Units are percentages unless noted.
> "Higher is better" everywhere except the two NLL charts ("lower is better").

### 5.1 Method visualization (interactive, top of page)

An interactive animated component with 3 tabs comparing how each method handles one
hard problem ("Solve a hard 4×4 Minesweeper instance — base model never samples it
correctly"). Play / Reset controls. Tokens stream in; loss-masked tokens get a highlight.

- **ROSE (ours)** — 5 steps: (1) student streams its on-policy prefix; (2) note prefix
  is natural to the student; (3) ✂ cut at stitch point before drift; (4) teacher
  continues from the prefix → correct; (5) loss masked on prefix, applied only on
  teacher continuation. Key idea: *student keeps its on-policy prefix; teacher only
  continues; SFT loss only on the teacher continuation — prefix preserved, never imitated.*
- **Verified SFT** — 3 steps: teacher generates full trace → verified correct → full
  trace is the SFT target, loss on every token. Key idea: *student must imitate every
  token incl. ones outside its distribution; distorts reasoning style on hard problems.*
- **On-policy RL** — 3 steps: student samples rollouts → all k=128 rollouts wrong →
  reward 0 → advantage 0 → no gradient, training stalls.

Color semantics: student tokens = orange (`#FAECE7` bg / `#993C1D` text / `#D85A30`
border); teacher tokens = purple (`#EEEDFE` bg / `#3C3489` text / `#534AB7` border);
loss-applied = gold ring `#BA7517`.

### 5.2 Figure 1 — "ROSE consistently outperforms full off-policy SFT"

Grouped bar charts (Avg@8 and Pass@8). Categories: Kukurasu, Minesweeper, RLVE (10 tasks).
Series: Original, Verified SFT, ROSE.

**Avg@8 (%)** — y-max 20

| Task            | Original | Verified SFT | ROSE  |
| --------------- | -------- | ------------ | ----- |
| Kukurasu        | 0.10     | 6.88         | 11.98 |
| Minesweeper     | 1.58     | 6.83         | 16.08 |
| RLVE (10 tasks) | 5.56     | 5.56         | 10.42 |

**Pass@8 (%)** — y-max 70

| Task            | Original | Verified SFT | ROSE  |
| --------------- | -------- | ------------ | ----- |
| Kukurasu        | 0.83     | 25.00        | 30.00 |
| Minesweeper     | 10.00    | 39.33        | 62.00 |
| RLVE (10 tasks) | 19.44    | 27.78        | 41.67 |

Footnote: single-task (Kukurasu, Minesweeper) = 5K examples each; multi-task (RLVE) =
2.5K total across 10 tasks (250/task). ROSE data is unfiltered (mixed correct/incorrect);
Verified SFT uses only verified-correct trajectories.

### 5.3 Figure 2 — "ROSE is robust to noisy supervision"

RLVE multi-task, 20 envs. All three methods use unfiltered (noisy) data of matched size.
Categories: 16K data, 8K data. Series: Mixed, Teacher Prefix, ROSE.

**Avg@8 (%)** — y-max 12

| Data | Mixed | Teacher Prefix | ROSE |
| ---- | ----- | -------------- | ---- |
| 16K  | 5.10  | 5.43           | 9.54 |
| 8K   | 4.61  | 6.25           | 8.39 |

**Pass@8 (%)** — y-max 35

| Data | Mixed | Teacher Prefix | ROSE  |
| ---- | ----- | -------------- | ----- |
| 16K  | 21.05 | 21.05          | 31.58 |
| 8K   | 14.47 | 23.68          | 26.32 |

Footnote: Student Qwen3-1.7B, Teacher Qwen3-4B-Thinking-2507, RLVE 20 envs. ROSE at 8K
already exceeds Mixed and Teacher Prefix at 16K on both metrics.

### 5.4 Figure 3 — "ROSE provides a stronger initialization for online learning"

3-epoch offline SFT warm-up on 5K examples, then online learning (GRPO or OPD).
Four grouped bar charts. Categories: Kukurasu, Minesweeper. Series: Original (no warm-up),
Verified SFT warm-up, ROSE warm-up.

**Online GRPO · Avg@8 (%)** — y-max 25

| Task        | Original | Verified SFT | ROSE  |
| ----------- | -------- | ------------ | ----- |
| Kukurasu    | 0.00     | 6.56         | 13.75 |
| Minesweeper | 1.42     | 10.33        | 20.67 |

**Online OPD · Avg@8 (%)** — y-max 25

| Task        | Original | Verified SFT | ROSE  |
| ----------- | -------- | ------------ | ----- |
| Kukurasu    | 0.21     | 8.75         | 17.60 |
| Minesweeper | 1.83     | 7.42         | 19.33 |

**Online GRPO · Pass@8 (%)** — y-max 80

| Task        | Original | Verified SFT | ROSE  |
| ----------- | -------- | ------------ | ----- |
| Kukurasu    | 0.00     | 27.50        | 41.67 |
| Minesweeper | 8.00     | 56.67        | 72.00 |

**Online OPD · Pass@8 (%)** — y-max 80

| Task        | Original | Verified SFT | ROSE  |
| ----------- | -------- | ------------ | ----- |
| Kukurasu    | 0.83     | 30.83        | 41.67 |
| Minesweeper | 11.33    | 39.33        | 61.33 |

### 5.5 Polaris MATH — Qwen3-4B

**Setup table** (render as a clean key/value block):

- Student model: Qwen3-4B
- Teacher model: Qwen3-4B-Thinking-2507
- Data source: POLARIS-Project/Polaris-Dataset-53K, filtered to problems where Pass@8 = 0
  (difficulty estimated via Deepseek-R1-Distill-Qwen-7B) → ~15K hard problems.
  Link: https://huggingface.co/datasets/POLARIS-Project/Polaris-Dataset-53K
- SFT data: 40K trajectories on the 15K problems (used for Full SFT, Teacher Prefix
  Supervision, and ROSE).
- RL/OPD data: same 15K problems for RLVR and OPD training.
- Evaluation: Pass@K and Avg@K on AIME 2025 (30 problems) and an in-distribution
  held-out set (100 problems), at Offline / RLVR Step 40 / OPD Step 40 checkpoints.

Series for all four charts: Plain Model, Full SFT, Teacher Prefix Sup., POPE, ROSE 4096.
X-axis categories: Offline, RLVR Step 40, OPD Step 40. `null` = bar not shown (N/A).

**AIME 25 — Pass@64 (%)** — y-max 90

| Method              | Offline | RLVR 40 | OPD 40 |
| ------------------- | ------- | ------- | ------ |
| Plain Model         | 73.33   | 73.33   | 70.00  |
| Full SFT            | 60.00   | 60.00   | 60.00  |
| Teacher Prefix Sup. | 63.33   | 63.33   | 60.00  |
| POPE                | null    | 66.67   | null   |
| ROSE 4096           | 76.67   | 80.00   | 76.67  |

**AIME 25 — Avg@64 (%)** — y-max 65

| Method              | Offline | RLVR 40 | OPD 40 |
| ------------------- | ------- | ------- | ------ |
| Plain Model         | 46.41   | 46.56   | 46.98  |
| Full SFT            | 32.66   | 34.32   | 32.34  |
| Teacher Prefix Sup. | 37.86   | 37.66   | 35.73  |
| POPE                | null    | 47.19   | null   |
| ROSE 4096           | 51.98   | 52.14   | 52.71  |

**In Distribution — Pass@8 (%)** — y-max 80

| Method              | Offline | RLVR 40 | OPD 40 |
| ------------------- | ------- | ------- | ------ |
| Plain Model         | 63.00   | 58.00   | 64.00  |
| Full SFT            | 57.00   | 54.00   | 49.00  |
| Teacher Prefix Sup. | 60.00   | 58.00   | 49.00  |
| POPE                | null    | 60.00   | null   |
| ROSE 4096           | 66.00   | 69.00   | 67.00  |

**In Distribution — Avg@8 (%)** — y-max 55

| Method              | Offline | RLVR 40 | OPD 40 |
| ------------------- | ------- | ------- | ------ |
| Plain Model         | 38.25   | 39.75   | 39.37  |
| Full SFT            | 32.37   | 32.62   | 30.88  |
| Teacher Prefix Sup. | 34.00   | 32.75   | 30.25  |
| POPE                | null    | 37.13   | null   |
| ROSE 4096           | 41.63   | 44.62   | 43.00  |

### 5.6 NLL analysis (horizontal bars, lower is better)

**Analysis 1 — "Full SFT is far from the student policy"**
Avg per-token NLL on the Polaris training set (x-max 0.7):

- Full SFT: 0.534
- Teacher Prefix 4096: 0.57
- ROSE 4096: 0.367  ← lowest

**Analysis 2 — "After ROSE, student outputs are closer to teacher"**
Avg per-token NLL on AIME25 under the teacher policy (x-max 0.5):

- Plain Model: 0.374
- Full SFT: 0.315
- Teacher Prefix 4096: 0.331
- ROSE 4096: 0.222  ← lowest

### 5.7 "Next steps" block

- **Experiment 1 — General reasoning domain**: apply ROSE to physics & chemistry using
  TIGER-Lab/WebInstruct-verified
  (https://huggingface.co/datasets/TIGER-Lab/WebInstruct-verified); tests whether the
  on-policy-prefix + teacher-suffix approach transfers beyond math/logic.
- **Experiment 2 — Scaling up logic domain**: scale RLVE to larger task counts / data
  budgets for stronger scaling evidence.
- **Timeline**: primary target ARR August 2026 (EACL cycle); near-term a CoLM workshop
  for early feedback.
- **Blocker**: DeltaAI partition (ghx4) out of credits; need a GH200 allocation to run
  Exps 1 & 2 before the August deadline.

---

## 6. Existing design system (keep unless Section 8 says otherwise)

- **Palette**:
  - ROSE / primary purple `#534AB7`; light purple surfaces `#EEF0FB`, `#EEEDFE`.
  - Student / orange accent `#D85A30` (bg `#FAECE7`, text `#993C1D`).
  - Teacher / purple accent `#534AB7` (bg `#EEEDFE`, text `#3C3489`).
  - Loss highlight gold `#BA7517`; insight box bg `#FAEEDA`, text `#412402`.
  - Chart series: Original/Plain `#B4B2A9`, Verified/Full SFT `#85B7EB`,
    Teacher Prefix `#5DCAA5`, POPE `#EF9F27`, ROSE `#534AB7`.
  - Neutral surfaces `#F7F6F2` / `#F0F0EE`; hairline borders `#D3D1C7` / `#e8e8e8`;
    body text `#333`, muted `#666` / `#888780`.
- **Type**: system sans stack (`-apple-system, BlinkMacSystemFont, "Segoe UI",
  "Helvetica Neue", Arial, sans-serif`); monospace for the animated tokens.
- **Layout**: centered, `max-width: 800px`, generous vertical rhythm, section dividers,
  2-col chart grids that collapse to 1-col below ~520px.
- **Components**: an "Ongoing Work" tag pill; figure label / title / subtitle headers;
  legend rows with color swatches; off-white footnote callouts with a left accent border.
- **Icons**: Tabler icons web font.

## 7. Existing tech stack

- Single self-contained `.html` file: inline `<style>` + inline `<script>`.
- Chart.js 4.4.1 (UMD via cdnjs) for all bar charts.
- Tabler Icons web font (jsDelivr).
- Vanilla JS for the interactive method animation (no framework).
- No build step, no external data files — all numbers are hard-coded in the script.

---

## 8. >>> REFACTOR GOALS — FILL THIS IN <<<

Describe what you actually want changed. Examples you might pick from / edit:

- [ ] Split the monolithic HTML into modular files (e.g. separate CSS / JS, or a
      small component structure) while keeping it deployable as a static page.
- [ ] Port to a framework (React/Vue/Svelte/Astro?) — specify which, and whether
      Chart.js stays or switches (Recharts/D3/Plotly?).
- [ ] Make it fully responsive / mobile-first.
- [ ] Add an abstract / intro section, author list, paper/code links, BibTeX.
- [ ] Restructure the narrative order of figures, or add a results summary table.
- [ ] Light/dark mode, accessibility pass (ARIA, contrast), print/PDF styling.
- [ ] Data-driven refactor: move all numbers into a single config object/JSON so
      charts are generated from data rather than duplicated code.
- [ ] Anything else: __________________________________________________

Constraints / preferences (fill in): target venue's page style? deploy via GitHub
Pages? must stay single-file? keep exact current colors? ________________________