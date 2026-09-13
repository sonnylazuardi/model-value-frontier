# Handover — LLM performance-vs-price chart (for an external AI agent)

**Repo:** https://github.com/sonnylazuardi/model-value-frontier
**Live URL:** https://vps.sonnylab.com/model-value-2026-07.html
**Last updated:** 13 September 2026
**File in this repo:** `model-value-2026-07.html` (do NOT rename — the live URL depends on it)

You are an AI agent on a **different machine** with **no access** to the serving box.
Your job: edit the chart, commit, push. A болезнен process on the server pulls
and deploys. You never deploy — you only push.

---

## 1. What this is

A one-page, dependency-free HTML report comparing frontier LLMs on **capability
vs price**. Open the file in a browser and it renders. Contents:

1. **Scatter** — blended $/1M tokens (log x-axis) vs Intelligence Index (y-axis).
   Value-frontier models in blue, dominated models in grey, frontier as a line.
2. **Bar chart** — raw Intelligence Index, sorted desc.
3. **Bar chart** — value (Index points per dollar), sorted desc.
4. **Table** — every figure, plus unplottable models as explicit blank rows.
5. **"What to trust here"** — caveats, corrections, why certain models are excluded.

Light/dark themes, hover tooltips, keyboard focus, table view. No build step, no
JS framework. All rendering is vanilla JS + SVG in the single `<script>` block.

---

## 2. Methodology (do not break this)

### Anchor metric
Everything is scored on the **Artificial Analysis (AA) Intelligence Index** —
the *only* capability score run identically across every vendor. Never mix vendor
benchmarks (SWE-bench, AIME, LiveCodeBench…): labs report them inconsistently and
third-party reproductions contradict each other. Mixing produces a fabricated ranking.

> The Index itself gets revised by AA (v4.1.1 → v4.2 → v4.3 within weeks in
> Aug–Sept 2026, each rebasing absolute scores). The chart always tracks **one**
> Index version throughout — check the Sources line for the current one, and
> never mix versions within the chart.

### Price axis
**AA blended price** from each model's AA page:

```
blended = 0.70 x cache_read + 0.20 x input + 0.10 x output
```

Bias: output is only 10% of the weight, cache reads 70% — the ranking assumes a
**cache-heavy, short-output** workload. Reasoning models burn output tokens on
chain-of-thought, so real cost-per-task runs above the blended figure. Raw
input/output columns stay in the table so readers can re-rank.

### Value frontier (computed in code, not by hand)
A model is **on the frontier** if nothing cheaper scores higher: sort by price
ascending, keep a running max of Index. Tie-breaks (both load-bearing):

- Equal price → higher Index wins (`a.blend - b.blend || b.idx - a.idx`).
- Equal Index → cheaper model takes the slot.

### Effort tiers
AA scores per reasoning-effort tier (max / high / xhigh). This chart uses
**whatever tier AA's canonical model page defaults to** — imperfect across
models, documented as a limitation in the notes.

### Plot rule
A model goes on the chart only with **both** an AA Index **and** a real
per-token price. Otherwise it goes in the table as an explicit blank row.
Never invent numbers. In particular:

- `$0` / free preview pricing → unplottable (no log-axis position, infinite pts/$).
- Subscription-only pricing → not a per-token price.
- No AA entry (stealth models, scoring lag ~3–5 days after release) → wait, keep blank.

---

## 3. How to add or update a model

1. **Get numbers from AA's own page**, not blogs:
   `https://artificialanalysis.ai/models/<slug>` — need Index, blended price,
   input, output, context, release date. Slugs are inconsistent; if `/models/x`
   404s, try `/providers/<vendor>`. If the page is a JS shell with no data, the
   model isn't scored yet — keep it pending.
2. **Edit the `DATA` array** in the HTML. One object per model:
   ```js
   { name:"Grok 4.6", maker:"xAI", idx:51, blend:1.35, inp:2, out:6, elo:null, ctx:"500K", dx:0, dy:-15, anc:"middle" },
   ```
   - `dx`/`dy`/`anc` control the scatter label: `"middle"` = above (`dy:-15`),
     `"start"` = right (`dx:12, dy:5`), `"end"` = left (`dx:-12, dy:5`).
     Stagger successors sharing a price so labels don't collide.
   - Keep the array roughly sorted by Index desc for readability (code sorts
     independently **except** for stable-sort ties — order matters there).
   - `elo:null` renders as an em dash (Arena Elo is sparse — only set it when known).
3. **Check axis domains still fit** (scatter IIFE):
   - `x0`/`x1` are `Math.log10(...)` bounds — widen if a blend falls outside,
     and add a matching tick to the tick array.
   - `y0`/`y1` likewise — widen if an Index exceeds them or a label clips, and
     extend the y-tick loop bound together with `y1`.
4. **Update prose that references counts or claims**: the header count
   ("Thirty-one frontier models"), the "spans N points / top five within N"
   caption, and any note bullet asserting who dominates whom. **These go stale
   constantly** — a new model routinely invalidates an existing bullet.
5. **Same-price successors**: when a vendor ships a `.x` release at identical
   rates with a higher Index (GLM-5.3, Muse Spark 1.2/1.3, DeepSeek V4 Pro 0813,
   V4.1 Flash, Gemini 3.8 Flash…), the new build *becomes* the charted row and
   the old one is retired/dominated. Check for this on every point release.
6. **Verify** (section 4), then **commit + push** (section 5).

### Gotchas (learned the hard way)

- **Don't trust pricing blogs.** Several listed GPT-5.6 Luna at $1/$6; OpenAI's
  docs say $0.20/$1.20 — a 5x error. Always vendor docs or AA.
- **DeepSeek's price table is column-ordered cache-hit first.** Reading column 1
  as input understates cost ~120x. Vendors also hike/cut rates without warning
  (DeepSeek ~3x hike Sept 2026, Sol cut, Gemini 3.6 cut) — re-verify prices on
  every touch, don't carry old ones forward.
- **"Max"/"high"/"xhigh" are reasoning-effort settings, not price tiers.**
- **Vendor blog pages (`z.ai/blog/...`, `qwen.ai/blog?id=...`) are client-side
  SPAs** — `curl` returns a JS shell. New models stay pending until AA scores
  them; link the blog in Sources only as provenance, never as numbers.
- **AA changelog entries can be stale.** Changelog snapshots sometimes show a
  previous Index version's score while the live model page already moved on.
  The live per-model page is always canonical — never take a changelog number
  without opening the page.

---

## 4. Verify (no browser needed — verify programmatically)

### Syntax check
```bash
cd model-value-frontier && python3 -c "
import re
s=open('model-value-2026-07.html').read()
open('/tmp/chart-check.js','w').write(re.search(r'<script>(.*?)</script>',s,re.S).group(1))
" && node --check /tmp/chart-check.js && echo "JS OK"
```

### Recompute the frontier (always regenerate — never reuse a cached script)
Run from the repo root (`model-value-frontier/`):
```bash
python3 << 'PYEOF'
import re
s = open('model-value-2026-07.html').read()
m = re.search(r'(const DATA = \[.*?\n\];)', s, re.S)
f = re.search(r'(// value frontier.*?d\.front = onF\.has\(d\.name\)\);)', s, re.S)
tail = """
console.log("FRONTIER:");
frontier.forEach(d => console.log("   " + d.name.padEnd(22) + " idx " + d.idx + "  $" + d.blend));
console.log("n=" + DATA.length);
[...DATA].sort((a,b)=>b.idx-a.idx||a.blend-b.blend).forEach((d,i) =>
  console.log(String(i+1).padStart(3) + "  " + d.name.padEnd(22) + " " + String(d.idx).padStart(2) + "  $" + d.blend.toFixed(2).padStart(5) + "   ppd " + d.ppd.toFixed(1) + (d.front ? "   <-- frontier" : "")));
"""
open('/tmp/fr.js','w').write(m.group(1) + "\nDATA.forEach(d => d.ppd = d.idx / d.blend);\n" + f.group(1) + tail)
PYEOF
node /tmp/fr.js
```
A stale `fr.js` once printed pre-edit numbers and faked a verification. Always regenerate.
Then confirm the frontier bullet in the prose matches the recomputed member list
and count exactly.

---

## 5. Git workflow (the sync contract — follow it exactly)

**You (external agent):**

```bash
git pull --rebase          # before every edit session
# ... edit model-value-2026-07.html ...
# ... run both verifications in section 4 ...
git add model-value-2026-07.html
git commit -m "<what changed, e.g. 'plot Agnes 3.0 Flash (36-est, $0.03); frontier 6->7'>"
git push
```

- One logical change per commit (one model add, one rebase, one price fix).
- Commit message states the frontier effect (`frontier unchanged` / `6->7`).
- Push to `main`. Never force-push.
- If `git pull --rebase` reports a conflict in the HTML, **stop and ask the
  human** — do not resolve DATA-array conflicts by hand-merging numbers; one
  side's figures are usually stale.

**The serving box (not you):** pulls `main`, copies the HTML to its webroot,
and serves it at the live URL. Deploys are pull-based — nothing you do needs
to touch the server. To confirm your push is live, fetch the live URL and
`diff` it against your committed file; they must be byte-identical.

**Concurrency rule (learned 11–12 Sept 2026):** two agents editing the HTML at
once will clobber each other — parallel same-file edits silently drop one
side's changes, and even sequential full-block rewrites race. Protocol:

- `git pull --rebase` immediately before editing, push promptly after verifying.
- Keep edits small and keep this file's history linear.
- If a push is rejected (non-fast-forward), pull first and re-verify the
  frontier before pushing again — the other side may have moved numbers
  your prose already describes.

---

## 6. Current dataset (31 plotted, Index v4.3, as of 12 Sept 2026)

| Model | Maker | Index | Blended $/1M | In | Out | Pts/$ | Frontier |
|---|---|---|---|---|---|---|---|
| Claude Fable 5.1 | Anthropic | 53 | 7.17 | 10.00 | 50.00 | 7.4 | YES |
| GPT-6 Astra | OpenAI | 53 | 7.70 | 10.00 | 50.00 | 6.9 | |
| Claude Opus 5 | Anthropic | 51 | 3.85 | 5.00 | 25.00 | 13.2 | YES |
| Claude Fable 5 | Anthropic | 50 | 7.70 | 10.00 | 50.00 | 6.5 | |
| GPT-5.6 Sol | OpenAI | 47 | 3.08 | 4.00 | 20.00 | 15.3 | YES |
| Muse Spark 1.3 | Meta | 45 | 0.78 | 1.25 | 4.25 | 57.7 | YES |
| GLM-5.3 | Z.ai | 45 | 0.90 | 1.40 | 4.40 | 50.0 | |
| Grok 4.6 | xAI | 44 | 1.35 | 2.00 | 6.00 | 32.6 | |
| Kimi K3 | Moonshot | 44 | 2.31 | 3.00 | 15.00 | 19.0 | |
| GLM-5.3 Flash | Z.ai | 42 | 0.10 | 0.15 | 0.50 | 420.0 | YES |
| GPT-5.6 Terra | OpenAI | 42 | 1.74 | 2.00 | 12.00 | 24.1 | |
| Gemini 3.8 Flash | Google | 41 | 0.58 | 0.75 | 3.75 | 70.7 | |
| Qwen 3.8 Flash Next | Alibaba | 40 | 0.09 | 0.15 | 0.47 | 444.4 | YES |
| DeepSeek V4.1 Flash | DeepSeek | 40 | 0.18 | 0.30 | 1.20 | 222.2 | |
| Muse Spark 1.2 | Meta | 40 | 0.78 | 1.25 | 4.25 | 51.3 | |
| Qwen 3.8 Max | Alibaba | 40 | 1.18 | 2.00 | 6.00 | 33.9 | |
| Gemini 3.7 Flash | Google | 39 | 0.58 | 0.75 | 3.75 | 67.2 | |
| Grok 4.5 | xAI | 39 | 1.21 | 2.00 | 6.00 | 32.2 | |
| GPT-5.6 Luna | OpenAI | 38 | 0.17 | 0.20 | 1.20 | 223.5 | |
| Claude Sonnet 5 | Anthropic | 38 | 1.54 | 2.00 | 10.00 | 24.7 | |
| Agnes 3.0 Flash | Sapiens AI | 36 | 0.03 | 0.05 | 0.15 | 1200.0 | YES |
| DeepSeek V4 Pro 0813 | DeepSeek | 36 | 0.69 | 1.32 | 3.96 | 52.2 | |
| Agnes 2.5 Pro Beta | Sapiens AI | 35 | 0.06 | 0.10 | 0.30 | 583.3 | YES* |
| DeepSeek V4 Flash Vision | DeepSeek | 35 | 0.23 | 0.44 | 1.32 | 152.2 | |
| Qwen 3.8 27B (xhigh) | Alibaba | 34 | 0.43 | 0.50 | 3.00 | 79.1 | |
| Gemini 3.6 Flash | Google | 34 | 0.63 | 0.75 | 3.75 | 54.0 | |
| Muse Spark 1.1 | Meta | 34 | 0.78 | 1.25 | 4.25 | 55.1 | |
| GLM-5.2 | Z.ai | 34 | 0.90 | 1.40 | 4.40 | 37.8 | |
| MiniMax M3 | MiniMax | 30 | 0.22 | 0.30 | 1.20 | 136.4 | |
| Hunyuan 3 (Hy3) | Tencent | 26 | 0.11 | 0.14 | 0.55 | 300.0 | |
| MiMo-V2.5-Pro | Xiaomi | 26 | 0.18 | 0.435 | 0.87 | 144.4 | |

*estimate tag survives only on the Agnes pair.

**Frontier (7, v4.3):** Agnes 3.0 Flash ($0.03) → Qwen 3.8 Flash Next ($0.09) →
GLM-5.3 Flash ($0.10) → Muse Spark 1.3 ($0.78) → GPT-5.6 Sol ($3.08) →
Claude Opus 5 ($3.85) → Claude Fable 5.1 ($7.17).

### Deliberately unplottable (blank table rows — do not invent numbers)

| Model | Blocker |
|---|---|
| **Ox Alpha** (stealth, 20 Aug) | No AA Index; price **$0** (no log position, infinite pts/$). |
| **Ornith 1.5** | No AA entry. |
| **GLM-5.2 Turbo** | No AA entry; absent from Z.ai's pricing table. |
| **Claude Mythos 5** | AA 404 (limited availability, never scored). |
| **Granite 4.2 30B / 8B / 3B** | AA-scored but far below the chart's floor. Do not plot. |
| **MiniCPM5-2B** (7 Sept) | AA Index ~13 (estimate) **and** price **$0** — double-blocked. |
| **K2 Horizon 375B** (MBZUAI) | Has an AA Index but no per-token price found — needs pricing research. |
| **Ling-3.0-flash-VL** | Index 25, below the chart's floor. Do not plot. |

### Calendar watches (check on/after these dates)

- ~~14 Sept 2026 12:00 Beijing~~ — `deepseek-v4-pro` routes to V4.1 Flash
  (verify whether the charted Pro 0813 row needs succession; check AA).
- Claude Sonnet 5's planned 1 Sept rise ($2/$10 → $3/$15) never appeared on AA
  — update only when AA's page moves.

---

## 7. Design constraints (keep these)

- Palette is validated, not eyeballed: frontier blue (`#2a78d6` light /
  `#3987e5` dark), dominated grey (`#898781`) — emphasis form, CVD-safe.
- One axis, never dual-axis. Log scale on price is essential.
- Thin marks, hairline solid gridlines (never dashed), 2px surface ring on
  scatter dots, ≥16px invisible hit targets, direct labels on every point.
- Dark mode is a selected palette, not an inversion (both
  `@media (prefers-color-scheme: dark)` and `:root[data-theme="dark"]`).
- Every value reachable without hover (table view). Tooltips enhance, never gate.
- Labels inserted with `textContent`, never `innerHTML`.
