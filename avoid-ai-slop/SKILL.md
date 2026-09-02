---
name: avoid-ai-slop
description: >-
  Write and edit prose documents so they do not read as AI-generated. Grounded
  in Wikipedia's Signs of AI writing catalog and peer-reviewed vocabulary
  research: replace generic significance claims with specifics, vary rhythm,
  cut AI-tell words, stock phrases, and chatbot residue, then run a revision
  pass. Use when producing or revising any prose deliverable — documents,
  reports, READMEs, documentation, emails, blog posts, proposals,
  announcements, release notes, marketing or UI copy, essays, cover letters,
  wiki pages, PR descriptions — or when the user says writing sounds like AI,
  robotic, or generic, or asks to humanize text or remove slop.
---

# Avoid AI Slop

AI slop is prose that regresses to the mean: generic claims, inflated significance, uniform rhythm, wording that could appear in any document on any topic. Readers recognize the texture before they can name it. Fix it at three levels, in order: substance, structure, surface. Surface fixes alone (deleting "delve" and em dashes) leave text that still reads as AI.

If the user's own voice is available — surrounding text in the document, prior emails, a house style — match it. Consistency with the user's voice beats any rule below.

## Substance

The strongest tell is content that survives being pasted into a different document unchanged.

- State facts, not significance. Cut claims that something "marks a pivotal moment", "underscores the importance", "reflects broader trends", or "stands as a testament". If the subject matters, the specific fact shows it. The core failure: the subject gets more exaggerated while the prose gets less specific.
- Cut trailing participle analysis — sentence-final "-ing" clauses that assert meaning ("...highlighting the team's commitment to quality", "...ensuring a seamless experience"). End the sentence at the fact.
- No vague authority: "experts argue", "observers have noted", "studies show", "industry reports". Name the person, study, or outlet, or drop the claim.
- No throat-clearing openers ("In today's fast-paced world...") and no closing platitudes about challenges or future prospects. Start with the point; stop when done. No "Conclusion" or "In summary" section that restates the document.
- Prefer the exact number, name, date, or example over an intensifier: "cut load time from 3.1s to 0.8s", not "significantly enhanced performance".
- Describe, don't sell. Default LLM tone drifts toward press-release ("boasts", "state-of-the-art", "commitment to excellence"). Even in marketing copy, specifics persuade more than superlatives.
- Commit to a point of view where the document allows one: a recommendation, not a catalog of considerations.

## Structure

- Vary sentence length. Uniform 18–24-word sentences are a measurable AI fingerprint. Short sentences are fine. So are fragments, where register permits.
- Don't force the rule of three. Two examples? List two. Four? Four. Don't balance every sentence into neat parallel clauses.
- Ration negative parallelism — "not just X, but Y", "It's not X; it's Y" — to at most one per document, ideally zero.
- Vary openers. If "Additionally", "This", "These", or "By" starts several sentences, rewrite. Don't chain formal connectives (Furthermore / Moreover / Additionally).
- Format only what needs formatting:
  - Headings in sentence case, only where they organize substantial content. Not one per paragraph, and not formula pairs ("Challenges and Opportunities", "Awards and Recognition").
  - Bullets for genuinely enumerable items — not prose chopped into "**Term**: description" lines.
  - Bold rarely; never every key phrase.
  - No emoji decoration, no horizontal rules between sections, no table for three facts that fit in a sentence.

## Surface

- Tell words, verified to spike in LLM output. One is fine if it is the right word; a cluster is the tell.
  - Verbs: delve, underscore, showcase, highlight, boast, foster, leverage, harness, navigate, elevate, bolster, garner, streamline, emphasize, enhance.
  - Adjectives: crucial, pivotal, key, vital, robust, seamless, vibrant, meticulous, intricate, comprehensive, multifaceted, enduring, groundbreaking, renowned, invaluable, transformative.
  - Abstract nouns: landscape, realm, tapestry, testament, journey, synergy, interplay, deep dive.
  - Stock phrases: "it's important to note", "plays a vital role in", "serves as a", "stands as a", "in the heart of", "nestled", "rich cultural heritage", "navigate the complexities of", "a wide/diverse range of", "valuable insights", "in conclusion".
- Prefer plain copulas and verbs: is/has over "serves as" / "features" / "offers" / "represents"; use over utilize; wrote over authored; died over passed away. Post-2022 text shows a measured drop in is/are, so the ornate substitutes read as AI.
- Em dashes: keep the ones doing real work; replace the rest with commas, colons, or parentheses. Never as recurring emphatic punctuation.
- Zero chatbot residue: "I hope this helps", "Certainly!", "Would you like...", "Here is a...", knowledge-cutoff disclaimers ("as of my last update", "not widely documented"), "as an AI", placeholders ("[Your Name]", "2025-XX-XX"), leftover citation artifacts (contentReference, oaicite, turn0search, "[cite: 1]").

## Don't overcorrect

- The word list is a detection signal, not a ban list. Mechanically deleting tell words from generic, evenly paced prose yields generic, evenly paced prose.
- Don't fake humanity: no injected typos, forced slang, or random informality. Match the register the user asked for.
- Don't pad. Unrequested length is itself a tell. A document that says its thing and stops reads as written by a person.

## Revision pass

After drafting, before delivering:

1. Search the draft for the surface tells above; rewrite any cluster.
2. Find sentence-final "-ing" clauses; cut them or replace with a concrete fact.
3. Read the first word of each sentence and paragraph; break up repeats.
4. Scan sentence lengths; if uniform, split some and merge others.
5. Delete any sentence that would fit unchanged in a different document on another topic.
6. Delete summary restatements and future-prospects endings.
7. Justify every heading, bullet, bold, and table; if it adds no structure the prose lacks, remove it.

## Example

Before:

> The v2.4 release marks a pivotal milestone in our ongoing journey to deliver a seamless user experience. Additionally, this release showcases robust improvements to the dashboard, underscoring our enduring commitment to innovation.

After:

> v2.4 cuts dashboard load time from 3.1s to 0.8s and adds CSV export, the most-requested feature since March. Upgrading requires no migration.

## Sources

The tells drift as models change; re-verify against the living catalog when in doubt.

- [Wikipedia: Signs of AI writing](https://en.wikipedia.org/wiki/Wikipedia:Signs_of_AI_writing) — evidence-based catalog maintained by WikiProject AI Cleanup.
- [Kobak et al. 2025, Science Advances](https://www.science.org/doi/10.1126/sciadv.adt3813) — peer-reviewed measurement of LLM-preferred vocabulary ("delves" at 28x expected frequency).
