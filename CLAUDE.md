# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository nature

This is a **documentation-only repository** — software-engineering notes in Markdown. There is no source code to compile, no build system, no test suite, no linter, no package manifest. Standard "build / lint / test" instructions do not apply. Edits are validated by reading.

Top-level folders:

- `books/` — Concentrated summaries of canonical SE books (Clean Architecture, DDD, Building Microservices, Unit Testing, Head First Design Patterns).
- `oop/` — Object-Oriented Programming reference (four pillars, concepts, a realistic e-commerce walkthrough, cheat sheet).
- `principles/` — Language-agnostic principles: DRY/KISS/YAGNI/SoC (core), SOLID, GRASP, plus additional (LoD, POLA, etc.).
- `dot-net/docs/` — .NET-flavored taxonomy of patterns, organized by **level of abstraction** (see "Pattern taxonomy" below).
- `dot-net/interview/`, `java/` — empty placeholders.

Code samples in the docs are mostly **C#**; principles and pillars are presented as language-agnostic with C#/Java/TS examples.

## The bilingual paired-files rule (most important)

Every documentation file exists in **two versions** that must stay in sync:

- English: `foo.md`
- Vietnamese: `foo-vi.md`

All docs are currently 1:1 paired. When you change one, change the other in the same commit. New docs must be added in both languages at once.

Quick sanity check (should produce empty output):

```bash
diff <(find . -name "*.md" ! -name "*-vi.md" ! -name "CLAUDE.md" | sed 's|\.md$||' | sort) \
     <(find . -name "*-vi.md" | sed 's|-vi\.md$||' | sort)
```

What to preserve when editing one side:

- **Section structure** — section numbers, heading order, and Table of Contents must mirror between EN and VI. Heading count per section should match.
- **Code blocks** — keep code, identifier names, and comments inside code identical across versions. Translate the surrounding prose, not the code.
- **Technical terms stay English in headings** even on the VI side (e.g., `## 1. Encapsulation`, `## 2. Abstraction`). Only translate the natural-language parts ("Table of Contents" → "Mục lục", "What It Means in Code" → "Trong code nghĩa là").
- **Anchor IDs** — when a heading's prose is translated, its auto-generated anchor changes too. Update internal TOC links on the side you edited to match the localized headings (e.g., EN `#5-how-the-pillars-relate` vs VI `#5-các-trụ-cột-liên-hệ-ra-sao`).
- **VI banner** — Vietnamese files open with `> 🇻🇳 Phiên bản tiếng Việt. English: [foo.md](./foo.md)`. Keep the link working.
- **Quick Reference block** — every doc in the repo (currently 100%) has one. Don't drop it; translate its prose. Format:
  - EN: `## Quick Reference (What · Why · When · Where)` with bullets **What / Why / When / Where**.
  - VI: `## Tham chiếu Nhanh (Cái gì · Tại sao · Khi nào · Ở đâu)` with bullets **Cái gì / Tại sao / Khi nào / Ở đâu**.

## Pattern taxonomy (canonical in `dot-net/docs/README.md`)

When discussing or cross-linking patterns, respect the **level**. The repo explicitly separates these — don't compare items across levels as if they were alternatives (e.g., Saga is not an alternative to Decorator).

| Level | Folder | Examples |
| --- | --- | --- |
| Methodology | `dot-net/docs/ddd/` | DDD |
| Architectural Style | `dot-net/docs/architectural-style/` | Layered, Hexagonal, Onion, Clean, EDA, Vertical Slice |
| Architectural Pattern | `dot-net/docs/architectural-pattern/` | Saga, CQRS, Event Sourcing, Outbox |
| Enterprise Pattern | `dot-net/docs/enterprise-pattern/` | Repository, Unit of Work, IoC/DI, Lifetimes |
| Tactical DDD | `dot-net/docs/ddd/tactical-patterns.md` | Aggregate, Entity, Value Object, Domain Event |
| Strategic DDD | `dot-net/docs/ddd/strategic-patterns.md` | Bounded Context, Context Map, ACL |
| GoF Design Pattern | `dot-net/docs/design-pattern/` | Strategy, Observer, Decorator, … |

`dot-net/docs/README.md` contains a "Where Does X Belong?" lookup table — use it as the source of truth when classifying a new concept.

## Cross-folder linking

The four content areas (`books/`, `oop/`, `principles/`, `dot-net/docs/`) cross-reference each other heavily via relative links (e.g., a book summary links to the matching pattern doc; a principle links to the architecture that embodies it). When you add or rename a doc:

- Update the relevant README index table (both EN and VI).
- Check inbound links from sibling folders — `grep` for the old filename across the repo before renaming.
- Use relative paths (`../oop/`, `./pillars.md`), not absolute paths.

## Editorial voice

- Opinionated, dense, recall-oriented — these are notes, not tutorials. Lean toward tables and short bullets over long prose.
- Each pattern entry typically follows: **intent → running example → code → key points → pitfalls**.
- "Further Reading" sections cite the canonical books in `books/` — keep references consistent across docs.
