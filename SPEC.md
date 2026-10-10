# toolbox: specification

Status: DRAFT, revision 3. It answers the 18 findings of the first read and the 8 of the second. Build starts after the
reviewer signs off on this revision.

## What it is

**One front door to every skill, workflow and public repo we maintain.** Toolbox holds no skill source.
Each skill stays in the repo where it is built and tested. Toolbox catalogs it, says how to install it,
and shows its latest test result.

Why a front door and not a monorepo: a skill's tests, cases and history live beside it. Copying the skill
here would make two versions, and one of them would go stale.

## What it contains

| Path | What |
|---|---|
| `README.md` | The front door: tables of skills, workflows and repos. Generated; never edited by hand. |
| `catalog.json` | Hand-written: which plugins, skills, workflows and repos exist, and their pins. |
| `snapshot.json` | Machine-written by `refresh.py`, at one moment: every fact the checks need, including each link's HTTP status. Refreshed by a bot PR, never by hand. |
| `workflows/` | Methods we repeat, one file each. The only original content here. |
| `.claude-plugin/marketplace.json` | Generated from `catalog.json`. Points at each plugin's home repo at its pinned SHA. |
| `tools/build.py` | `build(catalog, snapshot)`, a pure function. It writes the README and the marketplace, and reads nothing live. |
| `tools/refresh.py` | Reads the live repos and writes `snapshot.json`. |
| `tools/check.py` | Exits 0 when everything agrees, 1 when something disagrees, and 2 when it could not check. It names the row. When one run has both, 1 wins, and the exit-2 repos are printed too. It takes `--today` so date rules can be tested. |

## Units: plugin and skill

The marketplace installs a **plugin**, so a plugin is the unit of pin, version and install. For example,
`rivendale/ai-redteam` is one plugin holding three skills. A **skill** is the unit of description and eval.

- A **plugin** row has `repo`, `subdir` (optional), `pin` (a full 40-hex SHA), `tag` (optional), `version`,
  `licenses` and `skills`. `licenses` lists each SPDX identifier and the path it covers.
  The version is read from the repo's `.claude-plugin/plugin.json`, into the snapshot.
- A **skill** row has `name`, `path`, `what`, `eval` and `runs_on`.
- A **skill folder** is a folder holding a `SKILL.md` whose frontmatter has `name` and `description`.
  Nothing else counts.

Skill-level release names, such as "redteam v2.3.2", are prose. Toolbox does not read them. The eval's
`measured_on` field is what ties a score to code.

## Eval fields (per skill)

| Field | Rule |
|---|---|
| `result_url` | A machine-readable result file (JSON) in the skill's repo, at a commit SHA. |
| `set` | `dev` or `hold-out`. The front page shows the hold-out result. A dev-only result shows as "dev only". |
| `measured_on` | The plugin SHA the result was measured on. |
| `bar_locked` | A link to the commit that locked the pass bar before the run. |
| score | Never stored in the catalog. `build.py` reads it from the result file, as captured in `snapshot.json`. |

To declare no eval, write `eval: {"none": "<reason>"}`. The field must be present.

Untested skills are listed, marked "untested", and sorted after tested ones. A plugin installs all of its
skills, so **a plugin is untested if any of its skills is**. An untested plugin's marketplace entry is named
`<plugin>-untested`.

**The result file schema is `toolbox-result-1`:**
```json
{"schema": "toolbox-result-1", "skill": "redteam", "set": "hold-out", "measured_on": "<40-hex>",
 "metrics": {"recall": 53}, "bar": {"recall": {">=": 53}}, "passed": true}
```
`build.py` recomputes `passed` from `metrics` and `bar`, and a mismatch fails (item 7).

## runs_on

Each entry names a runner (Claude Code, Codex, Gemini CLI, chat) and has a `tried` link to a run record:
a receipt, a transcript excerpt or an eval result. An entry without one fails.

## Workflows

Workflow 3 is a habit, not a gate: no check reads messages between agents.

Each file in `workflows/` has five headings, in this order:
1. When to use it.
2. Steps.
3. Done looks like.
4. The check that proves it.
5. Example, with a link into a public repo.

Workflows at the start:
1. **Build a tested skill.**
   - Write the spec and every way the skill could fail, first.
   - A different agent writes the cases.
   - Each clean case gets a blind solve in a neutrally named empty folder. Reset the folder before a retry,
     because an agent that died mid-run leaves files behind.
   - Lock the pass bar before any scored run.
   - The final run uses unseen hold-out cases, and its result is published whatever it shows.
2. **Second reader.**
   - No author reviews their own change.
   - A review is findings with evidence; the owner decides.
   - The merge waits for the review and green checks.
3. **Findings, not directives.** Messages between agents carry what was measured, how it is known, and a date.
4. **Gates, not promises.** A rule that matters ships with a check, proved on an input it must refuse.
5. **Hand off with a receipt.** Work handed to another agent gets an acknowledgement, or an alert after a set time.

## Repos table

There is one row per public repo, grouped by purpose:
- **rivendale:** opensource, ai-redteam, hsi-operator, local-ai, homelab, ci-workflows, and the public games.
- **Protocol-Wealth:** pwos-core, nexus-core, pw-redact-core, pw-router, pw-aave, pwplan-core,
  pw-learnai-core, pwcli-core, shard-core, pw-ai-agent-code-core.

A Protocol-Wealth row is a link to the repo root, the SPDX license, and the repo's GitHub description shown
as a quotation and attributed to the repo. Toolbox writes no description of its own for those repos, and
links nowhere inside their code. Protocol Wealth owns them.

When a repo is archived, renamed or transferred, its row fails until a person updates it. The same applies
to a skill row.

## What the check must catch (written before the build)

**Two kinds of fact.** A fact addressed by a pinned SHA cannot change, so a disagreement is an error. A fact
about a branch head drifts with every commit, so a disagreement only means the snapshot needs a refresh.
Only `refresh.py` reads the network. Every item below reads `catalog.json` and `snapshot.json`, except item
11 and item 16.

`check.py` exits 1 and names the row for each of these:

1. A plugin `repo`, a skill `path`, or a `SKILL.md` does not exist at the pinned SHA.
2. A plugin's catalog `version` differs from `plugin.json` at the pin.
3. A `pin` is not 40 hex characters, does not resolve, or is not an ancestor of the default branch.
4. A `tag` no longer points at the `pin` (a moved tag).
5. A cataloged skill's path differs between the pin and the head of the default branch. This means an
   unreleased change. It shows as BEHIND, with exit 0. The clock starts at the first commit after the pin
   that touches the skill path, and it fails after 30 days.
6. A plugin's `licenses` differ from the SPDX identifiers of the license files at the pin, path by path.
   The fixture includes a repo with two licenses.
7. An eval result is unreachable at refresh, is not valid `toolbox-result-1`, has a `passed` that does not
   follow from `metrics` and `bar`, or names a `measured_on` that is not the cataloged pin or an ancestor of
   it. It also fails when the `bar_locked` commit is not an ancestor of, and older than, the result's commit.
   That check is what proves the bar was locked before the run. When `measured_on` is an older pin, the front
   page shows STALE and the exit is 0.
8. A skill folder exists in a cataloged repo with no catalog row. It shows as NEW, with exit 0, and fails
   after 14 days. The clock starts at the commit that added its `SKILL.md`.
9. `marketplace.json` differs from `build(catalog, snapshot)`, or an entry has no catalog row.
10. `README.md` differs from `build(catalog, snapshot)`. This is offline: it compares two committed files.
11. `snapshot.json` disagrees with live state. A pin-addressed fact that disagrees fails. A head fact that
    disagrees prints a refresh notice, and fails only when the snapshot is more than 7 days old.
12. A `runs_on` entry has no `tried` link, or the link is dead.
13. A workflow file lacks any of the five headings, or its example link does not resolve to a public repo path.
14. A repos-table link does not resolve, or points at a private or archived repo. A Protocol-Wealth row also
    fails when it links below the repo root, the owner is not Protocol-Wealth, or the repo was renamed,
    transferred or made private.
15. Forbidden content. Two layers:
    - **Public patterns** (in the repo): attribution trailers (`Co-Authored-By`, `Claude-Session`),
      absolute home paths, and key-shaped strings.
    - **Private names** (family members, private repos, clients and internal seat names): read from a CI secret,
      `TOOLBOX_DENYLIST`, which is never written to the repo or the log. A finding prints the file and line,
      never the matched text. Fork PRs have no secrets, so they run the public layer only. A fork PR is not
      merged until a maintainer runs the private layer on it. The private layer also runs on every push to
      `main` and nightly.
16. The nightly install fails. It installs each marketplace entry into an empty directory and confirms each
    skill's `SKILL.md` is present at its pin.

`check.py` exits 2, naming the repo, when GitHub cannot be reached or rate-limits it. A check that could not
run never passes.

Each item gets a fixture that must produce its exit code before the check is accepted. Items 8 and 15 also get
a fixture that must pass: a folder that is not a skill, and a word that only looks like a name.

**Pin format (verified 2026-10-10).** Anthropic's official marketplace pins plugins by commit:
`{"source": "url", "url": "<repo>.git", "sha": "<40-hex>"}`, or `"source": "git-subdir"` with `path`, `ref`
and `sha`. Toolbox uses the same fields. **Spike before the build:** install one plugin from a toolbox
marketplace entry pinned by `sha` and confirm the checked-out commit. If that fails, the pin design is
revised before any code.

## Out of scope

- Moving or copying skill source.
- Private repos.
- Any build or review work on Protocol Wealth repos.
- A website. The README is the front page.

## Before the build: changes in other repos

These are owner PRs, listed here so the first catalog can pass:
- **ai-redteam:** publish a machine-readable eval result per skill. Bump `plugin.json` past 2.2.0 to match
  what is live, and cut a release tag.
- **hsi-operator and opensource:** the same, when each skill has an eval.
