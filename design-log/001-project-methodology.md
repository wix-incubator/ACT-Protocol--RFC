# Design Log #1: Project Methodology

## Background

The ACT Protocol project is an **RFC/specification project**, not a code implementation project. The primary deliverable is the proposal document (`docs/proposal.md`).

## Problem

Standard design log methodology assumes code implementation as the end result. We need to adapt the workflow for a specification-authoring project where the proposal itself is the artifact.

## Design

### Artifact Roles

| Artifact | Role |
|---|---|
| `docs/proposal.md` | **The deliverable.** The canonical, self-contained specification. Should read cleanly on its own. |
| `design-log/*.md` | **Working notes.** Exploration, trade-off analysis, Q&A, and rationale for proposal changes. |

### Workflow

1. **New topic or gap identified** -- create a design log to discuss options, trade-offs, and open questions
2. **Design log reaches conclusion** -- distill the result into proposal language
3. **Update proposal** -- integrate the conclusion into `docs/proposal.md`
4. **Design log stays as-is** -- serves as historical rationale; not updated after proposal is written

### What Goes Where

| Content | Where |
|---|---|
| Final spec language (normative) | Proposal |
| "Why did we choose X over Y?" | Design log |
| Open questions before a decision | Design log |
| Examples that clarify spec behavior | Proposal |
| Alternatives considered | Design log (summary in proposal §7 if significant) |

### Conventions

- Design logs are numbered sequentially: `NNN-short-title.md`
- The proposal version is bumped when significant sections are added or changed
- Design logs reference proposal sections (e.g., "§6.3") and vice versa when useful
