# Worktrees and automatic local integration

Each coding task owns a worktree and a `codex/<name>-<id>-<suffix>` branch. Completed changes merge automatically into **local main** after tests and build validation. The helper never pushes or starts the Pages deployment.

In the Codex new-task composer, select this repository, **Worktree**, and starting branch **main**. Codex remembers the execution-mode selection for this repository. Select the Hallowmere local environment to install dependencies automatically. This app preference is separate from the repository configuration; `AGENTS.md` also instructs tasks started in Local mode to move their actual work to an isolated directory.

## Start a task

Use Node.js 22 or newer (`nvm use` reads the checked-in `.nvmrc`). Codex setup and actions use `bash scripts/node22.sh` to select the installed nvm version when needed and check the runtime. You can prefix terminal commands with that launcher too. It does not download a new Node version automatically.

```sh
npm run worktree -- start --task example-task-id --name inventory-polish
```

Use the returned `path` as your working directory, then run `npm ci` there. `--task` may be omitted when `CODEX_THREAD_ID` is set. Reuse the same task ID for the task's lifetime. Calling start again returns the same directory; after a successful completion, it starts a new branch from current main for follow-up work. New tasks must use different IDs.

App-created worktrees are adopted where they are; detached HEADs receive a task branch. Tasks started in the primary checkout receive a worktree under its ignored `.worktrees/` directory. Both forms use the same completion workflow. Existing files are not copied from a dirty main checkout.

```sh
npm run dev
```

Each registered worktree has a reserved port from 5300–6299. Reservations are serialized and remain stable for follow-ups. Main retains port 5182. `PORT` can override the reservation if another unrelated application occupies a port. Run the server in only one terminal per worktree. It watches files and prints both local and network URLs.

## Finish a task

Review and commit your task changes, then run this **from its worktree**:

```sh
npm run worktree -- finish --task example-task-id
```

The helper takes a repository-wide lock, prepares a merge commit in a separate integration worktree, and runs `npm ci`, `npm test`, and `npm run build`. Only the exact tested commit can advance a clean main checkout. It checks again that neither source branch nor main changed during validation. Task branches and directories remain available afterward. Repeating finish without new commits is a no-op.

Planning, unanswered questions, canceled tasks, and unfinished work do not count as completion. Repository instructions make the agent invoke finish at implementation completion; there is no turn-ended hook or background process guessing whether code is ready.

## When integration is blocked

- **Dirty main:** leave its files untouched; let their owner finish before retrying.
- **Conflicts:** the error includes the retained integration directory. Resolve clear conflicts there, commit the result, and run finish again from the task worktree. Never choose an entire side simply to make the merge pass.
- **Failed checks:** fix and commit the task branch, or fix the retained integration commit, then retry. Main stays unchanged.
- **Main or task advanced:** retry. A fresh integration checkout is built against the new commits. Old failed integration directories are retained in status for inspection.
- **Lock busy:** another operation is integrating. The helper waits up to 60 seconds, then reports the lock path. Retry once the operation finishes. If a process crashed, inspect the lock's PID and confirm that process is no longer running and no integration is active before manually removing that specific lock file. Locks are never expired or stolen automatically.

```sh
npm run worktree -- status
```

Task metadata and the lock live in `codex-workflow/` under Git's common directory, so all linked worktrees share them. Successful temporary integration checkouts are removed without forcing; failed ones remain. Retained task worktrees can be removed with ordinary `git worktree remove <path>` after confirming they are clean and no task needs them. Start deliberately reports missing registered worktrees instead of silently recreating deleted work.

The workflow is repository-local. It does not change global Codex permissions, other projects, GitHub branch rules, or remote branches.
