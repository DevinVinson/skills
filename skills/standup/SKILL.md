---
name: standup
description: Facilitates daily standup check-ins using markdown daily notes, carries forward relevant unfinished tasks, and updates the current day's note. Use when the user asks for a standup or daily check-in, asks what's on their plate, or wants help reviewing and updating today's work.
---

# Standup

Facilitate daily standup check-ins using markdown daily notes in a user-provided or discovered daily note directory. Many users keep these in an Obsidian-style vault, but do not hardcode a single personal path. Use date-named files in the format `YYYY-MM-DD.md`.

## Setup and Discovery

Before editing anything, determine the daily note directory in this order:

1. If the user gave a path, use it.
2. Otherwise, look for likely daily note directories relative to the current workspace, preferring `Daily/`, `Daily Notes/`, or `Notes/Daily/`.
3. If multiple candidates exist or none are obvious, ask the user which directory to use.

## Expected Structure

When creating a new note, prefer this structure:

```markdown
# Daily Notes - YYYY-MM-DD

## Todo

## Notes

## Agent Notes
```

Treat `## Agent Notes` as optional. Create it only when a short factual session log is useful.

When reading an existing note, be tolerant of variations. Do not reformat older files unless needed. If a note lacks a `## Todo` section, use the closest equivalent section such as `## Today`, or fall back to top-level checklist items while preserving the file's existing content.

## Standup Flow

### 1. Resolve the Target Date

Use the user's requested date if they specify one. Otherwise, use the system's current local date. If the user refers to a different timezone or the date is ambiguous, ask before proceeding.

### 2. Check the Target-Day File

If the file exists, read it first and continue with an update-oriented standup.

If the file does not exist, create it and then find the most recent prior daily note.

### 3. Find the Previous Daily Note

Do not only check "yesterday." Search the daily note directory for the latest `YYYY-MM-DD.md` that is earlier than the target date.

- This must handle Mondays by finding Friday's note when weekend notes do not exist.
- This must also handle skipped days, vacations, and irregular standup usage.
- When the previous note is older than one day, refer to it as the "last note from YYYY-MM-DD," not "yesterday."

### 4. Extract Carryover Safely

Only extract carryover tasks from the main actionable task section of the previous note:

- Prefer `## Todo`
- Otherwise fall back to `## Today`
- Otherwise use top-level checklist items if the file is legacy or unstructured

Stop at the next same-level heading. Do not pull unchecked items from `## Notes`, `## Agent Notes`, audit sections, future-planning sections, transcript summaries, or any other non-primary section.

Carry forward only incomplete checklist items matching `- [ ] ...`.

- Deduplicate repeated items with identical or near-identical wording.
- Preserve wording that helps disambiguate similar tasks.
- If the previous note contains a large backlog or stale tasks, summarize it and ask the user which items still matter instead of dumping everything into today's note.

### 5. Create the New Note

Initialize the new note with the standard structure.

If `## Agent Notes` is present or useful, add a brief entry such as:

- Started standup for YYYY-MM-DD.
- Previous daily note found: YYYY-MM-DD.

If no earlier dated file exists, record that no previous daily note was found.

Do not automatically copy a large backlog of unchecked tasks into today's `## Todo` without confirming with the user first.

### 6. Interactive Check-in

**For new files:**

1. If carryover exists, present a concise summary.
   > "Good morning! I found 3 open items from your last daily note on 2026-04-29. Are these still active today?"
2. Ask what the user is working on today.
3. Add confirmed tasks under `## Todo`.
4. Put context, blockers, and meeting takeaways under `## Notes`.

**For existing files:**

1. Summarize today's `## Todo` section only.
2. Ask focused questions about incomplete items.
3. Update items based on the user's response.

### 7. Update Behavior

- Mark completed items as `- [x]`.
- Keep still-open items as `- [ ]`.
- Remove items only when the user confirms they are no longer relevant.
- Save changes after each meaningful update.
- Keep `## Agent Notes` short and factual when present.

### 8. Completion

Listen for completion signals such as:

- "all done"
- "that's it"
- "nothing else"
- "I'm good"
- "that's all"

When detected:

1. Append a short summary to `## Agent Notes` if that section exists or is useful.
2. End with a positive closing message.

## Interaction Guidelines

**Do:**
- Be specific about outstanding items.
- Ask about ambiguous or stale tasks.
- Treat Monday and post-gap standups as a continuation from the last existing note.
- Keep carryover focused and deduplicated.
- Preserve the user's existing note style when reading legacy files.

**Don't:**
- Say there is "no today file" and stop.
- Assume the previous note is always yesterday.
- Scan the entire previous file for unchecked boxes.
- Pull tasks from audit or reference sections.
- Flood the user with a huge backlog without triage.
- Rewrite older notes just to match the template.

## Example Openings

**New day after a weekend or gap:**
> "Good morning! I created today's daily note and found your last note from Friday, YYYY-MM-DD. You still had 3 open todo items in the main Todo section. Which of these are still active today?"

**Returning to an existing day:**
> "Welcome back! I see 2 open items and 4 completed items in today's Todo section. What changed since the last check-in?"
