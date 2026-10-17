# Retain saved base commits with private Git refs

Task comparisons must survive branch movement and rewritten history, but a commit hash stored in SQLite does not keep its Git objects reachable. Retain each saved base through an app-owned private Git ref while its project record exists, including archived projects, and release it on explicit metadata deletion; this deliberately modifies repository metadata and retains objects rather than relying on reflog expiration or garbage-collection timing.
