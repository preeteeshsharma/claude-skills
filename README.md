# claude-skills

Personal Claude Code skills library. Backup and cross-device sync for custom skills.

## Skills

### interview-research

Researches real-world interview processes and questions for specific companies and roles using live web searches. Sources from Blind, LeetCode Discuss, Glassdoor, Reddit, HackerNews, and candidate blogs. Every answer comes with source links so you can verify and deep-dive.

**Triggers on:** questions about interview loops, round-specific questions, what to expect at a company for a given role.

**Modes:**
- **Process** — full round-by-round breakdown (format, duration, what each round tests)
- **Questions** — actual questions reported by candidates, categorized, each with a source link

### interview-prep-assistant

Socratic tutor for working through interview questions compiled by `interview-research`. Guides you to think, not just read solutions. Supports LeetCode-style problems, local integration/plumbing problems, and code review.

**Triggers on:** "let's start prep", sharing a compiled question list, "scaffold this question", "review my code".

**Requires:** output from `interview-research` as input.

**Modes:**
- **LeetCode** — structured breakdown, pattern hints, hint ladder (name → explain → skeleton → chunk)
- **Local/Integration** — Maven scaffold (Java 21, OkHttp, Gson), Socratic guidance through parts
- **Code Review** — structured feedback (correctness, pattern fit, one improvement, what's solid)

---

## Installing (Claude Code)

Edit `%USERPROFILE%\.claude\settings.json` and add these two entries:

```json
{
  "extraKnownMarketplaces": {
    "claude-skills": {
      "source": {
        "source": "github",
        "repo": "preeteeshsharma/claude-skills"
      }
    }
  },
  "enabledPlugins": {
    "interview-prep@claude-skills": true
  }
}
```

Then run `/reload-plugins` in Claude Code. Both skills will be available as `interview-prep:interview-research` and `interview-prep:interview-prep-assistant`.

## Installing on Claude Web

1. Open `skills/<skill-name>/SKILL.md`
2. Copy everything after the `---` frontmatter block
3. Create a Claude Project → paste into project instructions

---

## Adding new skills

1. Create `skills/<skill-name>/SKILL.md`
2. Add evals to `skills/<skill-name>/evals/evals.json`
3. Bump the version in `.claude-plugin/plugin.json`
4. Update this README
