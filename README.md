# PES-VCS Project

## Phase 5: Branching and Checkout
## Phase 5: Branching and Checkout

### Q5.1: Checkout Implementation
A branch in PES-VCS is represented as a file in `.pes/refs/heads/` containing a commit hash. Implementing `pes checkout <branch>` involves the following steps:
1. Check if the branch exists by verifying the file `.pes/refs/heads/<branch>`.
2. Update the `.pes/HEAD` file to point to the new branch:
ref: refs/heads/<branch>
3. Read the commit hash from the branch file.
4. Load the commit object and extract the tree hash.
5. Traverse the tree object recursively to reconstruct the working directory:
- Create directories as needed
- Write file contents from blob objects
6. Replace the current working directory contents with the new snapshot.
This operation is complex because it must safely overwrite files, reconstruct directory structures, and ensure no uncommitted changes are lost.

### Q5.2: Dirty Working Directory Detection
Before performing checkout, the system must ensure that the working directory is clean (no uncommitted changes).
To detect a dirty working directory:
1. For each file tracked in the index:
- Read its stored hash and metadata from the index.
- Read the current version of the file from the working directory.
2. Recompute the file’s hash using the same method as `object_write` (without storing it).
3. Compare the recomputed hash with the stored index hash.
If any file differs:
- The working directory is considered "dirty"
- Checkout must be aborted to prevent data loss
This uses:
- Index → expected state
- Object store → stored content
- Working directory → current state

### Q5.3: Detached HEAD
In normal operation:
HEAD → branch → commit
In detached HEAD state:
HEAD → commit (directly)
This happens when HEAD points directly to a commit instead of a branch.
If a commit is made in this state:
- A new commit is created
- But no branch points to it
- The commit becomes unreachable once HEAD moves away
To recover such commits:
1. Create a new branch pointing to that commit
2. Or manually checkout using the commit hash
Detached HEAD is useful for temporary exploration but can lead to lost commits if not handled carefully.

## Phase 6: Garbage Collection

### Q6.1: Finding Unreachable Objects
Over time, the object store may contain objects that are no longer referenced by any branch.

To identify and remove unreachable objects:

1. Start from all branch heads in `.pes/refs/heads/`
2. Traverse each commit:
   - Mark the commit as reachable
   - Follow its parent pointer recursively
3. For each commit:
   - Mark its tree object
   - Recursively mark all subtrees and blobs
4. Maintain a set (e.g., hash set) of all reachable object hashes
5. Iterate through `.pes/objects/`:
   - Delete objects not present in the reachable set

For a repository with 100,000 commits and 50 branches:
- The traversal visits all commits and associated trees/blobs
- Total objects visited can be several hundred thousand

A hash set is used for efficient lookup of reachable objects.

### Q6.2: Race Condition in GC
Running garbage collection concurrently with commit operations can lead to race conditions.

Example scenario:
1. GC scans and identifies an object as unreachable
2. At the same time, a commit is being created that references this object
3. GC deletes the object before the commit is finalized
4. The new commit now points to a missing object → repository corruption

To prevent this, Git uses:
- Atomic operations (write temp → rename)
- Locking mechanisms to prevent concurrent modification
- Reference checks before deletion
- Grace periods before removing objects

Thus, GC must ensure that objects are not deleted while they are being referenced or created.
