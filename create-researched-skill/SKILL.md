---
name: create-researched-skill
description: Create an Agent Skill grounded in fresh web research instead of only built-in knowledge. Researches the web for quality information on the skill's topic (biased toward recent sources when the topic evolves), assesses the information for quality, then combines the distilled findings with existing knowledge to author the skill. Use when the user runs /create-researched-skill or asks to create a skill on a topic where current or specialized information matters — new or changing APIs, tools, frameworks, standards, or best practices.
disable-model-invocation: true
---

# Create Researched Skill

Create a new Agent Skill whose contents are grounded in verified web research combined with existing knowledge.

For requirements gathering, skill structure, frontmatter, descriptions, storage locations, and authoring conventions, read and follow the `create-skill` skill available in Cursor. This skill references `create-skill` by name and intentionally does not reproduce its contents, due to intellectual property and terms of use concerns. If `create-skill` is unavailable, follow the official Cursor Agent Skills documentation instead.

## Workflow

First, read `create-skill` and follow its requirements-gathering guidance to establish the new skill's purpose, triggers, and storage location. Then:

### 1. Research the web

- List what the skill needs from external sources: version-specific APIs, commands, configuration, current best practices, domain facts. Mark the load-bearing claims — facts the skill breaks without — as these must be verified in step 2.
- Run targeted searches per research need; prefer several narrow queries over one broad one.
- Bias toward recent information when appropriate — for anything that evolves (APIs, tools, frameworks, best practices), include the current year in queries and check publication or last-updated dates. For stable fundamentals, older authoritative sources are fine.
- Prefer primary sources: official documentation, release notes, specifications, maintainer announcements. Use reputable secondary sources (engineering blogs, well-regarded community answers) to fill gaps.
- Fetch full pages when search summaries are too thin to act on.

### 2. Assess the information for quality

- Weigh each source by authority (official > maintainer > community), recency, and consistency with other sources.
- Cross-verify load-bearing claims against a second independent source or existing knowledge; commands, API signatures, and version numbers deserve extra scrutiny.
- Resolve conflicts by preferring the more recent and more authoritative source. If a conflict remains, verify empirically when cheap (run the command, check the package registry) or record the uncertainty explicitly in the skill.
- Discard content that is undated and unverifiable, SEO-farmed, or contradicted by primary sources.

### 3. Create the skill

- Distill the surviving research to the minimum an agent would not already know; leverage existing knowledge for everything else.
- Author the skill following `create-skill` conventions end to end, including its final verification checklist.
- Phrase researched facts that are likely to change (versions, endpoints, defaults) so they age well, and link the primary source where future re-verification would help.
- Report to the user which key facts came from research and list the main sources used.

## Example

User: "/create-researched-skill for deploying to Cloudflare Workers"

The agent reads `create-skill` and confirms scope and location; searches for current Wrangler CLI documentation and release notes using the current year; verifies deploy commands and `wrangler.toml` fields against official docs; discards a stale tutorial contradicting them; then authors the skill from the verified commands plus existing knowledge, linking the official docs, and reports the sources used.
