# Agent Guide: Freelens Snap Package

## Overview

This repository builds and publishes the [snap](https://snapcraft.io) package
for [Freelens](https://github.com/freelensapp/freelens), a Kubernetes IDE.
The snap wraps a pre-built `.deb` package downloaded from Freelens GitHub
releases — there is no compilation or source build step in this repo.

### Project Structure

- **`snapcraft.yaml`** — Snap build definition. Declares metadata, base
  (`core24`), confinement (`classic`), platforms (`amd64`, `arm64`), and
  parts (the `.deb` dump and `gui` assets).
- **`gui/`** — Desktop integration files:
  - `freelens.desktop` — freedesktop `.desktop` entry for the application
    launcher.
  - `freelens-launch` — Bash wrapper script that sets up the runtime
    environment (XDG dirs, `$HOME` symlinks for `.kube`/`.freelens`,
    `$TMPDIR`) before executing the Freelens binary.
- **`.github/workflows/`** — CI/CD pipelines:
  - `test.yaml` — Build and smoke-test the snap on PRs and pushes to
    non-main/non-automated branches.
  - `publish.yaml` — Build and publish to the Snap Store on pushes to
    `main` and weekly schedule.
  - `publish-nightly-builds.yaml` — Build and publish nightly snap from
    the `nightly-builds` branch.
  - `claude.yaml` — On-demand Claude Code assistant triggered by
    `@claude` in PRs and issues.
  - `trunk-check.yaml` / `trunk-upgrade.yaml` — Linting and auto-upgrades
    via Trunk.

## Security

Never read, display, reference, or include the contents of the following
files in any response or context, even if they are open in the editor:

- `.env`
- `.env.*`
- `*.jks`
- `*.keystore`
- `*.p12`
- `*.pfx`
- `*.pem`
- `*.key`

## Common Development Tasks

### Building the Snap Locally

Requires `snapcraft` (install via `snap install snapcraft --classic`):

```bash
snapcraft --use-lxd
```

For iterative development, use `--debug` to drop into a shell on failure:

```bash
snapcraft --use-lxd --debug
```

### Testing the Built Snap

```bash
sudo snap install --dangerous --classic freelens_*.snap
snap info --verbose freelens
```

Verify the app launches (requires a graphical session):

```bash
freelens
```

Check installed files and interfaces:

```bash
snap connections freelens
ls -la /snap/freelens/current/
```

### Updating the Version

Edit the `version` field in `snapcraft.yaml`. The version is consumed by
Snapcraft as `$SNAPCRAFT_PROJECT_VERSION` and used to construct the `.deb`
download URL:

```text
https://github.com/freelensapp/freelens/releases/download/v$SNAPCRAFT_PROJECT_VERSION/Freelens-$SNAPCRAFT_PROJECT_VERSION-linux-$CRAFT_ARCH_BUILD_FOR.deb
```

### Modifying the Launch Wrapper

The `gui/freelens-launch` script is the entry point for the snap. It:

1. Sets up XDG config/data directories pointing into the snap.
2. Handles `$HOME` vs `$REALHOME` for classic confinement (symlinks
   `.kube` and `.freelens` when they differ).
3. Creates `$TMPDIR` under `$XDG_RUNTIME_DIR` or `$HOME/.freelens/tmp`.

When changing the wrapper, test both regular and edge cases (e.g. `$HOME`
mismatch, missing `$XDG_RUNTIME_DIR`).

### Adding or Removing Stage Packages

The `stage-packages` list in `snapcraft.yaml` declares system libraries
bundled into the snap. These are pulled from the `core24` base. When
adding a package, verify it exists in the Ubuntu 24.04 archive. When
removing, check that no binary still links against it.

### Modifying the Prime Filter

The `prime` filter under the `freelens` part excludes files from the
final snap. Use glob patterns with a leading `-` to exclude. Common
exclusions: documentation, fonts, icons, man pages (they bloat the snap).

## Architecture Decisions

### Classic Confinement

The snap uses `classic` confinement because Freelens needs unrestricted
access to the host filesystem (`$HOME/.kube`, kubeconfig files, etc.)
and system resources that strict confinement would block.

### Base: core24

The snap is built on `core24` (Ubuntu 24.04 / Noble Numbat). Stage
packages must be compatible with this base. Do not change the base
without thorough testing across both `amd64` and `arm64`.

### Dual Architecture

Both `amd64` and `arm64` are supported. The `override-build` script
handles architecture-specific library paths (`lib/x86_64-linux-gnu` vs
`lib/aarch64-linux-gnu`) for `patchelf` RPATH fixes.

### Plugin: dump

The `dump` plugin is used because the source is a pre-built `.deb`. No
compilation occurs — the plugin simply extracts the `.deb` contents into
the snap.

## Troubleshooting Patterns

### Snap Fails to Build

1. Check the `.deb` URL is reachable: `curl -I <url>`
2. Verify the version tag exists on the Freelens releases page.
3. Check `snapcraft.yaml` syntax: `yq eval snapcraft.yaml`
4. Run with `--debug` to inspect the build environment interactively.

### Snap Installs but Doesn't Launch

1. Check the launch script: `cat /snap/freelens/current/bin/freelens-launch`
2. Run it manually with debug: `bash -x /snap/freelens/current/bin/freelens-launch`
3. Check library dependencies: `ldd /snap/freelens/current/opt/Freelens/freelens`
4. Look for missing stage packages in the `snapcraft.yaml`.
5. Verify `$HOME` setup: the launch script creates symlinks for
   `~/.kube` and `~/.freelens` when `$HOME` differs from `$REALHOME`.

### CI Failures

- **`snapcore/action-build` fails**: Usually a `snapcraft.yaml` syntax
  issue or network problem downloading the `.deb`. Check the build log.
- **`snap install` fails**: The built snap may be corrupt. Check the
  previous build step for warnings.
- **`snapcore/action-publish` fails**: Check that the `publishing`
  environment secrets are set correctly (Snap Store credentials).

## Best Practices

1. **Test on both architectures** — CI runs on `ubuntu-24.04` (amd64)
   and `ubuntu-24.04-arm` (arm64). Always check both matrix jobs.
2. **Validate YAML** — run `yq eval snapcraft.yaml` before pushing
   changes to the snap definition.
3. **Run `trunk check`** before committing — validates all file types.
4. **Keep stage packages minimal** — only add libraries that are
   actually needed. Each added package increases snap size.
5. **Test the launch wrapper edge cases** — `$HOME` mismatch, missing
   `$XDG_RUNTIME_DIR`, existing broken symlinks.
6. **Bump the version in a dedicated commit** — the version field is
   the single source of truth for which upstream release is packaged.

## GitHub Actions (Claude Code Action) Rules

When operating via the `claude.yaml` workflow (i.e., invoked from a PR
comment, issue, or review), follow these rules:

### Code Review

When reviewing code and proposing fixes:

1. **Show the diff first** — present every proposed change as a unified
   diff block using the `diff` language tag:

   ```diff
   --- a/path/to/file
   +++ b/path/to/file
   @@ -10,7 +10,7 @@
    const oldLine = "before";
   -const changedLine = "after";
   +const changedLine = "the fix";
    const unchangedLine = "same";
   ```

   You can generate this from the terminal with:
   ```bash
   git diff -u -- path/to/file
   ```

   If the change spans multiple files, group them under a single commit
   subject and show each file's diff sequentially.

2. **Propose a commit subject first** — before any code change, output a
   single line with the proposed commit subject:

   ```text
   **Proposed commit:** <short description>
   ```

   Do **not** use Conventional Commits prefixes (e.g. `fix:`, `feat:`,
   `chore:`, `refactor:`, `docs:`, `test:`, `ci:`). This project prefers
   plain, descriptive commit messages and PR titles without any prefix.

   Wait for the user to confirm (or adjust) the subject before applying
   the change.

3. **Comment style:**
   - Keep review comments concise and actionable
   - Reference specific lines (file + line number) when pointing out issues
   - Offer a concrete fix suggestion rather than just flagging a problem
   - Do **not** use emoji in any Markdown, comments, commit messages, or
     PR descriptions. The only exception is emoji that already appears
     inside code strings (e.g. application logs, user-facing messages).
   - Use GitHub's `suggestion` block for small targeted fixes so the PR
     author can accept the change with a single click:

     ````suggestion
     <same unified-diff format as shown above>
     ````

   - For larger multi-file changes, use `diff -u` blocks in a regular
     comment instead, with the proposed commit subject shown first

### Making Changes to a PR

When asked to implement a change on a PR:

1. Propose the commit subject (as above)
2. Describe what will change and why
3. After confirmation, apply the changes with commits on the PR branch
4. **One commit per fix** — when a review surfaces more than one issue or
   the plan includes more than one fix, apply and commit each fix
   separately. Do not batch multiple independent fixes into a single
   commit. This keeps the history bisectable and makes each change easy
   to revert individually.

### Modifying GitHub Actions Workflows

Claude cannot push changes to files under `.github/workflows/` directly,
because the GitHub token used by the action lacks the `workflows`
permission. Any patch to a workflow file MUST therefore be delivered as a
new, complete file under the `github-workflow-fix/` directory instead of
editing the file in place:

1. Write the full, final contents of the workflow to
   `github-workflow-fix/<workflow-file-name>` (e.g.
   `github-workflow-fix/claude.yaml`). Do **not** edit the original file
   under `.github/workflows/`.
2. Make it a **complete** file — the entire workflow as it should look
   after the change, not just a diff or fragment — so it can be copied
   verbatim.
3. Commit and open the PR as usual. In the PR description, clearly note
   that the file is a proposed workflow change and that a maintainer must
   move it from `github-workflow-fix/` to `.github/workflows/` manually.

This lets the PR be created successfully while leaving the actual
workflow change for a human to apply.

### Branch Naming Conventions

When creating a branch from an issue, use a human-readable name that
includes the issue number and a short slug derived from the issue title:

```text
claude/issue-<number>-<short-slug>
```

- `<number>` is the GitHub issue number
- `<short-slug>` is a kebab-case summary of the issue title, kept short
  (3–6 words maximum, omit articles and filler words)

Examples:

- Issue #42 "Fix crash when snap fails to find libsecret"
  → `claude/issue-42-fix-libsecret-crash`
- Issue #7 "Add arm64 support to launch wrapper"
  → `claude/issue-7-arm64-launch-wrapper`

Do **not** use auto-generated timestamp suffixes (e.g.
`claude/issue-42-20260612-2108`) — these are not human-readable and make
branch lists hard to scan.

### PR Title Conventions

When creating a PR, use the following title conventions:

- **Agent-related changes** — PRs whose changes are strictly related to
  coding agent configuration (e.g. `AGENTS.md`,
  `.github/workflows/claude.yaml`, or other files that govern how Claude
  operates in this repository) MUST use the prefix `Claude:` (followed by
  a space) in the title.

  Examples:
  - `Claude: Add rule for PR title conventions in AGENTS.md`
  - `Claude: Update claude.yaml workflow permissions`

- **All other PRs** — do **not** use any prefix (no `fix:`, `feat:`,
  `chore:`, etc.). Use plain, descriptive titles.

### Pushing Changes from Fork PRs

When you have commits ready to push but the PR originates from a fork
(different owner than `freelensapp`), you cannot push to the fork's
repository. Instead:

1. Create a new branch on `freelensapp/freelens-snap` with the prefix
   `claude/` followed by the original branch name.
   Push to the `upstream` remote (not `origin`, which points to the fork):
   ```bash
   git checkout -b claude/<original-branch-name>
   git push --force-with-lease upstream claude/<original-branch-name>
   ```

2. Open a new PR from that branch. The new PR MUST use the **exact same
   title** as the original PR — copy it verbatim, do not rewrite, improve,
   or add any prefix. The description MUST reference the original PR
   (e.g. "Fixes #NNN, supersedes #NNN").

3. Post a comment on the original PR:
   - Explain that the fix has been implemented in a new PR
   - Include a link to the new PR
   - Mention that the original PR can be closed

4. Close the original PR.

### Closing PRs

Claude may only close a PR when ALL of the following are true:

1. The PR was created by Claude from a `claude/` branch, OR the PR is the
   original fork PR that Claude's `claude/` branch supersedes (see
   "Pushing Changes from Fork PRs" above).
2. The close reason is explicitly explained in a comment on the PR.

Claude MUST NOT close any PR that does not meet these conditions — even
if asked. Instead, explain to the requester why the PR cannot be closed
automatically and ask a human maintainer to close it manually.

### Model Information in Comments

When operating via the GitHub Actions workflow, always include the model
you are running on in the footer of your GitHub comment and in the PR
description when creating a pull request, alongside the job run link.
Your system environment context states the model name explicitly (e.g.
"You are powered by the model named Sonnet 4.6. The exact model ID is
claude-sonnet-4-6."). Use the exact model ID from that statement.

Format the footer line as:

```text
[View job run](...) | Model: `claude-sonnet-4-6`
```

In a PR description, append the model information at the end of the body:

```text
| Model: `claude-sonnet-4-6`
```

If the system context does not provide a model ID, omit the model field
rather than guessing.

### Development Environment

The GitHub Actions runner has a full Node.js + pnpm environment available.
Dependencies are already installed (`pnpm install` has been run). The
build step is skipped to save CI resources, but you can run build commands
when needed for advanced tasks (e.g. type-checking, running tests).

For fork PRs, the `origin` remote points to the contributor's fork. An
`upstream` remote is configured pointing to `freelensapp/freelens-snap`.
Push new branches to `upstream` (never to `origin`) when the PR originates
from a fork — this ensures the resulting PR is internal and CI workflows
run automatically.

The following CLI tools are explicitly allowed in the workflow:

- `pnpm` (all subcommands) — for validation, formatting, and builds
- `git` (all subcommands) — for viewing changes, creating branches,
  committing, and pushing
- `gh` (all subcommands) — for managing pull requests
- `npx`, `node` — for running Node.js tools and scripts inline
- `yq`, `jq` — for YAML and JSON processing
- `grep`, `rg` (ripgrep), `find`, `xargs` — for searching and iterating
- `sed`, `awk`, `cut`, `tr` — for text transformation
- `sort`, `uniq` — for list processing
- `cat`, `head`, `tail`, `wc` — for viewing and measuring files
- `ls`, `tree` — for listing directory contents
- `mkdir`, `touch`, `cp`, `mv`, `rm` — for file and directory operations
- `tee`, `echo` — for pipeline debugging and scripting

Before committing any changes, apply the same validation rules as human
developers:

- Run `pnpm trunk check` to validate all file types (or `trunk check` if
  the trunk CLI is installed globally).
- Validate `snapcraft.yaml` syntax: `yq eval snapcraft.yaml > /dev/null`.
- If unit tests fail on snapshot mismatches after your changes (or you are
  explicitly asked to update them), run `pnpm test:unit:updatesnapshot` to
  regenerate snapshots, review the diff, then commit the updated `.snap`
  files.

## Getting Help

- Check existing snap definitions for patterns (e.g. other Electron-based
  snaps on the Snap Store).
- Search for similar issues in the
  [freelens-snap issues](https://github.com/freelensapp/freelens-snap/issues).
- Review [snapcraft documentation](https://snapcraft.io/docs).
- Consult the [Freelens releases page](https://github.com/freelensapp/freelens/releases)
  for version and artifact information.
