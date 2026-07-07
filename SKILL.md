---
name: sasuke
description: >
  Sasuke (佐助寫輪眼) — a data-readiness steward for agent systems. Keeps every raw file
  in your knowledge vault (epubs, PDFs, catalogs, markdown) converted to agent-readable
  form BEFORE it is needed, with gaps triaged by priority and ready-to-run dispatch
  commands. Registry-driven weekly sweep + deny-by-default global census. Use when the
  user says "sasuke", "data readiness", "sweep the vault", "what's not converted yet",
  or wants new data pipelines brought under readiness management.
metadata:
  category: meta-infra
---

# Sasuke 佐助寫輪眼 — Data-Readiness Steward

> Design motto: **兵馬未動，糧草先行** — provisions move before the army.
> Raw material is converted *before* it's needed; the student/agent never waits.
> Born from a real incident: opening a book course took 11 minutes of on-demand
> conversion — the student had already left.

**One sentence**: an all-seeing eye over your vault — every raw file is either
agent-readable, queued with a priority and a dispatch command, or deliberately excluded
by an auditable policy. Nothing is invisible.

## The five layers

| Layer | Responsibility |
|---|---|
| L0 Source-point sync | Every ingestion pipeline converts at output time (e.g. book translator emits per-chapter MD); the sweep is the safety net, not the workhorse |
| L1 Registry | Contract per data class: raw globs / readiness check / converter (with `llm: true/false` flag) / priority / converter version |
| L2 Sweep engine | Weekly scheduled + on-demand; zero-LLM conversions auto-heal, drift re-conversion, self-test gate |
| L3 Gap dispatch | LLM-needed gaps are not just listed — they are priority-ranked with ready-to-paste dispatch commands |
| L4 Observability | Single status dashboard; heal-first-report-after; only failures raise alerts |

## Iron rules

1. **Zero-LLM conversions run automatically; anything that burns metered LLM is report-only** until the owner approves (subscription-based RAG services can get a weekly quota).
2. **Every new database/pipeline delivery must register in the registry** — an unregistered pipeline is an incomplete delivery.
3. **Self-test gate**: after any converter code change, a fixture golden test must pass before the sweep may touch the library (prevents silently corrupting hundreds of converted files at once). Fault-injection verified.
4. **Drift re-conversion**: source newer than output → auto re-convert. Converter version bump → "re-convertible" report only, never auto mass-rebuild.
5. **Boundaries**: the steward owns "raw file → readable form" + search-index freshness only. Service health belongs to your pipeline monitor; content-source DB freshness belongs to each source's own cron.

## Global census layer (v2.1 — red-team hardened design)

The registry is a whitelist: it only lights up corners you registered. The census walks
the *entire* vault, groups every non-markdown file by (directory × extension), and runs
a **census → policy → triage** loop. Dual cross-vendor red teams failed the naive
version of this design; the hardened rules:

1. **Policies are auditable, not a second whitelist**: every policy carries
   `owner / reason / created_at / last_seen_at / review_after / sample_paths / risk_level`.
   New groups default to `triage_blocked` — held and reported, never silently ignored,
   never auto-excluded (auto-exclude is how blind spots quietly return).
2. **Sidecars are contained**: extracted markdown lives in one dedicated sidecar
   directory (never scattered next to originals), with full provenance frontmatter
   (`source_path / source_sha256 / source_mtime / converter / converter_version /
   policy_id / generated_at`). Quality filter: outputs <100 bytes or >10× source size
   are quarantined as suspicious.
3. **Index admission gate**: the search index only accepts allowlisted sidecar paths.
   Before auto-converting, estimate output size / word count / file count / duplication;
   over threshold → `needs_approval`. Raw-source corpora (textbooks, per-chapter
   originals, data files) are **mechanically denied** from the index — "find what's
   missing" and "don't flood the index" are two faces of the same coin.
4. **Fail loudly when untriaged**: metrics `untriaged_groups_count`,
   `oldest_untriaged_days`, `blocked_files_count`; >14 days untriaged flips status to
   *failing*, and the status command must surface it — success counts alone are banned.
5. **Three-layer fuses**: caps on output MB, sidecars per policy, and new index docs
   per run — not just a total action cap. Per-policy circuit breaker (one failing
   policy doesn't halt the others); a converter failing >5 files in one run is
   temporarily disabled with an alert.
6. **Drift by content hash**: mtime is only a cache hint — real drift detection uses
   `source_sha256` (junctions and sync tools scramble mtimes; mtime-only causes mass
   false re-conversions).
7. **Tombstones**: when a source file disappears, its sidecar is tombstoned and removed
   from index candidates next run; the report lists "source missing but sidecar alive".
8. **Pathological fixture set**: the self-test covers malformed spines, missing OPF,
   duplicate hrefs, CJK filenames, giant chapters, broken encodings, >260-char paths,
   junction double-paths, mixed encodings. Startup checks OS long-path support and
   aborts loudly if absent.

**The failure mode this is all designed against** (red-team consensus): a year from
now the system dies not from broken code but from *people no longer trusting the
reports* — triage piles up, policies have no owner, sidecars sprawl, the index returns
duplicates. Deny-by-default, contained sidecars, auditable policy, admission gates, and
loud failure are the countermeasures.

## Converter routing matrix

| Format | First choice (local, zero-LLM) | Fallback |
|---|---|---|
| epub | OPF-spine chapter splitter (see companion repo `shanya-shuyuan`) | pandoc |
| Text PDF | pdftotext | PyMuPDF |
| Scanned PDF / images | — | NotebookLM ingest → `source fulltext` pull-back (Google does the OCR; subscription, zero marginal cost) |
| docx/pptx/xlsx | stdlib zip+xml / pandoc | MarkItDown |
| Audio / video | local whisper ASR (**confidential material must stay local**) | NotebookLM ingest → fulltext (non-confidential only) |
| Web pages | your fetcher skill | headless browser |
| Charts needing *understanding* | — | LLM vision (expensive, on demand) |

Two routing principles: mechanically parseable → local zero-cost (fast, faithful,
offline); understanding-type parsing (OCR/ASR/vision) → subscription RAG service as
outsourced labor, **never uploading confidential material**.

## Trigger table

| User says | Action |
|---|---|
| "sweep the vault" / "sasuke patrol" | run the sweep, summarize actions/gaps/failures in one paragraph |
| "vault status" | read the status dashboard + scheduler state; must surface *failing* states |
| "fill capsules N" | take top-N gaps by priority → run the RAG capsule pipeline → report |
| "register new database" | add a registry entry (ask for raw location / readiness check / converter / llm flag) |
| "self-test" | run the fixture golden tests + dry-run, paste results |

## Hard-won pitfalls

- Junction/symlink double-paths: always dedupe by `realpath` or you convert everything twice.
- Glob patterns must anticipate filename variants (`*_bilingual*.epub`, not `*_bilingual.epub`).
- A daily "gaps exist" report is a feature, not noise — the backlog staying visible is
  the design; clear the backlog rather than silencing the report.
- Windows: schedule via PowerShell `Register-ScheduledTask` (schtasks quoting breaks in
  git-bash); batch files need CRLF; pin full python paths to avoid store stubs.
