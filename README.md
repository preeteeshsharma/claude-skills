# claude-skills

Personal Claude Code skills library. Backup and cross-device sync for custom skills.

## Skills

### interview-research

Researches real-world interview processes and questions for specific companies and roles using live web searches. Sources from Blind, LeetCode Discuss, Glassdoor, Reddit, HackerNews, and candidate blogs. Every answer comes with source links so you can verify and deep-dive.

**Triggers on:** questions about interview loops, round-specific questions, what to expect at a company for a given role.

**Modes:**
- **Process** — full round-by-round breakdown (format, duration, what each round tests)
- **Questions** — actual questions reported by candidates, categorized, each with a source link

## Installing on a new machine (Claude Code)

1. Clone this repo
2. Copy the skill into your Claude plugins directory:
   ```
   xcopy /E /I skills\interview-research "%USERPROFILE%\.claude\plugins\cache\local\interview-research\1.0.0\skills\interview-research"
   ```
3. Register the plugin by adding this entry to `%USERPROFILE%\.claude\plugins\installed_plugins.json` under `"plugins"`:
   ```json
   "interview-research@local": [
     {
       "scope": "user",
       "installPath": "C:\\Users\\<YOUR_USERNAME>\\.claude\\plugins\\cache\\local\\interview-research\\1.0.0",
       "version": "1.0.0",
       "installedAt": "2026-03-08T07:00:00.000Z",
       "lastUpdated": "2026-03-08T07:00:00.000Z",
       "gitCommitSha": ""
     }
   ]
   ```
   *(Replace `<YOUR_USERNAME>` with your Windows username)*
4. Run `/reload-plugins` in Claude Code

## Installing on Claude Web

1. Open the `skills/interview-research/SKILL.md` file
2. Copy everything after the `---` frontmatter block
3. Create a Claude Project → paste into project instructions

## Adding new skills

1. Create `skills/<skill-name>/SKILL.md`
2. Add evals to `skills/<skill-name>/evals/evals.json`
3. Package with skill-creator if needed
4. Update this README
