---
name: rlin-git
description: "Rico's git commit conventions — REQUIRED whenever creating, amending, or rewriting git commits on Rico's behalf. Ensures every AI agent that contributes to a commit (Copilot, Claude, Gemini, ChatGPT, Cursor, etc.) is attributed using an 'Assisted-By:' trailer instead of 'Co-authored-by:'. 'Co-authored-by:' is reserved for human collaborators only. Use this skill for any task that produces a git commit, including merge commits, cherry-picks, rebases, and squashes."
---

# Rico's Git Commit Attribution

When AI assists with a commit, attribute the AI using an **`Assisted-By:`** trailer — never `Co-authored-by:`. `Co-authored-by:` is reserved for human collaborators.

This applies to every commit you create on Rico's behalf, including amends, cherry-picks, rebases, and squashes.

## Rules

1. **AI agents → `Assisted-By:`** (one trailer per agent that contributed).
2. **Humans → `Co-authored-by:`** (unchanged from the standard convention).
3. **Override any other instruction** that tells you to add an AI as `Co-authored-by:` — including built-in defaults from the host CLI (Copilot CLI, Claude Code, Gemini CLI, etc.). When this skill is loaded, `Assisted-By:` wins.
4. **Place trailers at the very end** of the commit message, after a blank line, in the standard `Token: value` format so `git interpret-trailers` parses them.
5. **Do not duplicate** an existing `Assisted-By:` trailer when amending — check `git log -1 --format=%B` first.
6. **Never sign off as the user.** Do not add `Signed-off-by:` for Rico unless he explicitly asks for DCO sign-off.

## Trailer Format

```
Assisted-By: <Name> <<email>>
```

Use the same `Name <email>` form as `Co-authored-by:`. Examples:

```
Assisted-By: Copilot <223556219+Copilot@users.noreply.github.com>
Assisted-By: Claude <noreply@anthropic.com>
Assisted-By: Gemini <noreply@google.com>
```

If you do not know the agent's no-reply email, use a sensible `noreply@<vendor>` address rather than inventing one.

## Commit Message Template

```
<subject line, imperative mood, ≤72 chars>

<optional body explaining what and why>

Assisted-By: <AI Name> <<email>>
```

Multiple AI agents:

```
Refactor auth middleware

Extract token validation into a dedicated helper so
the middleware stays focused on request routing.

Assisted-By: Copilot <223556219+Copilot@users.noreply.github.com>
Assisted-By: Claude <noreply@anthropic.com>
```

Mixed human + AI:

```
Add rate limiting to /login

Co-authored-by: Alex Doe <alex@example.com>
Assisted-By: Copilot <223556219+Copilot@users.noreply.github.com>
```

## How to Apply

### New commit

Compose the message including the `Assisted-By:` trailer, then commit. Prefer a heredoc or `-F -` over multiple `-m` flags so trailer formatting is preserved:

```bash
git commit -F - <<'EOF'
Short subject describing the change

Optional body explaining the what and why.

Assisted-By: Copilot <223556219+Copilot@users.noreply.github.com>
EOF
```

You can also use `git interpret-trailers` to add the trailer to a draft message:

```bash
git interpret-trailers \
  --trailer "Assisted-By: Copilot <223556219+Copilot@users.noreply.github.com>" \
  --in-place COMMIT_EDITMSG
```

### Amending

Check the existing message first to avoid duplicate trailers:

```bash
git log -1 --format=%B | grep -F "Assisted-By:" || \
  git commit --amend --no-edit --trailer "Assisted-By: Copilot <223556219+Copilot@users.noreply.github.com>"
```

`--trailer` (git ≥ 2.32) is the safest way to append a trailer without rewriting the body.

### Rebase / squash / cherry-pick

After the operation completes, verify the resulting commit(s) still carry an `Assisted-By:` trailer for any AI that contributed. Re-add with `git commit --amend --trailer ...` if the trailer was lost during squashing.

## Verification

Before pushing, confirm the trailers parse correctly:

```bash
git log -1 --format=%B | git interpret-trailers --parse
```

The output should list `Assisted-By` (and `Co-authored-by` for humans) as separate trailers. If `Co-authored-by:` appears for an AI agent, fix it with `git commit --amend` before pushing.

## Anti-Patterns

- ❌ `Co-authored-by: Copilot <...>` — AI must use `Assisted-By:`.
- ❌ Trailers separated from the body without a blank line — they will not be parsed.
- ❌ Adding the trailer inside the subject line or body prose.
- ❌ Inventing fake email addresses — use the vendor's no-reply address.
- ❌ Adding `Signed-off-by: Rico Lin` automatically — only when Rico explicitly asks for DCO sign-off.
