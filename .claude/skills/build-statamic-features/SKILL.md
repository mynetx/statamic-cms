---
name: build-statamic-features
description: Use when asked to build a feature the Statamic maintainers have already agreed to — source candidates from statamic/ideas "on deck" and the roadmap, verify the greenlight is real and current, implement it TDD on 6.x, verify in the local Herd sandbox, then PR upstream with a statamic/docs issue.
disable-model-invocation: true
---

# Build Agreed-Upon statamic/cms Features

Workflow for finding a feature the maintainers have **already said yes to**, confirming that yes is
real and current, building it in the fork, and proving it in the local Herd sandbox before PRing
upstream. Sibling of `triage-statamic-issues` (bugs); this one is features only.

Setup facts: `[[statamic-cms-fork-setup]]`. Conventions: `[[statamic-cms-test-policy]]`,
`[[statamic-cms-pr-style]]`, `[[statamic-cms-codex-review-gate]]`, `[[statamic-cms-comment-style]]`,
`[[statamic-cms-ai-norms]]`.

## Environment (local)

| Thing | Value |
|---|---|
| Fork (edit + phpunit here) | `/Users/jm/Web/mx/statamic/cms`, branch `6.x`, `origin`=fork, `upstream`=statamic/cms |
| Sandbox (run/verify here) | `/Users/jm/Web/mx/statamic/sandbox` — path-repo symlinked to the fork, serves fork code live |
| CP URL (Herd, TLS) | https://sandbox.test/cp · super user `jm@pixelkode.com` / `sandbox123` |
| PHP | `~/Library/Application Support/Herd/bin/php` (8.4) — Herd binary, not system php |
| composer / npm | `/usr/local/bin/composer`, `/opt/homebrew/bin/npm` (NOT the 1Password wrappers — they hang headless) |

## Step 1 — Source candidates

Feature requests live in **statamic/ideas** (plain issues, ~600 open), not statamic/cms — the cms
`feature request` label is defunct. The greenlight signal is the **`on deck`** label ("Under
evaluation for next release", applied by jackmcdade). 👍 reactions are the official prioritization
signal (jasonvarga: they sort by reactions). `gh issue list --json` has no reactions field — use REST:

```bash
# Primary pool — open on-deck ideas, with 👍 counts
gh api 'repos/statamic/ideas/issues?labels=on+deck&state=open&per_page=100' \
  --jq '.[] | "\(.number)\t+1:\(.reactions["+1"])\t\(.created_at)\t\(.title)"'

# Secondary pool — top-reacted open ideas WITHOUT on-deck (only count if Step 2 finds an
# explicit maintainer yes-comment; -label:nah drops rejected ones)
gh api -X GET search/issues -f q='repo:statamic/ideas is:issue is:open -label:"on deck" -label:nah' \
  -f sort='reactions-+1' -f order=desc -f per_page=20 -f advanced_search=true \
  --jq '.items[] | "\(.number)\t+1:\(.reactions["+1"])\t\(.title)"'
```

Cross-check the roadmap: WebFetch `https://statamic.com/roadmap` and extract the **In Progress** and
**Planned** sections with their inline issue numbers. **In Progress = OFF-LIMITS** — the core team is
actively building those (collision guaranteed). **Planned** items are candidates only when a matching
ideas/cms issue pins down scope (e.g. "Optional Taxonomy Magic Routes" → ideas#839); a roadmap line
alone is not a spec.

## Step 2 — Verify the agreement is real AND current

An `on deck` label is a start, not proof — many were applied in the v3 era. For each candidate:

1. **Read the thread**: `gh issue view NNN -R statamic/ideas --comments`. Quote the actual maintainer
   yes (jackmcdade, jasonvarga, duncanmcclean, jaygeorge) — "would love this" from a community member
   doesn't count.
2. **When was the label applied?** The label date matters, not the issue date — #223 is a 2020 issue
   but was put on deck 2026-02-14:

```bash
gh api repos/statamic/ideas/issues/NNN/timeline --paginate \
  --jq '.[] | select(.event=="labeled") | "\(.label.name) by \(.actor.login) at \(.created_at)"'
```

3. **Not already built / being built** — all four checks:

```bash
# a) cross-referenced PRs on the idea (catches cms PRs referencing it)
gh api graphql -f query='query{repository(owner:"statamic",name:"ideas"){issue(number:NNN){
  timelineItems(itemTypes:[CROSS_REFERENCED_EVENT],last:20){nodes{
    ... on CrossReferencedEvent{source{... on PullRequest{number title state repository{nameWithOwner}}}}}}}}}' \
  --jq '.data.repository.issue.timelineItems.nodes[].source | select(.number) | "\(.repository.nameWithOwner)#\(.number) [\(.state)] \(.title)"'

# b) cms PRs by keyword (titles describe the feature, not the idea #)
gh pr list -R statamic/cms --state all --search "<keywords>" --json number,title,state
```

   c) grep the 6.x codebase + statamic.dev docs — the feature (or half of it) may already exist;
   d) scan thread comments for anyone announcing "working on this".

Worked skip: idea #223 (localize asset alt text) is on deck with 115 👍 — but check (a) surfaces open
cms PR #13928 implementing it, and the roadmap lists it as Planned. Skip; pick another.

## Step 3 — Pick by confidence + size

Build only when, from the thread alone, you can write a one-paragraph spec with no open questions:

- **Scope pin-downable** — the thread states expected behaviour; no unresolved UX debate, no
  `feedback wanted` still in play.
- **Small first** — a few files, no new architecture, no new subsystem. New tag parameter, modifier,
  fieldtype option, command flag: yes. New subsystem or CP area: no.
- **Analogous pattern exists** — you can name an existing feature in the codebase whose structure
  you'll mirror (file layout, registration, tests). If nothing similar exists, it's not small.
- **Testable headless** — provable via phpunit/tinker, or a short CP click-path.

If nothing qualifies, report which candidates you checked and why each failed — don't force a weak pick.

## Step 4 — Announce intent (decision rule)

CONTRIBUTING.md ("Core Enhancements") wants an issue first and warns they may redirect features to
addon-land. Rule:

- **Small + fully pinned scope** (Step 3 bar cleared, no judgment calls made): build directly — the
  PR itself is the announcement; the on-deck label / maintainer yes is the pre-approval.
- **Non-trivial, or you made ANY scope judgment the thread doesn't settle**: comment on the ideas
  issue first — one short paragraph: what you'll build, the approach, the analogous pattern you'll
  mirror (`[[statamic-cms-comment-style]]`). Wait for a maintainer ack before coding. Silence is not
  a yes — timebox ~3–5 days, then pick another candidate rather than building on hope.

## Step 5 — Design before code

Write the spec before the branch: inputs/outputs, naming, edge cases — each traced to a sentence in
the idea thread, not invented. Then find the nearest analogous feature and read its full vertical
(src class, registration, tests, docs) — mirror it, don't innovate on structure. For anything with a
CP/UI surface: it's Vue 3 under `resources/js`, needs `npm run build` to show in the sandbox, and has
UX decisions — invoke `superpowers:brainstorming` with the user before writing code.

## Step 6 — Build TDD on 6.x

New features target **`6.x` only** — no branch decision needed (5.x takes security patches only,
`[[statamic-cms-branch-targeting]]`).

```bash
cd /Users/jm/Web/mx/statamic/cms
git fetch upstream && git switch -c feat/<slug> upstream/6.x
```

Failing test first; mirror the analogous feature's test file; focused cases per branch of new code,
don't overtest (`[[statamic-cms-test-policy]]`). Match surrounding comment style. Then:

```bash
PHP="$HOME/Library/Application Support/Herd/bin/php"
"$PHP" -d memory_limit=-1 ./vendor/bin/phpunit tests/Path/To/YourTest.php
./vendor/bin/pint <changed files>
./vendor/bin/pint --test   # full repo before push — CI lints everything; if files you didn't
                           # touch fail, the base branch drifted: fix in a separate
                           # "Fix code style" commit, note it in the PR
```

## Step 7 — Verify in the sandbox (REQUIRED before PR)

The sandbox serves the fork's PHP live via the path repo — backend changes apply with no rebuild.

- **Backend feature:** demonstrate it working end-to-end — `"$PHP" please tinker` script or template
  in the sandbox site. Give the user the exact demo: snippet or numbered CP clicks, output before
  (feature absent) and after.
- **CP/UI feature:** `cd /Users/jm/Web/mx/statamic/cms && /opt/homebrew/bin/npm run build`, hard-reload
  https://sandbox.test/cp, walk the flow. NEVER commit `resources/dist*` (gitignored;
  anti-malware rule — core team regenerates).

## Step 8 — Docs issue (required for feature PRs)

CONTRIBUTING.md:86: feature PRs must open a referencing issue in **statamic/docs** — they write the
docs, hints appreciated. Do this right after `gh pr create` so you can reference the PR number:

```bash
gh issue create -R statamic/docs \
  --title "Document <feature> (statamic/cms#<PR>)" \
  --body "statamic/cms#<PR> adds <one-liner>. Hints: <2-3 bullets: name, where it lives, example usage>."
```

## Step 9 — Review gate, then PR

0. **Re-run Step 2 check (a) on the idea now** — hours passed; someone may have opened a PR meanwhile.
1. Critical Codex review of the diff; apply judgment, don't blind-comply
   (`[[statamic-cms-codex-review-gate]]`).
2. Title `[6.x] <summary>` (prefix mandatory). Body short, problem-first, first person
   (`[[statamic-cms-pr-style]]`); UI features get Before/After screenshots. **No em-dashes**
   anywhere public (title/body/comments) — user flags them as an AI tell. The Claude co-author
   byline on commits must **name the model actually running the session** — check your own
   identity, don't copy an example (`[[statamic-cms-ai-norms]]`). Optional but maintainer
   practice: end the PR body with `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
   Link the idea **cross-repo** — `Fixes #NNN` won't close across repos, so write:
   `Implements statamic/ideas#NNN` (maintainers close the idea with the `added` label themselves).
   Mention the docs issue once opened. **One** PR.

```bash
git push -u origin feat/<slug>
gh pr create --repo statamic/cms --base 6.x --title "[6.x] <summary>"
git switch 6.x   # return the fork to its base branch
```

## Red flags — stop, this is not an agreed feature (or not yours to build)

- `on deck` applied years ago and **no maintainer signal since** (check the label timeline, not the issue date).
- Roadmap lists it **In Progress** — the core team is on it.
- Scope needs UX decisions the thread doesn't answer, or `feedback wanted` is unresolved.
- A maintainer hinted "addon territory" anywhere in the thread.
- No analogous pattern in the codebase to mirror — it's bigger than it looks.
- Feature already (partially) exists in 6.x, or an open/merged cms PR covers it (#223 → cms#13928).
- Idea carries `nah` (rejected) or `added` (shipped).
- Your "maintainer yes" is actually a community member's enthusiasm.
