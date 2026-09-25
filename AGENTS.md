# Working in this repository

Always use an isolated Git worktree and its own `codex/` branch for repository changes, including code, documentation, and configuration. The primary checkout stays on `main` and is the integration target. This policy applies to this repository only. Local commits are authorized. Never merge into `main` or run the integration helper until the user explicitly authorizes that merge. Completing implementation or choosing a variation does not authorize a merge. Pushing, opening pull requests, and deploying require a separate request.

## Working style and variations

- Focus on speed and efficiency. Keep changes scoped, reuse existing code and assets, batch independent reads, and avoid unnecessary setup, repeated checks, and unrelated refactoring.
- Whenever the user asks for variations or alternatives, create exactly **10 distinct variations in a new HTML file** in the task worktree. Give each option a clear number and label so the user can compare and choose. Preserve previous variation files.
- Show the HTML comparison and wait for the user to choose an option before implementing it in the game. Do not select or implement a winner on the user's behalf.
- Always show screenshots of your changes in the response. Capture the affected UI or variation page; for documentation or configuration changes, capture the updated content or diff. Embed the screenshots so the user can see them directly.
- Whenever you share a preview URL, include both a clickable **local URL** and a clickable **network URL** for the same page and port. Start the server with network access, use the machine's actual LAN address, and preserve the same path and query in both links. If a network URL cannot be made available, explain the blocker instead of inventing one.

## Before editing

Use Node.js 22 or newer. The repository includes `.nvmrc`; run `nvm use` if the shell has selected an older Node version. In a shell without nvm loaded, prefix commands with `bash scripts/node22.sh` (for example, `bash scripts/node22.sh node scripts/worktree.mjs start --task <task-id> --name <short-name>`). Codex setup and actions use this launcher automatically.

1. Run `node scripts/worktree.mjs start --task <task-id> --name <short-task-name>` from the current checkout. Use `CODEX_THREAD_ID` as the stable task ID when available (the helper reads it automatically). Otherwise use this task's ID or choose one stable unique ID and retain it for follow-ups.
2. Read the returned `path` and use that directory as the working directory for **every** edit, command, test, and commit. Never continue implementation in the primary checkout. This also applies when the app composer was accidentally left in Local mode.
3. Install dependencies with `npm ci` in the task worktree when needed to run the app or prepare an authorized merge, unless the Codex environment setup has already installed them. Skip unnecessary installs for documentation-only edits. Do not share `node_modules` between worktrees.

The helper reuses this task's registered worktree and adopts a new Codex-managed worktree, creating a branch if its HEAD is detached. It refuses to adopt another task's registered directory. When a completed task receives a follow-up, run `start` again before editing; it creates a fresh branch from current local `main` in the retained task directory.

For planning, explanations, and read-only reviews, no worktree or merge is necessary. Respect Plan mode and any user instruction to leave work uncommitted or unmerged.

## Development and previews

- `npm run dev` starts the game with the port reserved for this task and prints both local and network preview URLs. The primary checkout uses port 5182. A deliberate `PORT` environment variable overrides the reservation.
- Defer automated tests and build validation until preparing an explicitly authorized merge into `main`, unless the user asks for them sooner. During iteration, do only the lightweight inspection needed to review the change and capture screenshots; do not run the full suite on each change.
- Keep edits focused on the requested task. Never stash, discard, reset, or commit another task's work.

## Completing implementation without merging

1. Review the diff and commit only this task's completed changes in its worktree. The checkout must be clean, including non-ignored untracked files.
2. Report the task branch and commit, summarize the changes, and show screenshots. Include both local and network URLs whenever sharing a preview. State that tests/build validation are deferred and the changes remain unmerged when applicable.
3. Keep the task worktree for follow-ups and wait for an explicit merge instruction. Implementation can be complete on the task branch; do not run `finish` just to close out a task.

## Merging only when the user authorizes it

1. Once the user explicitly authorizes the merge, run `node scripts/worktree.mjs finish --task <same-task-id>` from the task worktree. This command integrates into `main`; it is not a routine completion command.
2. The helper serializes integration, merges against the latest local `main` in a temporary worktree, installs dependencies, runs `npm test` and `npm run build`, and fast-forwards the clean main checkout to that exact validated merge commit. It never pushes. Run any additional checks appropriate to the change at this stage; do not duplicate the helper's checks unnecessarily.
3. If a merge conflicts, use the integration path in the error. Resolve only conflicts whose intended behavior is clear, commit the resolution there, and rerun `finish` from the task worktree. If fixing the source task instead, commit there and rerun `finish`; the helper builds a fresh integration checkout.
4. If main or the task advances during validation, rerun `finish` against the new state. If another merge holds the lock, wait for it and retry. If main is dirty, preserve those edits and report the blocker; never clean main automatically. Report ambiguous conflicts or failing checks with the retained branch and integration path.
5. On success, report the task branch, merge commit, and validation results. Keep the task worktree for follow-ups. Never delete a running task's worktree or force-remove failed integration worktrees.

Use `node scripts/worktree.mjs status` to inspect registered tasks, ports, pending integrations, and the repository lock. A lock left by a terminated process must be investigated; the helper never steals it based on age.
