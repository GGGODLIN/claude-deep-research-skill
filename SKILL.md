---
name: deep-research
description: 多源驗證 + citation 追蹤 + 結構化研究報告的深度研究 skill。中文觸發：「深度研究 X」「深入研究 X」「完整分析 X」「幫我研究一下 X」「X 的現況 / 趨勢 / 全貌」「研究報告」「比較 X 跟 Y」「X vs Y 全面評估」「X 選哪個」「X 怎麼選」「整理一下 X 的 landscape」。英文觸發：「deep research」「comprehensive analysis」「research report」「compare X vs Y」「analyze trends」「state of the art」。不要用在：簡單查詢、debug、一兩次搜尋就答得出的事實題（交給 research-before-answer 或一般 WebSearch）。
---

# Deep Research

## Core Purpose

Deliver citation-tracked research reports through a structured pipeline with evidence persistence, source identity management, claim-level verification, and progressive context management.

**Autonomy Principle:** Operate independently. Infer assumptions from context. Only stop for critical errors or incomprehensible queries. Surface high-materiality assumptions explicitly in the Introduction and Methodology rather than silently defaulting.

---

## Decision Tree

```
Request Analysis
+-- Simple lookup? --> STOP: Use WebSearch
+-- Debugging? --> STOP: Use standard tools
+-- Complex analysis needed? --> CONTINUE

Engine Routing (ASK EVERY TIME -- see "Engine Routing" section below)
+-- 本 skill 管線 --> Mode Selection (below)
+-- 官方 built-in workflow --> Workflow({name:"deep-research"}); skill 不往下
+-- 平行對照 --> 背景官方 workflow + 前景本 skill 管線

Mode Selection (only when "本 skill 管線" is chosen)
+-- Initial exploration --> quick (3 phases, 2-5 min)
+-- Standard research --> standard (6 phases, 5-10 min) [DEFAULT]
+-- Critical decision --> deep (8 phases, 10-20 min)
+-- Comprehensive review --> ultradeep (8+ phases, 20-45 min)
```

**Default assumptions:** Technical query = technical audience. Comparison = balanced perspective. Trend = recent 1-2 years.

---

## Engine Routing (ask every time)

After the STOP gate passes (this genuinely needs deep research), ALWAYS ask the user which engine to run BEFORE anything else. Use the `AskUserQuestion` tool, single-select, with these three options (繁中 labels shown to the user):

1. **本 skill 管線** — main-session structured pipeline: citation tracking, `evidence.jsonl`/`claims.jsonl` persistence, McKinsey HTML/PDF, 繁中輸出。慢但可追溯、可交付。
2. **官方 built-in workflow** — call `Workflow({name:"deep-research", args:"<topic> — 請以繁體中文輸出報告"})`。8-agent 真並行，快、廣度足、對抗式驗證。⚠️ args 必須註明繁中，否則官方那支預設吐英文。
3. **平行對照** — 背景起官方 workflow + 前景同時跑本 skill 管線，兩邊都回來後並排對照發現與品質差異。花雙倍 token，適合重要題目或評估期。

**Routing after the answer:**
- 本 skill 管線 → proceed to Mode Selection and the 8-phase workflow below.
- 官方 built-in workflow → invoke the Workflow tool; this skill's pipeline is NOT run. Relay the workflow's cited findings.
- 平行對照 → start the official workflow in the background, run this skill's pipeline in the foreground, then present a side-by-side comparison.

**Only exception to asking:** the current request already names an engine explicitly (e.g. 「用官方 workflow 跑」「用 skill 出 PDF 報告」「平行跑」) — honor it directly without re-asking.

---

## Workflow Overview

| Phase | Name | Quick | Std | Deep | Ultra |
|-------|------|-------|-----|------|-------|
| 1 | SCOPE | Y | Y | Y | Y |
| 2 | PLAN | - | Y | Y | Y |
| 3 | RETRIEVE | Y | Y | Y | Y |
| 4 | TRIANGULATE | - | Y | Y | Y |
| 4.5 | OUTLINE REFINEMENT | - | Y | Y | Y |
| 5 | SYNTHESIZE | - | Y | Y | Y |
| 6 | CRITIQUE | - | - | Y | Y |
| 7 | REFINE | - | - | Y | Y |
| 8 | PACKAGE | Y | Y | Y | Y |

**Note:** Phases 3-5 operate as an evidence loop per section (retrieve → evidence store → refine outline → draft → verify claims → delta-retrieve if needed), not as strict sequential gates.

---

## Execution

**On invocation, load relevant reference files:**

1. **Phase 1-7:** Load [methodology.md](./reference/methodology.md) for detailed phase instructions
2. **Phase 8 (Report):** Load [report-assembly.md](./reference/report-assembly.md) for progressive generation
3. **HTML/PDF output:** Load [html-generation.md](./reference/html-generation.md)
4. **Quality checks:** Load [quality-gates.md](./reference/quality-gates.md)
5. **Long reports (>18K words):** Load [continuation.md](./reference/continuation.md)

**Templates:**
- Report structure: [report_template.md](./templates/report_template.md)
- HTML styling: [mckinsey_report_template.html](./templates/mckinsey_report_template.html)

**Scripts:**
- `python scripts/validate_report.py --report [path]`
- `python scripts/verify_citations.py --report [path]`
- `python scripts/md_to_html.py [markdown_path]`

---

## Output Contract

**Required sections:**
- Executive Summary (200-400 words)
- Introduction (scope, methodology, assumptions)
- Main Analysis (4-8 findings, 600-2,000 words each, cited)
- Synthesis & Insights (patterns, implications)
- Limitations & Caveats
- Recommendations
- Bibliography (COMPLETE - every citation, no placeholders)
- Methodology Appendix

**Output files (all to `~/Documents/[Topic]_Research_[YYYYMMDD]/`):**
- Markdown (primary source of truth)
- `sources.jsonl` — stable source registry with canonical IDs
- `evidence.jsonl` — append-only evidence store with quotes and locators
- `claims.jsonl` — atomic claim ledger with support status
- `run_manifest.json` — query, mode, assumptions, provider config
- HTML (McKinsey style, auto-opened)
- PDF (professional print, auto-opened)

**Quality standards:**
- 10+ sources, 3+ per major claim (cluster-independent, not just count)
- All factual claims cited immediately [N] with evidence backing in `evidence.jsonl`
- Claim-support verification mandatory: no unsupported factual claims pass delivery
- No placeholders, no fabricated citations
- Prose-first (>=80%), bullets sparingly

---

## When to Use / NOT Use

**Use:** Comprehensive analysis, technology comparisons, state-of-the-art reviews, multi-perspective investigation, market analysis.

**Do NOT use:** Simple lookups, debugging, 1-2 search answers, quick time-sensitive queries.
