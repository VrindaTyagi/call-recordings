# Call Recordings — Schema & Sync Runbook

Operating manual for Claude (local or cloud) when working with this vault. Read this file in full before touching anything here. It is the single source of truth for structure, conventions, and the sync procedure, so a cloud agent with only this checkout can run the sync correctly.

## Purpose

Archive Vrinda Tyagi's Granola meeting notes (AI summary + full verbatim transcript) into git, and build a context graph of People / Projects / Jira tickets that Obsidian's backlinks and graph view can render, so Granola can be cleaned up once a meeting is archived here. All timestamps use +05:30 (IST).

Granola folders are **not** used for organization. Every meeting is classified by its content (title, then Granola summary) using `_meta/project-map.yml`, and by a meeting type derived from its title and attendees.

## Structure

```
call-recordings/
├── AGENT.md / SCHEMA.md / Welcome.md
├── _meta/
│   ├── sync-state.json      # watermark + synced meeting IDs (idempotency manifest)
│   ├── project-map.yml      # project slug -> keywords (content-based classification)
│   ├── jira-project-map.yml # Jira project prefix -> ticket type
│   ├── people-context.md    # optional colleague directory for People identity fields
│   └── log.md               # append-only sync run log
├── People/   index.md + <slug>.md
├── Projects/ index.md + <slug>.md
├── Jira/     index.md + <KEY>.md
└── Meetings/YYYY-MM/YYYY-MM-DD-<slugified-title>.md   # one file per occurrence
```

Meeting files never move once written. Grouping happens through frontmatter links (`project:`, `attendees:`, `jira:`), which Obsidian renders as backlinks and graph edges. Reclassifying a meeting means editing its `project:` field, not moving the file.

Slugified title: lowercase, replace any run of characters outside `[a-z0-9]` (including non-Latin script, e.g. Hindi) with `-`, trim `-`, max 60 chars; if nothing remains, use `untitled`. If the resulting path already exists for a different `meeting_id`, append `-2`, `-3`, ...

## Page conventions

### Meeting page
```yaml
---
meeting_id: "<granola-uuid>"
title: "<title, exactly as in Granola>"
date: "YYYY-MM-DD"
time: "HH:MM +05:30"
source: granola
granola_url: "https://notes.granola.ai/d/<uuid>"
type: 1on1                       # 1on1 | standup | team-meeting | working-session | solo
attendees:
  - "[[People/<slug>]]"
project: "[[Projects/<slug>]]"   # or "unclassified"; omit the key for people-only meetings (see Classification)
jira: []                         # list of "[[Jira/<KEY>]]"; empty unless a ticket was discussed
synced_at: "<ISO 8601 +05:30>"
tags: [meeting]
---

## Notes / Summary
<Granola AI summary, verbatim>

## Full Transcript
<details>
<summary>Expand transcript</summary>

<verbatim transcript, exact speaker labels>

</details>
```
Rules: one file per occurrence; notes and transcript together. Transcript verbatim: no summarizing, translating or truncating (Hindi / Hinglish stays as spoken). If a transcript is unavailable, say so; never fabricate. Notes come only from Granola's summary.

### Person page (`People/<slug>.md`)
Frontmatter: name, email, team, role, manager, tags: [person], created, updated. Body: `# Name`, `## Identity` (email/team/role, "(unknown)" when not known; refreshed from `_meta/people-context.md` by email match and Granola attendee data), then the protected notes region:
```
<!-- notes:start -->
## Notes

<!-- notes:end -->
```

### Project page (`Projects/<slug>.md`)
Frontmatter: name, slug, description, tags: [project], created, updated. Body: `# Name`, `## Identity` (description, and "Classified by keywords in `_meta/project-map.yml`"), then the protected notes region. Create a project page the first time a meeting is classified into it, using `name` and `description` from `project-map.yml`.

### Jira page (`Jira/<KEY>.md`)
```yaml
---
key: "ABC-123"
jira_url: "https://mindtickle.atlassian.net/browse/ABC-123"
type: "deliverable"     # production-issue | deliverable | unclassified
summary: ""             # from the Jira lookup
issue_type: ""          # Epic, Story, Bug, ... from the lookup
parent: ""              # "[[Jira/<PARENT-KEY>]]" from the lookup, or ""
verified: true          # false if the lookup failed; retried later
tags: [jira]
created: "YYYY-MM-DD"
updated: "YYYY-MM-DD"
---
```
Body: `# KEY`, `## Identity` (Jira link, Type, Parent), then the protected notes region.

**Hard rule for People/Projects/Jira pages:** everything above `<!-- notes:start -->` is routine-owned (field updates only). Everything from `<!-- notes:start -->` on is owner-owned; the sync never writes, reorders or deletes there.

### Index pages
Regenerated every run; never hand-edit.
- People: alphabetical `- [[slug|Display Name]]`.
- Projects: alphabetical `- [[slug|Display Name]] — N meetings`, followed by a final `## Unclassified` section listing meetings with `project: "unclassified"` as `- [[Meetings/YYYY-MM/<file>|<title>]] (YYYY-MM-DD)`, or `_none_`.
- Jira: grouped under `## Production issues`, `## Deliverables`, `## Unclassified` (by `type:`), `_none_` for an empty group; each line `- [[KEY]] — <summary>`; a ticket whose parent is in the same group is nested (two-space indent) under it.

## Classification

Granola folders are ignored for classification. Every meeting captured by the owner, or listing the owner as a participant, is a sync candidate.

### Project (`_meta/project-map.yml`)
```yaml
<project-slug>:
  name: "<Display Name>"
  description: "<one line>"
  keywords: ["<lowercase phrase>", ...]
```
Matching is case-insensitive substring on whole phrases.
1. Match the **title** against every project's keywords. Exactly one project matches → use it.
2. No title match → match the **Granola summary** (not the transcript). Exactly one project matches → use it. If several match, pick the one with the most distinct keyword hits; on a tie, use `unclassified`.
3. Still no match:
   - If `type` is `1on1` (e.g. "Neha / Vrinda Daily Sync"), omit `project:` — it is a **people-only** meeting, linked through its People pages. This is not an error.
   - Otherwise set `project: "unclassified"` and report it under "Unclassified meetings" with a suggested project (an existing slug, or a proposed new slug + keywords).
4. Never create a new project or edit `project-map.yml` during a sync. The owner adds keywords; meetings already written as `unclassified` are re-checked each run (title + summary only, from the existing page) and upgraded if a keyword now matches. Never overwrite a non-`unclassified` `project:`.

### Type
Decide in this order; first rule that matches wins.
1. `solo` — the owner is the only known participant.
2. `team-meeting` — title contains "team meeting", "all-hands", "all hands", "town hall".
3. `standup` — title contains "stand-up", "standup", "scrum".
4. `1on1` — exactly two participants (owner + one), or a title of the form "X / Y", "X <> Y", "X/Y", "X & Y" naming the owner and one other person, or containing "1:1" / "1on1" / "daily sync" / "weekly sync" with two participants.
5. `working-session` — everything else (walkthroughs, brain dumps, reviews, context sessions, finalising discussions).

### Interview safety net (all meetings)
Check title + summary for: interview, round (e.g. "LLD Round"), debrief, panel discussion, screening call, culture fit, candidate, hiring; or 1–2 attendees from outside Mindtickle and a title that reads like an interview stage. On a match, still sync it, set `project: "unclassified"`, and list it under "Possibly interview-related — please review" in the report. Never auto-delete.

## Jira detection and classification

### Detection (which keys a meeting gets)
1. Keys in the **title** matching `[A-Z][A-Z0-9]+-\d+`: always tagged.
2. Keys in the **Granola notes/summary**: tagged only when the meeting has a discussion, a decision change, or a call-out on that ticket. Keys merely read out (board walk-through, status list) are not tagged.
3. Otherwise `jira: []`. Don't invent one.
Never mine the transcript (speech-to-text garbles spoken keys; tokens like GPT-4 match the pattern). Recurring meetings are tagged per occurrence; no per-series mapping. A key in both title and notes is listed once. A meeting's existing non-empty `jira:` is never overwritten.

### Jira lookup
Read-only via the Atlassian connector: only `getJiraIssue` (and `getAccessibleAtlassianResources` for the cloudId). Never create, edit, comment, transition or link.
- Look up only keys with no page yet, or a page with `verified: false`.
- At most 10 lookups per meeting: title keys first, then notes keys by how substantive the discussion was; report overflow under "Jira cap overflow".
- Record current key (if the issue moved, use the returned key), summary, issue_type, parent. Take the parent from the child's `parent` field (no extra call). Never record status.
- Create a page for a new parent (its own `parent: ""`). Meetings are not tagged with the parent.
- On failure: a title key is still tagged with `verified: false` and retried next run; a notes key is skipped this run. List both under "Unverified Jira tickets".

### Classification (`type:`)
1. `_meta/jira-project-map.yml` lists the key's project prefix → that type.
2. A parent page created from a child lookup → `deliverable` if Epic, else `unclassified`.
3. Otherwise from the meeting context where the key first appears: `production-issue` (escalation, incident, outage, bug, customer-reported, RCA, hotfix), `deliverable` (epic, story, feature, design/scope review, planning, migration), `unclassified` when ambiguous (report it).
4. Never overwrite an existing `type:` (except upgrading `unclassified` with clear context), nor a verified page's summary/issue_type/parent.

## Sync operation

1. **Load state** from `_meta/sync-state.json`. If `last_sync_at` is null, window start = now − `backfill_window_days`; else `last_sync_at` − 1 day (overlap buffer). Window end = now.
   **Per-run cap: 5 meetings.** Sort not-yet-synced meetings by start time, oldest first, process the first 5; the rest are backlog for the next run. Flagged meetings count toward the cap. No subagents; process sequentially.
2. **Enumerate.** `list_meetings` with a custom range covering the window and `involvement: {captured_by_me: true, listed_as_participant: true}`. Do not query by folder. Drop IDs already in `synced_meeting_ids`.
3. **Flag off-topic** recordings (transcript clearly unrelated to the title and no work content): don't file; report under "Flagged"; don't add the ID. Run the interview safety net.
4. **Pull content.** `get_meetings` (batches of ≤10) for notes/attendees; `get_meeting_transcript` per meeting. If access is denied, note it on the page.
5. **Classify and write.** Type, then project (see Classification); resolve/create People pages (slug = kebab-case full name, skip Vrinda Tyagi); create the Project page if new; determine `jira:`, look up new keys, create Jira pages; write the meeting page. A transcript saved to a file because it was too large: copy it in with a script (e.g. python), never retype. Never touch a notes region. Add the ID to `synced_meeting_ids`. Re-check existing `unclassified` meetings against the current `project-map.yml`.
6. **Regenerate indexes.**
7. **Update state.** No backlog: `last_sync_at = now`. Backlog remains: `last_sync_at` = start time of the oldest backlog meeting (never now). ISO 8601 with +05:30.
8. **Log.** Append to `_meta/log.md`: date, window, counts (synced / flagged / unclassified meetings / unclassified jira / unverified jira / backlog remaining), and which meetings are now safe to delete from Granola.
9. **Commit and push.** `git add -A`; commit `sync: YYYY-MM-DD — N meetings`; `git push origin HEAD:main` (the cloud checkout can be a detached HEAD). On failure, keep the commit and report the error.
10. **Report.** Meetings synced (paths, with type and project), flagged (why), possibly-interview-related, unclassified meetings (with suggested project), meetings upgraded from unclassified, new Jira tickets with type and parent, unclassified jira, unverified jira, Jira cap overflow, backlog remaining (count + oldest date, or "none"), safe-to-delete list.

## Query protocol
To answer a question from the vault: read this file, then the index pages, then the relevant pages and the meetings they link, and answer citing [[wikilinks]].
