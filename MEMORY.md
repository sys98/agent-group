# Project memory

## Worker isolation decision — 2026-10-20

- codex-orchestration adapts the orca-dispatch worker lifecycle. Each implementation task requires a separate branch, worktree, and independent Codex CLI session; shared-directory native child agents are not an implementation substitute.
- Native child agents are suitable for read-only investigation and Matt Standards/Spec review. Worktree isolation is an orchestration requirement, not a requirement imposed by Matt's individual skills.
- Coordinator duties include explicit baselines, supervised session recovery, fixed-candidate review, serial integration, and safe cleanup. A CLI turn or process completing does not establish product acceptance.
- The published skill resolves installed dependencies through host metadata and real paths; author-specific absolute skill paths must not be distributed.
- README copy, diagrams, invocation metadata, and the installed skill must describe the same implementation model.

## Governance and validation

- The repository had no AGENTS.md or MEMORY.md at the start of this revision. No matching template was available under the configured local template directory. This file records the durable workflow correction; other governance fields remain Unknown.
- Skills and references are Markdown with YAML metadata; README visuals are static SVG. No package manager or application test runner is configured.
- Validate skill metadata with the skill-creator quick validator and README assets with the beautify-github-readme audit. Also inspect commands against the actual Codex CLI help and exercise Git isolation and integration in a disposable repository.
- Isolated Git/process checks and document validation do not prove a real model-backed multi-worker delivery. Report that distinction until runtime acceptance exists.
