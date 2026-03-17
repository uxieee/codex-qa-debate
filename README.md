# codex-qa-debate

**Two AIs walk into a code review. Only production-ready code walks out.**

A Claude Code skill that runs automated 3-round QA debates between Claude Code and OpenAI's Codex CLI. Instead of one AI reviewing its own work, two different AI models argue about your code until they reach consensus — or escalate to you.

## How It Works

```
Round 1  Claude sends code to Codex for review
         Codex (as a Staff Engineer with 15+ years of experience) finds issues

Round 2  Claude rebuts disputed findings with evidence
         Codex either concedes, holds firm, or revises

Round 3  Final arguments on remaining disputes
         Unresolved items go to you for the final call
```

You watch the debate unfold in real-time and approve before anything gets fixed.

```
QA Debate Summary — Loading Screen Redesign

| # | Finding                          | Severity | Verdict       | Rounds |
|---|----------------------------------|----------|---------------|--------|
| 1 | Missing null check in showError  | Medium   | Agreed fix    | 1      |
| 2 | Tips array should be const       | Low      | Agreed no-fix | 2      |
| 3 | Spinner ARIA role insufficient   | Medium   | Deadlocked    | 3      |

Agreed fixes: 1
Agreed no-fix: 1
Deadlocked: 1 (needs your call)
```

## Why Two AIs?

A single AI reviewing its own code has blind spots. The same reasoning that produced the bug will rationalize why it's fine. By bringing in a second model (GPT) as an adversarial reviewer, you get:

- **Different failure modes** — Claude and GPT catch different things
- **Real debate** — Not a rubber stamp. Codex is prompted to hold firm unless shown concrete proof
- **Human final say** — You approve everything before fixes land. Deadlocked items are YOUR call

## Install

```bash
npx skillsadd uxieee/codex-qa-debate
```

Or manually:

```bash
mkdir -p ~/.claude/skills/codex-qa-debate
curl -o ~/.claude/skills/codex-qa-debate/SKILL.md \
  https://raw.githubusercontent.com/uxieee/codex-qa-debate/main/codex-qa-debate/SKILL.md
```

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- [Codex CLI](https://github.com/openai/codex) installed and authenticated (`codex` command in PATH)
- A git repository with code changes to review

## Usage

After installing, just tell Claude Code:

```
run QA on my recent changes
```

Or any of these:

```
codex review
qa debate
review my changes
code review with codex
```

The skill handles everything:
1. Gathers changed files via `git diff`
2. Asks if you have a spec or out-of-scope items
3. Runs all 3 debate rounds
4. Presents the summary table
5. Waits for your approval
6. Fixes agreed issues and commits

## The Staff Engineer Persona

Codex doesn't just "review" your code. It reviews it as a Staff Engineer who:

- Has been paged at 2 AM because of bugs that "passed review"
- Reviews code as if they'll be oncall for it
- Checks for failure modes, cascade errors, security issues, and scalability
- Does NOT concede findings without concrete evidence (code references, tests, or spec citations)
- Holds firm in Round 3 if they genuinely believe an issue is real

## Configuration

The skill uses `gpt-5.4-xhigh` by default for maximum review quality. To change the model, edit the `-m` flag in the `codex exec` commands inside `SKILL.md`.

Codex runs in **read-only sandbox mode** (`-s read-only`) — it can read your codebase but cannot modify any files.

## License

MIT
