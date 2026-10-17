# Microlith

Microlith organizes tasks spanning repositories into projects with independent worktrees.

## Language

**Project**:
A named group of worktrees for one task spanning repositories, containing at most one worktree per repository. Its lifetime extends from task setup through completion and eventual archive or cleanup.
_Avoid_: Workspace, service

**Repository**:
A Git history shared by its checkouts. Its contents may include services, libraries, or frontend applications.
_Avoid_: Service, worktree

**Worktree**:
A checkout of a repository with its own working files. A worktree belongs to at most one project.
_Avoid_: Repository, workspace

**Source checkout**:
An existing checkout used to discover a repository and suggest a creation base. Original/main checkouts serve as sources rather than project members.
_Avoid_: Project worktree

**Tracked worktree**:
A worktree whose identity, location, and resource ownership Microlith remembers, whether or not it currently belongs to a project.
_Avoid_: Project member

**Workspace directory**:
A project's designated directory for worktrees and supporting files. Imported worktrees may reside outside it.
_Avoid_: Project, workspace

**Service**:
A kind of software contained in a repository, rather than a synonym for repository or project membership.
_Avoid_: Repository

**Saved base commit**:
The commit selected as the starting point for reviewing a worktree's task changes. It remains fixed unless explicitly replaced.
_Avoid_: Current base reference, merge base

**Base reference**:
The named Git reference selected as a worktree's comparison target. Its current commit can change independently of the saved base commit.
_Avoid_: Saved base commit

**Archived project**:
A project hidden from the active list whose membership, files, branches, and saved bases remain available for restoration.
_Avoid_: Deleted project

**In-use flag**:
A manually set project restriction preventing archive, member detachment, and worktree removal. It expresses user intent rather than observed activity.
_Avoid_: Agent status, Git worktree lock
