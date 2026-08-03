# fnf-claude-master-cookbook

Open-source, reusable Claude skills and patterns — extracted from real work on
Funny not Funny's own build (FNF's own tooling stays in its private repos;
what lands here is the generic, project-agnostic version of anything worth
sharing).

## Skills

- [`chat-tagging`](skills/chat-tagging/SKILL.md) — link related Claude chat
  sessions together with hashtag-style labels (e.g. `#myproject #frontend`)
  so a build that spans several conversations can be looked up as one thing
  later. Config-driven: no fixed registry location or project-specific
  values baked in, so it works the same way in any project.
