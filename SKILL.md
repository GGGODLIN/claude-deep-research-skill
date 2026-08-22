---
name: deep-research
description: 多源驗證 + citation 追蹤 + 結構化研究報告的深度研究 skill。觸發：「深度研究 / 深入研究 / 完整分析 X」「研究報告」「比較 X 跟 Y / X vs Y / X 怎麼選」「X 的現況 / 趨勢 / landscape」；英文 deep research / comprehensive analysis / research report。不要用在：簡單查詢、debug、一兩次搜尋就答得出的事實題（交 research-before-answer）。
upstream: 199-biotechnologies/claude-deep-research-skill
upstream-path: SKILL.md
upstream-pinned: f2f2c0f
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

Pre-scout Gate (see "Pre-scout Gate" section below)
(skip 此 gate 直接 dispatch engine 是常見誤用 — 多半燒了一輪才發現 source 爛掉)

Engine Routing (ASK EVERY TIME -- see "Engine Routing" section below)
+-- 本 skill 管線 --> Mode Selection (below)
+-- 官方 workflow（限流版）--> Workflow({name:"deep-research-paced"}); 只勾它時 skill 不往下
+-- ChatGPT DR 委外 --> see "ChatGPT Deep Research 委外" section
+-- 多選: 勾幾個跑幾個並行 (1+2 = 舊「平行對照」)

Mode Selection (only when "本 skill 管線" is chosen)
+-- Initial exploration --> quick (3 phases, 2-5 min)
+-- Standard research --> standard (6 phases, 5-10 min) [DEFAULT]
+-- Critical decision --> deep (8 phases, 10-20 min)
+-- Comprehensive review --> ultradeep (8+ phases, 20-45 min)
```

**Default assumptions:** Technical query = technical audience. Comparison = balanced perspective. Trend = recent 1-2 years.

---

## Pre-scout Gate

Engine routing cold-starts: it doesn't see the user's existing memory clusters, wiki pointers, or prior `runs/` reports. Most deep-research invocations are補洞, not first-time exploration — half the relevant sources are already named somewhere in the user's brain. Five minutes of main-session scout wires those into the URL pool the engine will inherit, AND surfaces dead sources (302/404/login-wall/JS-only) that engine fetch agents silently abstain on.

F22 2026-06-15: three batches × 5.4M tokens each silently abstained on `platform.xiaomimimo.com` because nobody followed the 302 to `mimo.mi.com` — main session direct WebFetch caught it in 30 seconds. The workflow-side discipline that pairs with this scout is `workflow-hardening` §9 (source-health).

**Scout has high ROI when:**

- A reference wiki / memory cluster names candidate sources (e.g. `_index_cc_model_swap` → vendor primary URLs)
- The topic decomposes into N enumerable candidates (vendor docs, repos, papers)
- 補洞 a previously-researched topic — every reflow after first is scout-first

**Scout has low ROI / skip when:**

- Concept question with no fetchable source (architecture / design pattern thinking, abstract design questions)
- Cold-start truly nothing in memory / wiki / prior runs to hint sources — but consider: a 5-min scout that returns 0 sources is also useful (confirms cold-start status before engine cold-starts)

**Scout steps (~5 min, main session):**

1. Pull source candidates from `~/.claude/memory/_index_*.md`, wiki pointers, prior `runs/` reports
2. Parallel WebFetch each landing page — log 302 redirects (follow them), 404s, login walls, suspiciously short renders
3. Skim each live page — note what's trivially extractable now vs what needs verification / contention work

**Perspective sub-scout（可選，題目發散時才跑）**

源自 stanford-oval/storm Perspective-Guided Question Asking 概念，治「題目沒定義清楚 / scope 過大 / 結果發散」痛點。跟上面 URL pool scout 並行跑、不互斥。事實盤點 / roster 批次類題目直接跳過。

Steps（main session 跑，scout 階段同時做）:

1. 找 3-5 個對位 artifact（GitHub awesome list / 既有 landscape post / 同類 wiki entry / Sequoia/Latent Space 之類 market map）
2. 觀察這幾篇共用哪些 perspective 軸（產品形態 / autonomy 軸 / 商業模式 / 對手 / 目標用戶 之類）
3. 抽 4-7 個 perspective set，給 user 點頭再進 Engine Routing
4. Perspective set 作 Engine routing args 的補強——`deep-research-paced` 的 angle decomposition 依 perspective set 分軸

低信心 fallback：找不到 ≥3 對位 artifact → 標「⚠️ 低信心 perspective set」並 fallback 讓 LLM 自己想；user 可選 skip。

觀察狀態（2026-07-25）：5 週零完整 scout run、母場景本身閒置，機制未受考驗；轉事件驅動觀察，backstop 對帳日 2026-08-24（見 `~/Desktop/projects/.claude/trials/active.md` storm-perspective-graft entry）。

**Gate B (after scout, decide before Engine Routing):**

- **Skip engine** — scout already covers what's needed → main-session integration directly
  ⚠️ Skip 等於放棄 3-vote adversarial verify。Vendor primary 不 verify 就會吞 cherry-pick (例：vendor 跨家比較表常用前代對照模糊現役落差)。若題目對 vendor 行銷誇大敏感、改選 narrow engine run 把可疑數字丟去 verify、別 skip。
- **Narrow engine run** — scout covers most, gaps need verify or cross-source contention → engine runs only on gap axes, inherits scouted URLs
- **Full engine run** — true cold-start or large unknown surface → engine runs angle decomposition from scratch

**完成 Scout + Gate B 後 → Engine Routing (next section)**

---

## Engine Routing (ask every time)

(Reached this section only after Pre-scout Gate passed and Gate B chose "narrow" or "full". If Gate B chose "skip", this whole section is bypassed.)

After the STOP gate passes (this genuinely needs deep research), ALWAYS ask the user which engines to run BEFORE anything else. List the three options as inline text in the response, then end the turn and wait for the answer — **多選，勾一至多個** (`AskUserQuestion` is globally denied since 2026-07-05; see memory `feedback_no_askuserquestion_inline_options_instead`). 繁中 labels shown to the user:

1. **本 skill 管線** — main-session structured pipeline: citation tracking, `evidence.jsonl`/`claims.jsonl` persistence, McKinsey HTML/PDF, 繁中輸出。慢但可追溯、可交付。
2. **官方 workflow（限流版）** — call `Workflow({name:"deep-research-paced", args:"<topic> — 請以繁體中文輸出報告"})`。對齊官方品質（3-vote、25 claims、繼承 session model、對抗式驗證），但 verify 分批跑（peak 並發 6）避開 Opus 端點的 burst 限流：完整、0 撞限，惟比原版慢約 2.5x。⚠️ args 必須註明繁中，否則預設吐英文。若要不限流的原版（非 Opus 端點較快；但 **Opus 端點會撞 burst 限流、findings 拿半套**）→ 使用者明說「用官方原版跑」才改走 `Workflow({name:"deep-research"})`。
3. **ChatGPT Deep Research 委外** — 丟一份給 ChatGPT 網頁版 Deep Research（吃 ChatGPT 訂閱額度、OpenAI 端非同步跑 10-30 min、零 CC token）。執行程序見下方「ChatGPT Deep Research 委外」段。

**多選規則**：勾幾個跑幾個並行。1+2 同勾 = 背景起 workflow、前景跑本 skill 管線，都回來後並排對照發現與品質差異（花雙倍 token，適合重要題目或評估期）。只勾 3 = 純委外：射出後等收割，報告標「未經本地 verify」，main session 只抽驗要引用的關鍵 claim，不跑完整管線。

**Routing after the answer（照勾選集合執行）:**
- 含 3 → 先跑「ChatGPT Deep Research 委外」步驟 1 射出，再啟動其餘勾選項。
- 含 1 → proceed to Mode Selection and the 8-phase workflow below.
- 含 2 → invoke `Workflow({name:"deep-research-paced"})`（僅當使用者明說「用官方原版跑」時改 `name:"deep-research"`）。只勾 2 而無 1 → this skill's pipeline is NOT run; relay the workflow's cited findings.

**Only exception to asking:** the current request already names an engine explicitly — honor it directly without re-asking. 關鍵字對應：「用官方 workflow 跑」/「用限流版跑」= `deep-research-paced`；「用官方原版跑」= 原版 `deep-research`（⚠️ Opus 端點會撞 burst 限流、拿半套）；「用 skill 出 PDF 報告」= 本 skill 管線；「平行跑」/「兩個都跑」= 勾 1+2；「順便丟 GPT DR」= 加勾 3。引擎被明示點名而跳過問句時，選項 3 也不另問，使用者明說才加。

---

## ChatGPT Deep Research 委外（選項 3 的執行步驟）

前提：OpenCLI 已裝且 bridge 活著（用法與紅線見 memory [[opencli-chatgpt-web-dispatch-2026-08-16]]）。產出定位是**一份待驗素材**，不繞過任何 engine 的 verify——ChatGPT 端 citation 品質未知，帶 citation ≠ 已驗。

1. **射出**（engine 選定後、主線開跑前）：
   ```bash
   opencli chatgpt ask "<refined research question>" --new --deep-research --wait false --window background -f json
   ```
   記下回傳 `conversationId`，並把它複誦在下一則給使用者的訊息裡（長跑 compact 後 context 可能丟失，訊息裡的 ID 是救援錨點）。指令失敗或 `opencli doctor` 不過 → 告知使用者、放棄本輪疊加，主線照常，不阻塞；只勾 3 時無主線可退，直接回報失敗、結束回合等指示。做完判準：conversationId 已出現在給使用者的訊息中，或已明講放棄／失敗。
2. **主線照常跑**所選 engine，不等委外。
3. **收割**（主線進 TRIANGULATE / synthesis 之前；只勾 3 時收割即主線）：
   ```bash
   opencli chatgpt deep-research-result <conversationId> --wait true --timeout 300 --window background -f md
   ```
   DR 常跑 10-30 min，首發超時是常態：每 5 分鐘重收一次、最多 6 次；輸出 `status` 欄非完成態 → 視同未完，不採用半成品文字。6 次仍未完 → 報告註明「委外未及時完成、未採用」。做完判準：拿到 status 完成的報告全文，或已註明未採用。
4. **併入**：素材標註來源「ChatGPT Deep Research（未經本地 verify）」餵進 synthesis；要引進最終報告的 claim 一律過所選 engine 的 verify 流程（本 skill claim verify 或 workflow 3-vote）；只勾 3 無 engine 時照多選規則的抽驗辦法——只抽驗要引用的關鍵 claim，不啟動完整 verify。做完判準：最終報告 Methodology/Sources 段列出委外素材「採用哪些／驗過哪些／棄掉哪些」三項，各有內容或明寫「無」。

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

**深度規則（hop 預算）：**

hop = 沿單一線索的延伸層數（種子來源內的引用 → 下一層來源 = 1 hop）。各檔位預算：

| 檔位 | hop 預算 |
|---|---|
| quick | 1 |
| standard | 2 |
| deep | 3 |
| ultradeep | 5 |

- 超出預算的線索**不追**，改記入報告尾端「未追的線索」清單：線索是什麼 + 為何在預算外（第幾 hop / 哪個檔位）
- 「未追的線索」清單為空時明寫「無」——沉默等於宣稱追完了，no silent caps
- 預算是深度上限、不是廣度上限；同一層平行多條線索不算 hop

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
