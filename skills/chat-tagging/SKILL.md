---
name: chat-tagging
description: Use this skill whenever the user wants to connect multiple separate Claude chats together as parts of the same build piece or project using hashtag-style labels — phrases like "tag this chat #myproject #frontend," "these chats are connected," "link this to #payments-v2," "what's tagged under #myproject," or "show me everything tagged #frontend" all qualify. Also trigger it when the user asks what chats are related to a project, wants a build-piece's status pulled together from several sessions at once, or wants to track both a broad project and its individual pieces via multiple tags on the same chat. Do NOT use this for tagging content inside a single document or email — this is specifically about recording which separate chat sessions belong together and surfacing that link later.
---

# Chat Tagging

## What this is for

A real build (a feature, a skill, a campaign) often isn't one conversation — it's several separate chats: one where the idea got scoped, one where a specialist chat did a piece of the work, one where it got tested. There's no built-in way to know those chats belong together. This skill records that connection explicitly, so the user can later ask "where did we leave the X build" and get an answer that spans every chat involved, not just the one currently open.

A single chat is often part of more than one thing at once — a broad project AND a specific piece of it. So this is a multi-tag system, not a one-tag-per-chat system: a session can carry several hashtag-style labels at once, and the user can look things up at whichever level of granularity they need — everything under a broad tag, or the narrower intersection of two tags together.

This is bookkeeping, not guesswork. Never assume two chats are related because their titles sound similar — the user names the tags and confirms which sessions they apply to. Silently linking the wrong chat is worse than asking one clarifying question.

## Requirements — check these before relying on this skill

This skill needs two things that aren't available in every environment:

1. **A way to list and read other chat sessions** — typically `list_sessions` / `read_transcript` tools (available in Cowork and similar multi-session setups). Without these, the skill can't find or summarize the chats it's meant to link — check they exist before promising this will work.
2. **A persistent, shared file location that survives across separate chat sessions** — a plain per-chat sandbox that resets when the session ends won't work, since the whole point is a shared registry every relevant chat can read and write. This usually means a real filesystem path (via a tool like Desktop Commander, or direct file access) or a cloud document (like a spreadsheet) rather than an in-session scratch file.

If either is missing, say so plainly rather than building a registry that will silently vanish or that only this one chat can ever see.

## The registry (source of truth)

This skill has no fixed file location baked in — pick one with the user the first time it runs, in a place that fits how they work:

1. Check whether the user has already told you a path in this project (e.g., mentioned in a memory file, a README, or earlier in the conversation). Use that if so.
2. Otherwise, ask once: a sensible default is a `chat-tags.json` file at the root of whatever project this build belongs to. Once they confirm a location, remember it for the rest of this and future sessions in the same project (e.g., by noting it in a project memory/config file if one exists) so the user isn't asked again every time.

Sessions carry a list of tags (not the other way around), since one chat can belong to several tags at once:

```json
{
  "sessions": {
    "<session_id>": {
      "title": "...",
      "summary": "...",
      "tags": ["#myproject", "#frontend"],
      "tagged_on": "<ISO date>"
    }
  }
}
```

A single tagging action can apply more than one tag to the same session at once — "tag this chat #myproject #frontend" adds both tags to that session's `tags` array in one write. To find everything under one tag, scan every session whose `tags` array contains it. To find a narrower intersection (e.g., "the myproject's frontend piece specifically"), scan for sessions whose `tags` array contains both `#myproject` and `#frontend`.

**Optional mirror:** if the user has a tracking spreadsheet or document they already use for this project, offer to mirror the same information there — one row per (tag, session) pair, columns like Tag | Chat Title | Session ID | Status Summary | Tagged On. This is genuinely optional; a JSON registry on its own is a complete, valid setup, and plenty of users won't have (or want) a spreadsheet in the loop. If a mirror is set up, the JSON file remains the source of truth if the two ever disagree.

## Tagging chats together

1. Call `list_sessions` to see recent chats — it returns title, session ID, and idle/active state. Users think in titles ("the frontend chat," "the login page one"), not session IDs, so match on title/topic.
2. If more than one session could match what the user described (several chats with similar titles, or a vague description), list the candidates and ask which one they mean. Don't guess — a wrong link silently pollutes every tag it's under and is easy to miss later.
3. Once the sessions are confirmed, call `read_transcript` on each to write a short, concrete status summary — what was actually decided or built, and what's still open. Avoid a vague restatement of the title ("worked on the login page") in favor of something someone could act on ("recommended centered-card layout over split-screen for MVP; not yet implemented in code").
4. If `read_transcript` fails, times out, or a session is still actively running, don't fabricate a summary from the title alone — say so, and ask the user for a one-line status instead.
5. Write all the tags named in the request to that session's `tags` array in the registry (adding to the array if the session's already tracked, not overwriting its existing tags), and update the optional mirror if one's in use.
6. Confirm back to the user what got tagged, showing every tag applied and the actual summaries pulled — not just "done." They should be able to spot a wrong match immediately from what you show them.

## Tags are free-form, but normalize the format

The user names tags however makes sense to them — there's no fixed taxonomy. But free-form only works long-term if the same thing always produces the same tag string, so normalize whatever the user says into a consistent shape before writing it:

- Lowercase, hyphenated within a single tag: "My Project" said as one tag becomes `#my-project`. But prefer separate tags over one compound one when the user is describing two distinct things — "tag this #myproject #frontend" (project + piece) is usually what's meant, not a single fused `#myproject-frontend` tag. If it's ambiguous which they mean, ask.
- Always prefix with `#`: store and display tags as `#myproject`, `#frontend`, `#backend`, etc.
- A natural pattern (and one to suggest if the user is deciding how to organize this): broader, project-level tags plus narrower, piece-level tags applied together on the same session. This isn't a rigid two-level schema the registry enforces — it's just a naming habit that makes broad lookups (project tag → everything) and narrow lookups (project tag + piece tag → just that piece) both useful.
- Before creating a brand-new tag, check the registry for anything close (e.g., user says `#my-project-app` but `#myproject` and `#app` already exist as separate tags) — if there's a near match, ask which they mean rather than silently creating a near-duplicate that splinters the same thing across different tags.

## Looking up tags

When the user asks what's tagged under something, read every session whose `tags` array contains that tag (or, for a multi-tag lookup, every session containing all the named tags) from the registry, and present each linked session's title + summary. If the user wants a fresher read on any of them (their own request, not a default), re-run `read_transcript` on that session and update the registry (and mirror, if in use) before answering.

## Guardrails

- Never link a session to a tag without the user confirming which session they mean, especially when titles are ambiguous.
- Never invent a status summary when the transcript can't be read — say so plainly instead.
- Never create a second registry file or a second mirror location without checking whether one already exists first.
- Never silently collapse two tags the user clearly meant as separate (a project tag and a piece tag) into one fused tag, or vice versa split one tag they meant as a single unit into several — when genuinely unsure which they intend, ask.
