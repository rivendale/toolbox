# toolbox: specification

Status: DRAFT for review. Build starts after a second reader signs off.

## What it is

**One front door to every skill, workflow and public repo we maintain.** Toolbox holds no skill source.
Each skill stays in the repo where it is built and tested. Toolbox catalogs it, says how to install it,
and shows its latest test result.

Why a front door and not a monorepo: a skill's tests, cases and history live beside it. Copying the skill
here would make two versions, and one of them would go stale.

## What it contains

| Path | What |
|---|---|
| `README.md` | The front door: one table of skills, one of workflows, one of repos. Generated. |
| `catalog.json` | The source the README is built from. Written by hand, checked by machine. |
| `workflows/` | Methods we repeat, one file each (see below). The only original content here. |
| `.claude-plugin/marketplace.json` | One marketplace that points at each skill's home repo, at a pinned version. |
| `tools/build.py` | Builds the README from `catalog.json` and the live repos. |
| `tools/check.py` | Fails when the catalog and the repos disagree. Runs nightly and on every PR. |

## The catalog: one row per skill

| Field | Rule |
|---|---|
| `name`, `repo`, `path` | Where the skill lives, such as `rivendale/ai-redteam`, `skills/redteam`. |
| `version` | The version the skill declares, read from the repo, never typed here. |
| `pin` | The commit the marketplace installs. |
| `license` | Read from the repo's LICENSE file. |
| `what` | One sentence: what a user gets. |
| `eval` | A link to the latest published result, its date, and the score. Use `none` if there is no eval. A skill with no eval says so on the front page. |
| `runs_on` | Claude Code, Codex, Gemini CLI, chat: only where it has been tried. |

Skills at the start:
- **rivendale/ai-redteam:** redteam, pr-review, assess.
- **rivendale/hsi-operator:** hsi-operator, context-steward, repo-triage, system-map, update.
- **rivendale/opensource:** the five studio skills (game-art, game-sfx, game-music, game-video, game-prototype) as they ship, each with its eval cases under `evals/studio/`.

## Workflows: methods written down once

Each workflow file follows the same shape:
- when to use it;
- the steps;
- what a done state looks like;
- the check that proves it;
- an example from a public repo.

Workflows at the start:
1. **Build a tested skill.**
   - Write the spec and every way the skill could fail, first.
   - A different agent writes the cases.
   - Each clean case gets a blind solve in a neutrally named empty folder.
   - Lock the pass bar before any scored run.
   - The final run uses unseen hold-out cases, and its result is published whatever it shows.
2. **Second reader.**
   - No author reviews their own change.
   - A review is findings, each with evidence; the owner decides.
   - The merge waits for the review and for green checks.
3. **Findings, not directives.** Messages between agents carry what was measured, tagged with how it is known, and a date.
4. **Gates, not promises.** A rule that matters ships with a check, and the check is proved on an input it must refuse.
5. **Hand off with a receipt.** Work handed to another agent gets an acknowledgement, or an alert after a set time.

## Repos

A table of every public repo, one line each, grouped by what it is for:
- **rivendale:** opensource, ai-redteam, hsi-operator, local-ai, homelab, ci-workflows, and the public games.
- **Protocol-Wealth (public repos only, as links):** pwos-core, nexus-core, pw-redact-core, pw-router, pw-aave, pwplan-core, pw-learnai-core, pwcli-core, shard-core, pw-ai-agent-code-core.

Protocol Wealth owns its repos. Toolbox only links to them and copies none of their text. A row for a
Protocol Wealth repo is a name, a link, its license and its own description.

## What the check must catch (written before the build)

`tools/check.py` exits non-zero, naming the row, when any of these is true:

1. A catalog row points at a repo, path or file that does not exist.
2. A row's `version` differs from the version the skill's repo declares. Example found while writing this
   spec: the ai-redteam marketplace says 2.2.0 while redteam v2.3.2 is live.
3. A `pin` is not an ancestor of the repo's default branch, or is more than 90 days older than its head.
4. A row's `license` differs from the repo's LICENSE file.
5. An `eval` link is dead, or its date is older than the skill's last change. The second case is a stale
   score, shown on the front page as STALE, not as a failure.
6. A skill folder exists in a cataloged repo but has no catalog row.
7. The README differs from what `tools/build.py` would produce.
8. Any file in this repo names a family member, a private repo, a client or an assistant attribution line.

Each item gets a fixture that must fail before the check is accepted. Item 6 also gets one that must
pass: a folder that is not a skill.

## Out of scope

- Moving or copying skill source.
- Private repos.
- Any build or review work on Protocol Wealth repos.
- A website. The README is the front page.

## Open questions for the reviewer

1. Should a skill with no eval be listed at all, or listed under "untested"?
   Proposed: listed, marked untested.
2. Should the marketplace pin follow each repo's latest release tag, or a commit chosen by hand?
   Proposed: release tags where a repo has them, otherwise a commit.
