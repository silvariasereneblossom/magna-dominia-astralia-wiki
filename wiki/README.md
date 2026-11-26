# Magna Dominia Astralia wiki — conventions

This is the living wiki for **Magna Dominia Astralia**. It lives in the repo (`docs/wiki/`) so docs move with the code in the same commits. The project's
existing docs stay where they are; the wiki links into them and adds overview pages, a doc map, session logs,
and consistency checks. Maintenance follows the protocols in `protocols/`, adapted from the IridescentCraft wiki template.

## Layout

- `home.md` — front door; every page must be reachable from here (Protocol 6)
- `overview.md` — what the project is and how it fits together, in one page
- `design/` — the unified design document: `index.md` plus one page per system, each with the same sections (status, how it works, key decisions, not yet built, discrepancies, sources)
- `engineering/` — how the project is measured and tested: performance goals and caveats, and the testing and benchmarking process
- `map.md` — generated section map of the project's own docs (`jeverifier wiki map`)
- `sessions/` — `index.md` plus one Jev-triaged log per session (Protocol 5)
- `checks/` — review verdicts, the accepted baseline (`wiki check`/`accept`), and `lint.json`: semantic code rules Jev checks per function (`jeverifier wiki lint`)
- `meta/` — style guide and how the AI-maintained wiki works
- `protocols/` — maintenance workflows, invoked as "run Protocol N"

## Conventions

- Relative Markdown links to `.md` files, with GitHub heading anchors for sections.
- The code and the project's check scripts are the source of truth; the wiki describes them.
- Pages end with a **Related pages** section.
- Commits follow the host repo's rules: authored by the owner only, no attribution trailer; push only when asked.
- Verification command for this project: `scripts/gate-remote.ps1`.

## GitHub Wiki mirror

`.github/workflows/sync-wiki.yml` mirrors this folder to the repository's GitHub Wiki on every push to `main` that touches it. `.github/scripts/wiki_sync.py` flattens the folders into page names (`design/combat.md` becomes `Design-Combat`), rewrites links, skips pages marked `<!-- INTERNAL ONLY -->` (session logs), and generates `Design-Document` (all design pages as one document) and the sidebar. The same workflow also builds a public copy (`--public .github/wiki-public.json`: links into this private repo become plain labels, the test host's name and paths are replaced, and the build refuses to publish a text on the deny list) and pushes it to the public repository `magna-dominia-astralia-wiki`, whose own Action mirrors it to that repository's Wiki. The Wiki is read-only output: edit the files here. Preview locally with `python .github/scripts/wiki_sync.py <some empty folder>`.

## Protocols

| # | Protocol | File |
|---|---|---|
| 1 | Harmonize: fix contradictions, anachronisms, tone | [1-harmonize.md](Protocols-1-harmonize) |
| 2 | Spot-check: Jev-ranked review list + sampled pairs, verdicts recorded | [2-spot-check.md](Protocols-2-spot-check) |
| 3 | Verify a claim against code, data and checks | [3-verify.md](Protocols-3-verify) |
| 4 | Recursive harmonize until nothing changes | [4-recursive-harmonize.md](Protocols-4-recursive-harmonize) |
| 5 | Session log from a Jev digest | [5-session-log.md](Protocols-5-session-log) |
| 6 | Homepage coverage: orphans and broken links | [6-homepage-coverage.md](Protocols-6-homepage-coverage) |
| 7 | Commit & ship | [7-commit-and-ship.md](Protocols-7-commit-and-ship) |
