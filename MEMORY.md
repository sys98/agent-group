# Project memory

## App task visibility and lifecycle — 2026-10-20

- codex-orchestration adapts the orca-dispatch worker lifecycle. New runs use explicitly user-requested Codex App chats with separate branches and managed worktrees. A worktree attached to the coordinator plus codex exec is insufficient: it does not establish a visible, enterable, continuable App task.
- App chat creation, coordination messages and lifecycle operations follow direct human authorization and host restrictions. A skill invocation or another agent's message does not itself grant those permissions.
- Existing CLI worker attempts retain their frozen contracts, sessions, worktrees and supervision; a skill update must not migrate, duplicate, interrupt or reclaim them.
- Native child agents are suitable for read-only investigation and Matt Standards/Spec review. Worktree isolation is an orchestration requirement, not a requirement imposed by Matt's individual skills.
- Coordinator duties include explicit baselines, same-task continuation/recovery, fixed-candidate review, serial integration, and safe cleanup. Chat/turn completion does not establish product acceptance or resource reclamation.
- New tasks initialize read-only before implementation. A pending clientThreadId cannot be used as a real threadId; an unknown creation result must not trigger a duplicate create.
- Chat archival, managed worktree reclamation and safe branch deletion have separate evidence. Preserve unmerged/dirty/running, manually taken-over, pinned/shared/permanent and unrelated work as applicable. Do not steal another chat's worktree ownership or raw-delete App-managed trees.
- Resume the same App thread and restore through its owning context; restoration can change the path, branch state and candidate SHA, invalidating previous review evidence.
- The published skill resolves installed dependencies through host metadata and real paths; author-specific absolute skill paths must not be distributed.
- README copy, diagrams, invocation metadata, and the installed skill must describe the same implementation model.

## Governance and validation

- The repository had no AGENTS.md or MEMORY.md at the start of this revision. No matching template was available under the configured local template directory. This file records the durable workflow correction; other governance fields remain Unknown.
- Skills and references are Markdown with YAML metadata; README visuals are static SVG. No package manager or application test runner is configured.
- Validate skill metadata with the skill-creator quick validator and README assets with the beautify-github-readme audit. Check App routing/lifecycle against actual tool contracts; CLI help applies only to the frozen legacy transport. Disposable Git fixtures cover isolation and integration mechanics.
- Schema checks, isolated fixtures and document validation do not prove live App task creation, continuation, archival or restoration. Report those distinct acceptance boundaries.

## Concurrency planning correction — 2026-10-21

- Before dispatch, the coordinator must classify independently verifiable slices, real data/contract dependencies, owned write paths, shared-resource conflicts, and a justified concurrency budget. Single-worker execution needs a concrete reason when multiple goals exist.
- Where multiple independent, authorized and isolatable tasks are ready and capacity allows, dispatch multiple workers. Do not process one task's complete create/initialize/implement/wait lifecycle before creating the next independent task.
- Serialize only the conflicting resource stages and integration; unrelated implementation remains concurrent. Shared worktrees, invented dependencies or layer-only slicing are not valid shortcuts to greater concurrency.
- Track planned concurrency, initialization, active implementation, blocked states and actual overlap separately. Several created chats do not prove parallel implementation.
