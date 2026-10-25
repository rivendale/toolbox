# toolbox

**One front door to every skill, workflow and public repo we maintain.** Toolbox will catalog each
one, say how to install it, and show its latest test result. It holds no skill source: each skill
stays in the repo where it is built and tested.

**Status: not built yet.** [SPEC.md](SPEC.md) is a draft (revision 3); building starts once its
reviewer signs off. Issue #2 tracks the first milestone.

## Until then, the tools live here

| Repo | What |
|---|---|
| [ai-redteam](https://github.com/rivendale/ai-redteam) | Adversarial review for AI-produced work: red-team decisions, plans and claims, plus bounded PR review. Claude skills and paste-in prompts, with an eval set. |
| [ci-workflows](https://github.com/rivendale/ci-workflows) | A reusable GitHub Actions workflow: an AI code-review panel for pull requests. |
| [hsi-operator](https://github.com/rivendale/hsi-operator) | Keep a human at the strategic level of agent work: surface only what needs a person, and establish what 'done' means before building. |

## License

MIT, see [LICENSE](LICENSE).
