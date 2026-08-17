# silverbeer shared repo defaults

Dependency policy for every repo in this account, declared once here and
inherited. A repo says *which* policy it follows; it never restates *what* the
policy is.

## Using it

One line, in a `renovate.json` at the root of any repo:

```json
{ "extends": ["local>silverbeer/.github"] }
```

That resolves `default.json` from this repo's default branch. Change the policy
here and every repo picks it up on its next run — there is nothing to copy and
nothing to keep in sync.

## Why this repo, and not somewhere else

The consumers are GitHub Actions and Renovate, and both resolve `silverbeer/...`
over the GitHub API. That rules out anything living only on a laptop.

There is a real boundary here, and it is the idea the whole setup rests on:

| | owns | answers |
| --- | --- | --- |
| **dotfiles** (chezmoi) | what tools exist on *my machines*, at what version | "why does the mac mini behave differently?" |
| **this repo** | what versions a *repo* uses in CI and hooks | "why do missing-table and myrunstreak disagree?" |

Version-skew bugs live on the seam between those two. **Where a tool exists on
both sides, the fix is to delete one copy, not to pin both** — a version that
exists in one place cannot skew.

`.github` is the name because GitHub already resolves account-level defaults
here, so reusable workflows and issue templates can land in the same repo later
without needing a second home.

## The rules, and why

### Group by ecosystem, not by dependency

One PR for "github actions", one for "python dependencies", and so on. Forty
separate *bump actions/checkout* PRs is how automation gets muted, and muted
automation is worse than none — it looks like coverage while being ignored.

### Auto-merge patch and minor; majors are always human

Patch and minor merge themselves when CI is green. Majors get their own PR, a
`major-update` label, and a person who reads the changelog.

**Caveat worth knowing:** auto-merge is only as good as the checks behind it. A
repo with no CI has nothing to prove a bump safe, so enabling this there merges
on hope. Repos without CI should either get some or be excluded deliberately.

### Formatters never auto-merge

A formatter's output *is* its contract. Ruff 0.9 changed how a sole dict argument
is formatted; a repo that picked that up silently would rewrite hundreds of
untouched lines the next time anyone ran it.

This is not hypothetical. In `missing-table`, `.pre-commit-config.yaml` pinned
ruff `v0.8.4` while `pyproject.toml` declared `ruff>=0.8.0`, which resolved to
0.12.12 locally. Running the formatter after a 25-line edit produced a
**199-line diff** — 102 insertions, 97 deletions, nearly all untouched code. It
was silent: ruff exited 0, `ruff check` passed, the tests passed. Nothing flagged
that 190 of the changed lines were noise.

Hence also: **`>=` on a formatter is always wrong.** It reads as "at least this
good" and behaves as "whatever shipped this morning".

### pre-commit hook revs are managed, and grouped separately

Renovate's `pre-commit` manager is **disabled by default**, so it is turned on
explicitly here. It is the mechanism that catches hook-rev drift before it bites,
which is exactly the failure above.

### A weekly window, and a three-day release age

Updates land in one Monday batch, so review is a habit rather than an interrupt.
`minimumReleaseAge: 3 days` means a release that gets yanked or hot-fixed over a
weekend is never picked up at all.

The window is a **whole day**, not a morning, and that is a consequence of the
hosting tier. Mend's free Community Cloud allows **one concurrent job per
account** and scans each repo roughly **every four hours**. A seven-hour window
therefore gives a repo about two chances to be picked up, with every repo in the
account queueing through a single job slot — so some would silently miss their
turn. A full day removes the race without spreading review across more than one
day.

Worth knowing if the tier ever changes: Enterprise Cloud raises this to 16
concurrent jobs and hourly scans, at which point a narrower window is safe again.

Security advisories bypass both — they are exempted from the schedule and the
waiting period.

### Runtime upgrades are migrations

A Python version bump is a decision about what you support and test, not a
dependency bump. It is labelled `runtime-upgrade` and never auto-merges.

## What was rejected

- **A monorepo.** Solves version skew by deleting the problem statement, and
  costs far more than the skew did.
- **Hand-rolled version sync in `janitor`.** Its value is orchestration Renovate
  cannot do. Reimplementing a dependency bot means maintaining the maintainer.
- **CI config in dotfiles.** Wrong distribution channel: chezmoi propagates when
  a human runs `chezmoi update`, and repo correctness must not depend on whether
  someone remembered to update their laptop.
- **`missingtable-platform-bootstrap` as the home.** It is IaC for
  missingtable.com and qualityplaybook.dev, not a developer platform.

## Taking the baseline

The argument for any of this is the measurement, not the assertion. Re-run these
from the directory holding the repos:

```bash
# Action versions in use, across every repo
grep -rhoE "uses: (actions|astral-sh|docker)/[a-z-]+@v?[0-9.]+" \
  */.github/workflows/*.y*ml | sed 's/uses: //' | sort | uniq -c | sort -rn

# Python floors
grep -rho 'requires-python = "[^"]*"' */pyproject.toml */*/pyproject.toml

# pre-commit hook revs
grep -A2 'astral-sh/ruff-pre-commit' */.pre-commit-config.yaml
```

Baseline on 2026-08-17, across 11 of 27 local repos:

- `actions/checkout` — v4 (49 uses) **and** v5 (10)
- `astral-sh/setup-uv` — **five majors live at once**, v3 through v7
- `actions/setup-python` — v4, v5, v6
- Six different Python floors, from `>=3.8` to `>=3.14`
- ruff hook revs spanning v0.6.8 to v0.14.2
- No Renovate or Dependabot anywhere

Tracked as the **Dependency & Version Hygiene** epic (SB-661 onward).
