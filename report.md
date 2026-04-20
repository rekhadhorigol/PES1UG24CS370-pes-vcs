Q5.1 — How would pes checkout <branch> work?
Ans: To implement pes checkout <branch>, three things must happen. First, .pes/HEAD is updated to contain ref: refs/heads/<branch>. Second, the target branch file (.pes/refs/heads/<branch>) is read to get the commit hash, and that commit's tree is loaded to get the full file snapshot. Third, the working directory is updated to match — files that exist in the new branch but not the current one are created, files that no longer exist are deleted, and files that differ are overwritten.
This operation is complex because it must surgically modify the working directory without losing data. It needs to compare the current HEAD's tree against the target tree entry by entry, and apply only the necessary changes.

Q5.2 — Detecting a "dirty working directory" conflict
Ans: To detect a dirty working directory, compare each index entry's recorded mtime and size against the actual file on disk using stat(). If they differ, the file has been modified since it was last staged. Next, check if that same file differs between the current branch's tree and the target branch's tree by comparing their blob hashes. If both conditions are true — the file is locally modified AND the branches disagree on its content — checkout must refuse to proceed, because continuing would silently destroy the user's unsaved changes.

Q5.3 — Detached HEAD
Ans: "Detached HEAD" occurs when .pes/HEAD contains a raw commit hash instead of a branch reference like ref: refs/heads/main. Commits made in this state are written to the object store normally, but no branch pointer is updated to track them. This means once you switch away, those commits become unreachable through normal navigation — they exist on disk but nothing points to them.
To recover, the user must remember the commit hash (visible in the terminal output after each commit). They can then run pes checkout -b new-branch <hash> to create a new branch pointing to that commit, making the chain reachable again. Without this, the commits will eventually be deleted by garbage collection.

Q6.1 — Garbage Collection Algorithm
Ans: The algorithm works in two phases: mark and sweep. In the mark phase, start from every branch in .pes/refs/heads/ and HEAD. For each branch, read the commit it points to, add its hash to a "reachable" set (implemented as a hash set for O(1) lookups), then follow the parent pointer to the previous commit, repeating until reaching the root commit. For each commit visited, also traverse its tree recursively — adding every tree and blob hash to the reachable set.
In the sweep phase, list every file under .pes/objects/ and delete any whose hash is not in the reachable set.
For a repository with 100,000 commits and 50 branches, assuming an average of 20 objects per commit (blobs + trees), you would visit roughly 2,000,000 objects in the mark phase. A hash set handles this efficiently in O(n) time.

Q6.2 — Race Condition between GC and Commit
Ans: Consider this race condition: a commit operation writes a new blob and tree to the object store, but has not yet called head_update() to move the branch pointer. At this exact moment, GC runs, scans all branches, does not see the new objects (nothing points to them yet), and deletes them. When the commit operation then tries to finalize by writing the commit object and updating HEAD, it now references objects that no longer exist — causing silent corruption.
Git avoids this using a "grace period" strategy: GC never deletes objects newer than 2 weeks old, regardless of reachability. This gives any in-progress operations plenty of time to complete and make the objects reachable before GC could touch them. Git also uses lock files to prevent concurrent writes to the same refs.


