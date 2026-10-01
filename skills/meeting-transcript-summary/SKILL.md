---
name: meeting-transcript-summary
description: This skill should be used when the user asks to "summarize this meeting transcript", "turn this transcript into meeting notes", "create a Slack-ready meeting summary", "pull out key meeting bullets", or wants an executive summary from meeting notes or call transcripts.
---

# Meeting Transcript Summary

Turn raw meeting transcripts into concise, shareable summaries.

## Core Requirements

- Start every output with an **Executive Summary** section.
- Follow with **Key Points** containing **4-5 bullets maximum**.
- Cover the main topics, decisions, blockers, and next steps rather than speaker-by-speaker detail.
- Write for Slack: keep language crisp, direct, and easy to scan.
- If the transcript is partial or noisy, summarize what is clear and briefly note uncertainty.
- When the user invokes **`/meeting-summary`** without a transcript file path, stop and ask for the file location before summarizing.

## Workflow

1. Confirm the transcript source first. If **`/meeting-summary`** is used without a file path, ask for the transcript file location and wait.
2. Scan the transcript for repeated themes, decisions, action items, blockers, and open questions.
3. Group overlapping discussion into a few main topics.
4. Prioritize the topics that mattered most to the meeting outcome.
5. Write a short executive summary that explains the overall direction, outcome, or state of the discussion.
6. Write up to five bullets covering the main topics.
7. Add an **Action Items** section only when concrete follow-ups are explicitly discussed.
8. Omit filler, timestamps, side conversations, and transcript artifacts unless they materially affect the summary.

## Output Template

Use this default structure unless the user requests a different format:

```markdown
**Executive Summary**
[2-4 sentences summarizing the meeting outcome, status, or direction.]

**Key Points**
- [Main topic or decision]
- [Main topic or decision]
- [Main topic or decision]
- [Main topic or decision]
- [Optional fifth bullet only if needed]

**Action Items**
- [Only include when explicit follow-ups are present]
```

## Summarization Rules

- Prefer outcomes over chronology.
- Merge repetitive discussion into a single bullet.
- Keep each bullet focused on one main idea.
- Mention unresolved questions or disagreements only when they are important.
- Use fewer than five bullets if the transcript only contains a few real themes.
- Preserve nuance, but avoid burying the reader in caveats.
- When the user asks for meeting notes rather than Slack copy, keep the same structure unless a different template is requested.

## Good Trigger Examples

This skill fits prompts like:

- "Summarize this meeting transcript for Slack."
- "Turn these notes into an executive summary and key bullets."
- "Create shareable meeting notes from this transcript."
- "What are the key takeaways from this call?"
- "Make this meeting summary ready to send to leadership."
