# cc-resources

Battle-tested prompts, workflows, and patterns for building real SaaS with Claude Code.

Not blog-post theory. Everything here has been used on production code, broken at least once, and refined until it held up. The goal: save you the iteration loop I already paid for.

Maintained by [@mhamzahashim](https://github.com/mhamzahashim).

---

## Contents

### Prompts

| Name | What it does |
|------|--------------|
| **[Claude Code Stress Plan](prompts/claude-code-stress-plan.md)** | A 4300-word self-review prompt you paste after Claude Code drafts a plan in plan mode. Forces Claude to stress-test the plan against every layer of the stack (build, routing, UI, hooks, API, DB, security, deploy, etc.) and surface missing pieces before a single line of code is written. Cut my production regressions by ~90%. |

### Coming soon

- Settings configurations (`settings.json`, `CLAUDE.md` templates)
- Custom hooks
- Slash commands and skills
- Workflow guides (git flow, ship-readiness checklist, deploy flow)

---

## How to use

Each file is self-contained. Click into a prompt or guide, copy the content, and paste it where it belongs (plan mode, `settings.json`, `CLAUDE.md`, a hook script). Most files include usage notes at the top.

## Philosophy

I don't share things here because they sound clever. I share them because they shipped, broke, got debugged at 2am, and were refined until they worked. If a prompt or hook is here, it earned its keep on real production code.

## Contributing

If you fork a prompt for a different stack (Rails, Django, Laravel, Nuxt, FastAPI, etc.), open a PR or drop it in an issue. A stack-specific library of these would be genuinely useful to the community.

## License

MIT. Do whatever you want. Attribution appreciated but not required.
