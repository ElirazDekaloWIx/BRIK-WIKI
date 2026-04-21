# AI Conversation Handoff — BRIK Project

**Purpose:** Allow a new AI conversation to pick up context and continue work on BRIK's product/strategy documentation without losing continuity.

**Date of handoff:** 2026-04-21
**User:** Eliraz Dekalo (elirazd@wix.com)
**Working environment:** Vibe-coding setup, Windows, project path `C:/Users/user/Desktop/Code/_BRIK/`

---

## 1. What is BRIK

**BRIK** (internal code names: Clay / Chatooly) is a live production platform by Wix that generates **creative micro-apps** from natural-language prompts.

- **Output:** a browser-based tool with a canvas (2D / WebGL / p5.js / Hydra) + a parametric controls sidebar (sliders, toggles, color pickers)
- **Flow:** user chats → AI agent designs controls + writes JavaScript → rendered in an iframe via an assembler
- **Community loop:** users can remix, publish to a gallery, export as `.clay.html` / PNG / MP4
- **Target audience (newly defined):** designers and creators across ALL disciplines (14 clusters: graphic, web, motion, industrial, fashion, type, generative, social, VJ, game/XR, spatial, education, art directors, freelance/hobbyist)

### Strategic vision (just established)
> BRIK aims to **replace** design software in the designer's workflow, OR at minimum **be in the pipeline of every project** (not exclusive). This means strong exports + integrations (AE plugin, Figma, industry formats, `.brik` open format).

### Core philosophy (in agent prompts)
- **Golden Handcuffs:** full creative freedom within a fixed template + design system
- **Silence is not an invitation:** if the user didn't specify, don't invent — expose as parameter
- **Parametric output:** tool is parametric, not a finished piece

---

## 2. Key Repositories

1. **Source code analyzed:** `https://github.com/yaelren/clay-frontend-split` (cloned to `C:/Users/user/Desktop/Code/_BRIK/clay-frontend-split/`)
2. **Wiki (this document lives here):** `https://github.com/ElirazDekaloWIx/BRIK-WIKI`
3. **Live wiki:** `https://elirazdekalowix.github.io/BRIK-WIKI/`
4. **Backlog spreadsheet:** `https://docs.google.com/spreadsheets/d/1jbTKYKem6n8QkoKOj0wc6l74w8EWBdIVlJflqEpattw/edit` (52 items, 5 tiers — now public)

---

## 3. Wiki Document Structure (what to read first)

The wiki has reorganized into **6 primary tabs** with sub-tabs:

### Primary tabs:
1. **סקירה (Overview)** — product summary, personas, thesis
2. **המוצר (Product) ▾**
   - ארכיטקטורה (Architecture) — data model, assembler, pipeline
   - סוכן ו-Skills (Agent & Skills) — weaknesses, 5 understanding dimensions
   - BRIKS — declared platform capabilities (terminology)
   - פרסונות (Users) — 6 personas
   - 207 תרחישים (Stories) — use cases across 14 designer disciplines (now in pure Hebrew)
   - שפת מוצר (Terminology) — unified BRIK vocabulary
3. **מחקרים (Research) ▾**
   - R1 · סוכנים — 24 agent patterns, Anthropic Context Engineering, 29 academic refs
   - R2 · מתחרים — 25 competitors + Tier-0 recent additions (Claude Design, Omma, Figma Weave, Google Stitch 2.0, OpenAI Apps SDK) + 5 comparison matrices (head-to-head vs BRIK)
   - R3 · טכנולוגיות — WebGPU, fal.ai, SAM 2, Poly Haven, etc. + Top 20 ranked + licensing
4. **אסטרטגיה (Strategy) ▾**
   - בידול ו-Moat
   - סיכונים — 53 problems taxonomy
   - שאלות פתוחות — 11 strategic questions (Q1-Q11)
5. **תכנון (Planning) ▾**
   - 🗺️ Roadmap — user-first, 4 parallel streams, Vibe-coding velocity
   - PRD · עתיד — living document (keep updating)
   - PRD · עבר — retrospective (stable)
   - המלצות — top 10 actions
   - Backlog — 52 items + team feedback cross-reference + 8 new feature requests
6. **נספחים (Appendix)** — glossary, update log, references

### Companion files:
- `team-feedback.html` — standalone HTML for Eliraz's team meeting (17 team items + 13 additions, with print button)
- `backlog-raw.csv` — source data
- `research-technologies-raw.md` — full 766-line tech research

---

## 4. Terminology Established (BRIK Language System)

Use these terms consistently throughout the product:

**Platform & output:**
- BRIK = the product AND the unit of tool
- BRIKWORK = composition of multiple BRIKs
- BRIKYARD = the public gallery
- BRIKS = layer of declared platform capabilities

**Verbs:**
- Cast (generate) / Mold (edit) / Forge (build from scratch) / Set (publish) / Stack (remix) / Chip (tweak) / Dust off (undo/restore)

**Surfaces:**
- FORGE (editor) / STAGE (canvas) / PANEL (controls) / CHATROOM (chat) / CABINET (my work)

**Agents:**
- Architect (Creative Director agent) / Caster (Creative Coding Agent)

**Community:**
- Masons (the community) / Masterwork (featured) / Trail (fork history) / Vault (brand archive)

**Format:** `.brik` (replacing `.clay.html` — strategic open format mirroring Rive's `.riv`)

---

## 5. Critical Technical Findings (from code analysis)

### Agent pipeline (current)
- 3 agents: Triage → Art Director (first message only!) → Creative Coding Agent (Mastra-based, 11 tools)
- Skill matching: keyword+stemming (doesn't scale past ~200 skills)
- No prompt caching (50-70% cost savings missed)
- No tiered model routing (Haiku for conversational, Sonnet for AD, Opus for coder)
- Art Director one-shot (contradicts own prompt that says "accompany through iterations")
- No feedback loop / no critic / no verifier

### Security (4 CRITICAL risks)
1. JS template literal injection in assembler
2. sandbox="allow-scripts allow-same-origin" (negates protection)
3. No CSP header
4. No library CDN URL whitelist
5. postMessage with "*" everywhere

### Reliability
- Zero atomicity in agent writes (tool can break mid-write)
- Silent error swallowing in session.ts (console.error instead of throw)
- No rate limiting (financial risk — one user can burn $10K/night)
- Full iframe reload on every agent write (catastrophic UX)

### Missing entities/features (but pre-planned)
- `ToolVersion` entity exists but NOT EXPOSED in UI (biggest low-hanging fruit)
- Credits system fully designed in `docs/credits-system-plan.md` but not implemented
- `Feedback` entity exists but not wired to UI

### Stack
- Frontend: React 18 + TypeScript + Vite + Tailwind + shadcn/ui
- Backend: Base44 BaaS (Deno functions) — **VENDOR LOCK-IN CONCERN**
- Mastra agent runtime — `@mastra/core@1.4.0` — startup, stability risk
- Models: Claude, GPT-4o, Gemini via Mastra providers

---

## 6. Competitive Landscape (Tier-0 CRITICAL)

1. **Claude Design (Anthropic, Apr 17, 2026)** — existential threat, same Claude model, has "adjustment sliders for spacing/color/layout" (BRIK's parametric thesis), "shaders and 3D" mentioned. Window: 6-12 months.
2. **Omma by Spline (Mar 24, 2026)** — 3D + apps + games, Community gallery, Native app export (.apk/Xcode). Spline's 30M-user distribution. Funding $32M.
3. **Figma Weave (Apr 2026 relaunch)** — node-graph workflow canvas with 20+ templates
4. **Google Stitch 2.0** — voice + multi-screen + free
5. **Google AI Studio App Gallery** — remix primitive already productized
6. **OpenAI Apps SDK** — mini-apps INSIDE ChatGPT. Strategic threat to "BRIK as destination"

---

## 7. User Personas (6)

| # | Persona | Fit | Status |
|---|---------|-----|--------|
| A | Creative Technologist (knows code) | 7/10 | Good |
| B | Designer Who Codes (partial) | 9/10 | **Sweet spot** |
| C | Designer Who Doesn't Code | 4/10 | Silent churn |
| D | Art/Creative Director | 6/10 | Decent |
| E | Tinkerer / Hobbyist | 8/10 | Sweet spot |
| F | AI-Native Creator | 5/10 | Silent churn |

**Key insight:** Landing page attracts C+F. UX serves B+E. Unaddressed silent churn.

---

## 8. Top Open Questions (as of handoff)

1. **Q1** — Who is the ONE user? B+E (narrow) or C+F (aspirational)? Decides roadmap direction.
2. **Q2** — "p5.js with AI" or "Artifacts with personalization"? Different products.
3. **Q3** — Timeline: extension or pivot? Not started without strategic decision.
4. **Q5** — What brings user back in week 4? Retention unsolved.
5. **Q7** — Base44 vendor lock-in: stay, abstract now, or migrate? Costs increase over time.
6. **Q8** — Mastra framework stability? Need adapter layer?

### H-07 (Horizon item in backlog) deserves elevation:
"Intent Separation: Creators vs Users" — split UI. Directly answers Q1/Q2. Team already considering.

---

## 9. Roadmap at a Glance (Vibe-Coding Adjusted)

**Week 1-2 (P0 — gate to scale):**
- Security: CSP + template escape + URL whitelist + rate limiting
- Prompt caching (50-70% cost savings)
- Bug fixes: Remix race, bakeDefaults, canvas crop

**Week 3-4 (P1 — user unblockers):**
- Controls Editor v1 (B-01 Top user request)
- Tiered model routing
- Test baseline

**Month 2 (Retention):**
- Version UI (ToolVersion entity exists, just expose)
- Brand Kit v1
- Atomic writes
- Notifications
- Embeddings skill matching

**Month 3 (Agent quality):**
- Living Art Director (Persistent Brief — answers team's "Plan Mode" + "Context")
- Critic Agent (Haiku, evaluator-optimizer)
- Asset Providers layer (Poly Haven + AmbientCG CC0)
- Export Presets + GIF + Loop

**Q2 (Moat):**
- WebGPU renderer
- Figma integration
- Mobile consumption-first
- Creator analytics

**Q3+ (Ecosystem):**
- `.brik` open format + NPM runtime
- AE + Cavalry plugins
- BrikFlow (node composition)
- Mobile authoring

---

## 10. Team Feedback (from launch film work — critical)

17 items surfaced by the team:
- **Most critical per team:** Undo / Version UI (they said "הכי קריטי")
- Plan mode + Context persistence (= Living Architect)
- Projects workspace + Collaboration
- bakeDefaults bug (publish loses values)
- Remix race condition
- Canvas crop bug
- Video export quality
- Drag image to chat
- Canvas memory (size + bg)

See `team-feedback.html` for full formatted list with 13 additional items.

---

## 11. Working Methodology Notes

- **Vibe coding environment** — strong devs + technologist designers + technical people
- Things can run fast, in parallel with small human dev power
- AI-assisted dev (Claude Code, Cursor) accelerates ~70% of work
- Sprint = 1 week (not 2-3 weeks)
- 4 parallel streams concurrently possible
- What doesn't scale with AI: design decisions, cross-team coordination, user testing, security review

---

## 12. User's Communication Style Notes

- Hebrew (Israeli, Wix employee)
- Wants direct/brutal honesty, no hedging ("אל תהיה עדין")
- Wants references/sources next to every claim
- Wants consistent updating across ALL relevant sections when something changes
- Wants update log (יומן עדכונים) to document every change
- Auto-mode: execute autonomously, don't wait for approval
- Strong preference for comparisons in table form (not prose)
- Unified terminology from BRIK universe (for marketing video use)

---

## 13. What's Pending (as of handoff)

- Strategic decisions on Q1/Q2/Q3/Q5 (see Open Questions tab)
- Decision on `.brik` open format commitment (Rive-style moat)
- Decision on Collaboration vs Projects priority
- Design decisions for Controls Editor, Brand Kit, Version UI interfaces
- Still need real user analytics before final Q1 decision

---

## 14. How to Resume Conversation (for new AI)

1. **First, read the live wiki** at the GitHub link above, or pull the repo.
2. **Start by confirming understanding** — summarize back what BRIK is, core tension, and current roadmap.
3. **Check for new commits** in BRIK-WIKI repo — user may have added things.
4. **Pull latest `team-feedback.html`** — user may have updated after their meeting.
5. **Preserve terminology** — use BRIK vocabulary consistently.
6. **Honor User-First principle** — every new feature proposal should answer "what user blocker does this remove?"
7. **Honor the 3 decision tests** — Thesis / Moat / Golden Handcuffs — for any proposed feature.
8. **Update update log for every change** — user explicitly asked.
9. **Use web research for current info** — training cutoffs may miss new launches.
10. **Update ALL related sections when changing something** — user explicitly asked.

---

## 15. Contact & Git Credentials

- Git identity: `Eliraz Dekalo <elirazd@wix.com>`
- GitHub user: `ElirazDekaloWIx`
- Push branch: `main`
- GitHub Pages enabled on BRIK-WIKI

---

**End of handoff. Everything above is current as of 2026-04-21.**
