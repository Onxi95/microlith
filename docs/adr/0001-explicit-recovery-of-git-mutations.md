# Persist Git mutation intent and recover explicitly

Git mutations and SQLite updates cannot commit atomically, and external tools may edit a worktree before interrupted creation or cleanup resumes. Persist each mutation's intent before invoking Git, reconcile recorded intent with repository state after restart, and require explicit user action to resume; retain completed resources rather than automatically rolling back or replaying operations that may now affect external work.
