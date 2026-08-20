# Agent Note: Companion writable roots — granting a workspace's sibling worktree tree under `workspace-write`

Status: implemented

English | [中文](2026-08-20-sibling-companion-writable-roots.zh.md)

## Problem

`workspace-write` grants exactly one root: the session's immutable cwd. A repository whose worktrees live in a sibling directory (`/repos/app` working tree, `/repos/app.worktrees/<branch>` worktrees) therefore has half its working set outside the boundary. Every worktree create, checkout, and edit lands in a denied path, and the only levers available are both wrong: widen the whole session to `danger-full-access`, which drops confinement entirely, or root the session at `/repos` so the parent's every sibling repository becomes writable.

The gap is structural rather than per-tool. `writableRoots` is the one place `workspace-write` means something, and it derived that meaning from a single field, so no consumer could express "the workspace plus this one companion" without a mode change.

## Decision

The policy owner resolves companion roots; the mode's vocabulary is unchanged.

### `SandboxExecutionPolicy.extraWritableRoots`

An optional `readonly string[]` carried beside `workspaceRoot`. Absent by default, so the resolved policy of a deployment that configures nothing is byte-identical to before. `writableRoots` folds it into the same canonical, deduplicated allow-list, which means the Seatbelt profile and the in-process fs fence pick it up without either one learning a new concept.

### `dsh-sandbox-policy.siblingWritableSuffixes`

A validated `Config` field: suffixes appended to the resolved workspace root's own path, each naming one companion directory. `['.worktrees']` resolves `/repos/app` → `/repos/app.worktrees`.

Suffixes rather than free paths, because the grant then cannot name anything but a sibling of the workspace: the derived path is computed from the session's own root, never supplied per call. An entry that is empty or carries a path separator is refused at load. Suffixes apply to the workspace's execution-world spelling, and `writableRoots` canonicalizes the workspace and its companion roots together, so enforcing providers compare one identity.

### Per-runner grant spellings

Seatbelt matches path strings, so a companion root is granted whether or not it exists — the agent can therefore create the worktree tree itself, which is the common first step.

bwrap `--bind` and the Landlock launcher both fail closed on a path they cannot open (`add_rule` returns `EXIT_LAUNCHER_FAILURE` on an unopenable grant root), so on those runners a companion root joins the grants only once it exists. A later call re-derives the profile and picks it up. Filtering lives in the dialect builders, not in `writableRoots`, because it is a runner materialization constraint rather than a change to what the mode promises.

### Model experience

`sandbox:policy` names the companion roots in the `workspace-write` context so the model knows it may write there. The sentence appears only when roots were resolved, keeping the default rendering byte-stable.

## Alternatives considered

**Hardcode `.worktrees` in `writableRoots`.** Smallest change, and wrong: it makes one team's layout convention a property of the sandbox vocabulary, and a deployment using a different directory name has no lever. Deployment-varying choices are validated `Config` fields.

**Accept absolute companion paths instead of suffixes.** More general, and strictly worse here: an absolute path is unrelated to the session's root, so a misconfigured entry silently grants a directory the workspace has nothing to do with. Suffixes make the grant derivable from the workspace, which is the property worth keeping.

**Grant the workspace's PARENT.** Covers the worktree tree with no new configuration, and grants every sibling repository alongside it — the boundary would exist in name only.

**Add a fourth `SandboxMode`.** A mode is a promise about file effects; "workspace plus companions" is the same promise over a different root set, so it belongs in root derivation rather than in the vocabulary every backend switches on.

## Consequences

- A deployment opts in per composition; the shipped bundles configure no suffix, so default confinement is unchanged.
- The companion grant is as wide as the named directory. `.worktrees` is writable in full, which is the intent — it is the agent's own scratch tree — but it is not narrowed to one branch's worktree.
- bwrap and Landlock behave differently from Seatbelt for a not-yet-created companion root. Pinned by test rather than smoothed over, matching how the dialects already differ on `/tmp`.
