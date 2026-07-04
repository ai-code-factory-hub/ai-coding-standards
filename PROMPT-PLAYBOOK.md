# Universal Project Build Playbook

A reusable, project-agnostic sequence of prompts to take any project from
"a folder of documents" to a working, NFR-complete application — with a strong
knowledge base, mapped requirements, and no drift.

HOW TO USE
- Replace every [BRACKETED] value with your own. In the Word copy these are RED.
- Run the prompts in order. Steps 0-4 are once per project. Steps 6-9 repeat per module.
- Always run PLAN (Step 4) before any build. Never jump from docs to "build it."

VARIABLE LEGEND (replace these)
- [PROJECT_NAME]         e.g. Acme
- [DOMAIN]               e.g. cloud multi-tenant Laboratory Information System for Indian labs
- [REQUIREMENT_DOC]      the PRD / functional spec filename(s)
- [STANDARDS_DOC]        the engineering-standards / NFR handbook filename
- [DESIGN_TOKENS_DOC]    the design-language / tokens filename
- [EXTRA_DOCS]           any other inputs (seed data, state machines, mockups) or "none"
- [PLATFORM]             web / mobile / both
- [STACK_PREFERENCE]     your stack, or "you recommend one"
- [PHASE_NO]             e.g. 0, 1, 2...
- [MODULE_ID]            e.g. M1
- [MODULE_NAME]          e.g. Identity & Tenancy
- [KB_PAGES]             the knowledge-base pages that scope this slice
- [DONE_CRITERIA]        the concrete outcome that proves the slice works
- [OPEN_DECISION_ANSWERS] your answers to the open product decisions

Recommended KB shape: MANY files, one per topic, plus README.md (index) and
REQUIREMENTS-DIGEST-AND-GAPS.md. Not one giant file.


==================================================================
STEP 0 - TOOLCHAIN (once per machine)
==================================================================
Use when: a fresh machine with no dev tools installed.

    My machine has no dev toolchain yet. Write me a single one-time bootstrap
    script for [macOS / Windows / Linux] that installs everything [PROJECT_NAME]
    needs for a [STACK_PREFERENCE] project (runtime, package manager, database).
    Make it idempotent and safe to re-run. Don't build anything else yet.


==================================================================
STEP 1 - INGEST & BUILD THE KNOWLEDGE BASE (once)
==================================================================
Use when: starting a new project. This is the most important step.

    New project: [PROJECT_NAME] - [DOMAIN].
    The source documents are in this folder:
    - [REQUIREMENT_DOC]   = the requirements (WHAT to build)
    - [STANDARDS_DOC]     = engineering standards (HOW WELL - the NFRs)
    - [DESIGN_TOKENS_DOC] = design language (how it LOOKS)
    - [EXTRA_DOCS]

    Read ALL of them fully. Then build a cross-linked knowledge base:
    - a knowledge-base/ folder, ONE markdown file per topic (not one big file),
    - a README.md index with a one-paragraph mental model of the product,
    - every requirement, module, screen, and NFR captured and cross-linked.
    Treat the requirement doc as WHAT, the standards doc as HOW-WELL, the tokens
    as LOOK. Do NOT write any application code yet. Show me the KB index when done.


==================================================================
STEP 2 - GAP ANALYSIS (once; can follow Step 1 in the same run)
==================================================================
Use when: right after the KB is built, before deciding anything.

    Now produce knowledge-base/REQUIREMENTS-DIGEST-AND-GAPS.md:
    - Part A: what each document defines (a digest).
    - Part B: cross-document inconsistencies to reconcile.
    - Part C: modules described but not fully specified (coverage gaps).
    - Part D: logic named but not defined (rules/algorithms missing).
    Rank the gaps by build priority. List every decision that is genuinely mine
    to make. Do not build. This is my "read before build" page.


==================================================================
STEP 3 - ARCHITECT: CONSTITUTION + DECISIONS (once)
==================================================================
Use when: the KB and gaps are ready. This is what stops you re-asking / drifting.

    Act as Solutions Architect. Based on the knowledge base, give me ONE clear
    recommendation (not a menu) for each: tech stack, multi-tenancy model,
    repo/folder structure, auth model. One line of "why" each.
    Then write two files:
    - CLAUDE.md  = the constitution: locked stack, conventions, NFR guardrails
      from [STANDARDS_DOC], and an explicit "Do NOT ask me about X" list.
    - DECISIONS.md = an append-only log: every locked decision (as short ADRs),
      an "Open questions" list (decisions only I can make), and an "Assumptions"
      section you append to whenever you pick a default while building.
    Ask me ONLY the decisions that are genuinely mine. Everything else: pick a
    sensible default and record it.


==================================================================
STEP 4 - CREATE THE DELIVERY PLAN (once)  <-- always do this before building
==================================================================
Use when: the foundation is locked. This is the "create a plan" step.

    Create DELIVERY-PLAN.md. Break the whole product into phases, foundation
    first (identity/tenancy/audit before anything that writes real data).
    For each phase give: goal, the modules it delivers, concrete deliverables,
    and which NFR domains it locks in. Map EVERY module and EVERY NFR domain to
    a phase - nothing dropped. Add honest effort estimates and separate "build
    velocity" from "calendar" (calendar depends on my reviews, open decisions,
    and external credentials). Do not build yet. Also give me a Word copy.


==================================================================
STEP 5 - (OPTIONAL) SEMANTIC KB/CODE LAYER - gbrain
==================================================================
Use when: after Phase 0, once the KB + codebase are large enough that
"where was that rule?" starts costing time. Optional.

    Set up gbrain for this project so you can semantically search the knowledge
    base and code. Initialize a local brain, index knowledge-base/ and the repo,
    register it as an MCP tool, and from now on use it to retrieve the relevant
    KB page or code file before each slice. Keep it synced as the code grows.


==================================================================
STEP 6 - BUILD ONE SLICE (repeat per module - your everyday workhorse)
==================================================================
Use when: building. This is 90% of day-to-day. Same shape every time.

    Build Phase [PHASE_NO] - Module [MODULE_ID] ([MODULE_NAME]).
    Source: knowledge-base/[KB_PAGES] + [STANDARDS_DOC].
    Done = [DONE_CRITERIA].
    Autonomy: build it, follow CLAUDE.md, log any assumptions in DECISIONS.md,
    write tests for all logic and state transitions, add audit + tenant scope by
    default. Don't stop to ask unless a choice is irreversible, costs money, or
    is a safety/product call only I can make. Hand me a git diff to review.


==================================================================
STEP 7 - OVERNIGHT AUTONOMOUS RUN (when you want maximum built unattended)
==================================================================
Use when: you've pre-answered the open decisions and want a long unattended run.
First paste your answers so nothing blocks me: [OPEN_DECISION_ANSWERS]

    Autonomous build session. Here are answers to the open decisions:
    [OPEN_DECISION_ANSWERS]
    Now work through DELIVERY-PLAN.md Phase [PHASE_NO], then as far into the next
    phase as you can, without stopping for reversible decisions.
    Rules:
    - Follow CLAUDE.md. Log every assumption in DECISIONS.md as you go.
    - If you hit a blocking decision, DON'T wait: pick the safest default, log it
      under "Assumptions (revisit)", keep moving.
    - Commit after each working slice with a clear message.
    - Keep a BUILD-LOG.md: what you built, what you assumed, what needs my review.
    I'll review the diffs and BUILD-LOG.md when I'm back.

    REALISTIC EXPECTATION: one long run = a complete running vertical (e.g. the
    foundation + first real feature slice), committed and tested - NOT an entire
    multi-module product production-hardened with live third-party integrations.


==================================================================
STEP 8 - VERIFY & REVIEW (after each slice)
==================================================================
Use when: a slice claims "done".

    Verify Phase [PHASE_NO] / [MODULE_ID] actually works end to end: run it,
    drive the real flow, show me the result - not just that tests pass. Then run
    a code review for correctness and simplification, and fix what you find.
    Summarize what changed and anything I should decide.


==================================================================
STEP 9 - PHASE RELEASE GATE (at the end of each phase)
==================================================================
Use when: a whole phase is done, before calling it shippable.

    Run the master release checklist from [STANDARDS_DOC] against Phase
    [PHASE_NO]: security, tenancy isolation, audit coverage, performance budgets,
    tests, accessibility, compliance. Give me a Go / No-Go with the gaps listed.


==================================================================
STEP 10 - RESUME / STATUS (any time you come back)
==================================================================
Use when: starting a new session on an in-progress project.

    Read CLAUDE.md, DECISIONS.md, DELIVERY-PLAN.md, and BUILD-LOG.md. Tell me
    where we are: what's done, what's in progress, what's blocked on my decisions,
    and the single best next slice to build. Then wait for my go-ahead.


==================================================================
THE GOLDEN RULES (why this works)
==================================================================
1. Three documents, three jobs: requirements = WHAT, standards = HOW WELL,
   tokens = LOOK. Keep them distinct and the gaps disappear.
2. Always: Ingest -> Gap -> Decide -> PLAN -> Build. Never docs -> "build it".
3. The unit of work is ONE module/screen, never "the app".
4. CLAUDE.md + DECISIONS.md + git = I self-manage drift. Decisions are made once
   and written down; defaults are logged, not re-asked.
5. Every slice states its "Done =" up front. No "Done =", no build.
