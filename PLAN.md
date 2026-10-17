# Microlith

Local desktop app for organizing repositories into task projects backed by Git worktrees.

Each project represents one task spanning repositories, with its own worktrees, and is archived or cleaned up after task completion. Repositories may contain services, libraries, or frontend applications. Use the canonical terms in [GLOSSARY.md](./GLOSSARY.md).

## Scope

- Select source folders anywhere on disk and discover Git repositories.
- Combine repositories from multiple locations into one project.
- Choose a workspace destination for each project.
- Create a branch and worktree for each selected repository.
- Inspect paths, branches, working changes, and commits relative to project bases.
- Open projects in VS Code with generated workspace files.
- Import existing worktrees and manage project cleanup.
- Support agents launched separately, editing workspace files externally.

Agent launching, Codex session integration, agent status, hosted services, and account management are outside the initial scope. Git state cannot establish whether an agent is active; users can manually mark projects as in use.

## Technology

| Area | Choice |
| --- | --- |
| Desktop | Electron |
| UI | React and TypeScript |
| Frontend build | Vite |
| Styling and components | Tailwind CSS and daisyUI |
| Git operations | Installed Git CLI |
| App metadata | SQLite |
| Change detection | Filesystem watchers and periodic Git refresh |

Target Linux first. Keep filesystem, editor launching, and desktop integration portable for later macOS and Windows support. Select compatible dependency versions during implementation.

## Architecture

### Renderer

Project dashboard, project creation flow, repository details, and diff viewer. Access desktop operations through a narrow typed preload API. Keep Node.js access in the desktop backend; enable context isolation and renderer sandboxing.

### Desktop backend

Own repository discovery, Git execution, SQLite persistence, filesystem watching, folder dialogs, and editor launching. Keep slow operations asynchronous. Publish bounded status updates to the renderer.

### Domain services

Separate project lifecycle, repository queries, worktree operations, and editor integration from Electron handlers. Serialize Git mutations by shared repository identity, including operations from different linked worktrees.

Git is authoritative for repository state. SQLite stores project membership, paths, saved base revisions, preferences, and manual usage locks. Reconcile stored metadata with disk on startup and refresh.

## Data model

- **Project:** ID, name, workspace destination, lifecycle state, manual in-use flag, timestamps.
- **Repository:** ID, original checkout path, shared Git directory identity, display name.
- **Worktree resource:** ID, repository ID, path, ownership of created resources, tracking state; persists independently of project membership.
- **Project repository:** project ID, worktree resource ID, observed branch or detached `HEAD`, selected base reference, resolved base commit, private base-retention ref.
- **Preferences:** recent source and destination folders, editor configuration, UI preferences.

Store the resolved base commit so task comparisons remain meaningful if the named base branch advances. Show ahead/behind against the selected reference separately from task changes against the saved commit.

### Confirmed creation and membership rules

- Require confirmation of each repository's base reference, initially suggesting the source checkout's current branch. Show the reference and resolved commit in the preview and save the commit before creation. Never fetch automatically.
- Require explicit base selection for imported worktrees; do not infer where their task history began.
- Suggest one editable branch name derived from the project name across repositories, with per-repository overrides. Validate every branch name and collision before creating resources.
- Let users select a parent folder, then create a dedicated project directory with one subdirectory per repository. Preview editable directory names and reject existing destination paths for new resources. Imported worktrees remain at their existing paths.
- Retain successfully created resources after partial failure, mark creation incomplete, and offer retry or explicit cleanup. Never automatically roll back worktrees that external tools may have edited.
- Allow repositories to be added to existing projects through the same creation or import rules. Allow members to be detached without deleting their worktrees; removal remains a separate explicit action.

### Confirmed review and lifecycle rules

- Task review compares the saved base commit with current `HEAD`. Show staged, unstaged, and untracked changes separately from committed task changes.
- If the saved base stops being an ancestor of `HEAD`, flag divergence and retain it until the user explicitly replaces it. Never silently move the comparison baseline.
- Archive hides a project from the active list while preserving files, branches, membership, and saved bases. Support restore. Stop background monitoring for archived projects and refresh when opened or restored.
- Block worktree removal when staged, unstaged, or untracked files exist, or Git reports the worktree locked. MVP offers no force removal; users resolve blockers externally and retry.
- Commits absent from the selected base reference trigger a prominent warning and explicit acknowledgment before worktree removal, rather than blocking removal. Preserve branches by default.
- A manual in-use flag blocks archive, member detachment, and worktree removal, while allowing inspection and repository additions. Clear it only through explicit user action; it cannot establish whether external tools are active.
- Imported worktrees start unchecked in cleanup previews and require explicit selection. Apply the same removal safeguards; importing never transfers resource ownership.
- Detaching a member retains a resource record with its path and ownership so its worktree remains discoverable for later cleanup.
- MVP preserves branches on every cleanup. Branch deletion remains an external Git operation.
- Block detached worktree removal unless its `HEAD` is reachable from a local branch or tag. Users create a preserving branch or tag externally before retrying removal.
- Retain saved base commits through private Git refs while project records exist, including archived projects. Release those refs on explicit project metadata deletion. See [base retention decision](./docs/adr/0002-retain-saved-bases-with-private-git-refs.md).
- Resource records survive deletion of their former project's metadata until the worktree is removed or the user explicitly stops tracking it. Show resources without project membership in a separate cleanup list.
- Each tracked worktree has one resource record and at most one current project membership. A detached member may be imported into another project without changing resource ownership; the former project's in-use flag no longer governs it.
- Allow explicit replacement of a missing base reference independently of the saved base commit. Replacing the reference restores live comparisons and cleanup checks; replacing the task baseline remains a separate explicit action.
- Persist each mutation's intent before running Git, then reconcile its result after restart. Require explicit user action to resume interrupted creation or cleanup; refresh never resumes mutations. See [recovery decision](./docs/adr/0001-explicit-recovery-of-git-mutations.md).
- Run one backend per app metadata store; subsequent launches focus the existing window. Serialize app-initiated Git mutations by shared repository identity, while accounting for external operations that can race.
- Block cleanup when the selected base reference is missing, Git status fails, or repository storage is unavailable. Show the failed check and allow retry; unverifiable state is not a clean result.
- Immediately before each removal, recheck resource ownership, worktree identity, in-use flags, and Git state. Stop remaining removals when a check fails and keep Git's normal removal protections enabled.
- Cleanup removes only selected worktrees, unchanged app-generated files, and empty app-created directories. Preserve edited context and workspace files and all other contents, report remaining paths, and never recursively delete the project directory.
- After worktree removal, retain project metadata and an operation summary. Allow permanent project metadata deletion separately only when no members or pending operations remain; keep failures and leftover paths visible until resolved or explicitly dismissed.
- Display actual Git state when external tools switch branches or detach `HEAD`; retain the saved base, flag the change, and never switch back automatically. Allow detached worktree import with explicit base selection.

### Confirmed discovery and integration rules

- Discovery defaults to five directory levels, skips dependency and build folders, does not follow directory symlinks, and stops descending once a repository is found. Allow depth and exclusion overrides and explicit selection of skipped paths. Show skipped locations and reasons.
- Ordinary clones serve as discovery and creation sources. Project import accepts linked worktrees only; original/main checkouts remain outside project membership and cleanup. Reject bare repositories and submodules as project members with an explanation before creation or import.
- Import adopts existing worktrees only. To reuse an existing branch, users create its worktree externally and then import it; app creation always uses a fresh branch.
- When a worktree disappears or its path contains a different repository, mark it unavailable and require explicit relinking with identity validation. Never silently adopt replacement contents or relocate files.
- Generate a `.code-workspace` file and `PROJECT.md` describing the task, member paths, branches, and saved bases. Preserve user edits; offer explicit workspace updates after membership changes and require review before overwriting regenerated context.
- Store editor configuration as an executable and separate argument entries, initially using VS Code's `code` command. Pass workspace paths as arguments without shell interpretation; offer configuration and retry after launch failures.
- Load diffs on demand with initial limits of 1 MiB or 10,000 lines per request. Clearly mark truncation and offer an external-editor fallback. Show binary-file metadata without rendering binary contents.
- Refresh active projects on focus, manual request, debounced filesystem events, and a 30-second fallback timer. Prioritize the visible project and avoid overlapping scans. Pause background polling while hidden and reconcile on return.
- Store SQLite in the operating system's per-user app-data directory, independently of workspace files. No cross-device project synchronization in MVP.

## Main screens

1. **Projects:** searchable cards or rows showing repositories, changed files, missing paths, and manual usage state.
2. **Create project:** select repositories, destination, base revisions, and branches; preview paths before creation.
3. **Project details:** repository list with branch, worktree path, change counts, and comparison summary; open project in VS Code.
4. **Repository details:** working and staged changes, untracked files, task diff, and commits.
5. **Cleanup:** preview owned resources, uncommitted changes, and unmerged commits before removing worktrees.

## Git and filesystem rules

- Use argument arrays for subprocess execution; never interpolate user paths into shell commands.
- Parse machine-readable Git output, using NUL delimiters where supported.
- Detect linked worktrees through Git commands; do not assume `.git` is a directory.
- Canonicalize repository identity while preserving useful display paths. Handle spaces, Unicode, symlinks, missing paths, and destination collisions.
- Give concurrent tasks separate worktrees and branches. Validate branch names and existing branch checkout constraints.
- Keep original clones persistent: linked worktrees depend on shared Git storage.
- Do not switch branches, reset files, or remove worktrees as an automatic refresh action.
- Manual in-use locks prevent app cleanup; they cannot detect or stop external agents.
- Remove worktrees through Git. MVP offers no force removal and always preserves branches.
- Track ownership so imported worktrees and preexisting directories are not treated as app-created resources.
- Allow at most one worktree per repository in a project and at most one project per worktree. Separate projects may use the same repository through separate worktrees.
- Leave original checkout edits in place. Create new worktrees from selected base commits without copying, stashing, or moving source edits.
- Normal creation makes a fresh branch for each repository. Resolve branch collisions before creation; handle existing branches and worktrees through explicit import.
- Import existing worktrees at their current paths, including paths outside the workspace directory. Reference those paths in generated VS Code workspace files without moving files.
- Bound discovery depth, skip dependency and build directories, and allow cancellation.
- Watch relevant workspace and Git metadata paths; debounce updates and refresh on window focus. Provide manual refresh and periodic reconciliation.
- Worktrees share Git objects but have separate checked-out files. Dependencies and build outputs consume additional space.

## Implementation phases

All six phases are required for the first release, including existing-worktree import, the in-app diff viewer, crash recovery, and Linux packaging. Implement incrementally without dropping these release requirements.

### 1. Desktop foundation

- Scaffold Electron, React, TypeScript, Vite, Tailwind CSS, and daisyUI.
- Establish typed IPC, SQLite migrations, error reporting, and application layout.
- Add native source and destination folder selection and recent locations.

Completion: app starts locally, remembers preferences, and accepts arbitrary accessible folder paths.

### 2. Repository discovery and import

- Scan selected folders for repositories with progress and cancellation.
- Query branches, repository identities, existing worktrees, and working status.
- Register source repositories and import existing linked worktrees without modifying checked-out files. Import may create private Git refs to retain confirmed saved bases.
- Represent unavailable paths and recover when they become accessible.

Completion: repositories from multiple disk locations appear with accurate paths and Git state.

### 3. Project creation

- Build repository selection, base selection, branch naming, and destination preview.
- Validate all planned operations before creation.
- Create worktrees sequentially with progress reporting and record completed steps.
- Handle partial failures with explicit recovery options; preserve files changed externally.
- Generate a project `.code-workspace` file and an editable project context document.

Completion: one project contains independent worktrees for selected repositories; original checkout branches and files remain intact.

### 4. Project monitoring and review

- Add dashboard and project repository table.
- Show staged, unstaged, and untracked changes plus commit comparisons.
- Build diff views for individual files and whole tasks against saved bases.
- Handle binary files, deleted files, renames, and large diffs with bounded loading.
- Add watchers, periodic refresh, and refresh on focus.
- Open generated workspaces in VS Code; support a configurable editor executable.

Completion: edits made externally appear in app and VS Code; committed task changes remain reviewable.

### 5. Lifecycle and recovery

- Add project rename, manual usage locks, archive, and worktree removal preview.
- Distinguish metadata-only archive from deletion of worktrees.
- Require explicit confirmation for cleanup and flag uncommitted changes and commits absent from the selected base reference.
- Reconcile interrupted creation and cleanup operations after restart.
- Preserve imported resources unless their removal is explicitly selected.

Completion: users can archive or remove projects without silently losing work.

### 6. Release readiness

- Validate Git operations using temporary repositories and real linked worktrees.
- Cover project persistence, IPC boundaries, branch collisions, partial failures, and cleanup protections.
- Exercise folder selection, project creation, diff review, editor opening, and archive flows.
- Check external edits, missing Git/editor executables, unavailable folders, and filenames containing spaces or Unicode.
- Package Linux build and document installation and supported Git requirements.

Run relevant tests before considering implementation complete. Never create commits automatically.

## MVP acceptance criteria

- User selects source locations and project destinations without hardcoded home-directory assumptions.
- Project can group multiple repositories on independent worktree branches.
- External agent edits and commits are visible through Git refresh.
- App displays the current branch and worktree path for every project member.
- Both uncommitted changes and committed task changes can be reviewed.
- Project opens in VS Code with all member worktrees available.
- Cleanup previews risks and respects manual usage locks and resource ownership.
- Project metadata survives restart and reports drift from filesystem state.

## Later possibilities

- Codex integration through an optional adapter.
- Reusable project templates and service context generation.
- Per-project commands and development environment configuration.
- macOS and Windows packaging.
- Support for additional editors.
