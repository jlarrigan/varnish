# Running Varnish in any coding agent

_For Cursor, Codex, GitHub Copilot, Gemini CLI, Windsurf, or any agent that can read files. Claude Code users: install the plugin instead (see README) and type `/varnish`._

**Agent: you were pointed at this file to run Varnish on the user's current project.** Do this:

1. **Varnish's root** is the folder this file is in. Wherever the playbooks say `${CLAUDE_PLUGIN_ROOT}`, use that folder's path.
2. **The command** is whatever the user asked for, in the shape `<type> [scope] [learn]` — e.g. "run varnish launch", "varnish security learn", "varnish status". That text is `$ARGUMENTS`. If they didn't name a type, follow the "No/unknown type" rule in the router.
3. **Read `skills/varnish/SKILL.md` in full** and follow it exactly: routing, the ⛔ Audit-mode rules, stack modules, Learn mode. The Audit-mode rules are not optional in any agent — look-only, nothing written outside `audits/` in the user's project, no installs, no git writes, ask before touching live accounts.
4. **Run against the user's project**, not this folder. Reports go in the user's project under `audits/`.
5. **No subagents in your tool?** Where a playbook says to verify with a separate subagent, do the verification as a deliberate second pass after finding, re-reading each cited line with the goal of disproving the finding, and say "verified in a second pass (no subagents)" in the report header.

## For the user: two ways to set it up

**Quick (any agent):** clone Varnish somewhere once:
```
git clone https://github.com/jlarrigan/varnish ~/varnish
```
Then, in your project, tell your agent:
> Read ~/varnish/START.md and run varnish launch on this project.

**So "varnish" works by name:** add this to your project's `AGENTS.md` (Codex, Copilot, Gemini, Windsurf all read it) or a Cursor rule (`.cursor/rules/varnish.mdc`):
```
## Varnish
When I say "varnish <something>" (e.g. "varnish launch", "varnish status"),
read ~/varnish/START.md and follow it.
```
Update anytime with `git -C ~/varnish pull`.
