# Skills Usage Audit & Migration Recommendations

_Audit date: 2026-06-03_
_Conversations scanned: 47 (all available)_
_Date range: 2026-05-14 to 2026-06-03_

## Summary

This audit scanned all 47 recent OpenHands Cloud conversations to identify which skills are actively invoked (`invoke_skill` calls) and which are loaded most frequently. The goal is to identify skills worth migrating to this personal skills repo for customization and maintenance.

## Current Skills in This Repo

| Skill | Description |
|---|---|
| `standup` | Daily standup and task carryover workflow |
| `openhands-agent-server-ui` | Reference for building browser UIs against OpenHands agent-server |
| `openhands-design` | Reference for implementing the OpenHands design system |
| `use-git-worktrees` | Guide for git worktree workflows |

## Skills Actually Invoked (invoke_skill calls)

These skills were explicitly called via `invoke_skill` during conversations:

| Skill | Invocations | Conversations | Use Cases |
|---|---|---|---|
| **notion** | 5 | 5 | Release highlights, daily automation, Notion page management |
| **openhands-api** | 2 | 2 | Browser panel implementation, daily Notion automation |
| **openhands-automation** | 1 | 1 | Setting up daily Notion release checks |
| **code-review** | 1 | 1 | PR review (agent-canvas bugfix PR #654) |
| **custom-codereview-guide** | 1 | 1 | Custom code review standards (PR #654) |

> **Note:** 10 additional `invoke_skill` calls had unresolved skill names ("unknown") across 7 conversations. These likely correspond to auto-triggered skill activations where the skill name wasn't captured in the action parameters.

## Non-Public Skills Frequently Loaded

These skills are NOT part of the standard OpenHands public skill set. They come from specific repositories (e.g., OpenHands/OpenHands, OpenHands/agent-canvas) and were loaded based on the repository context:

| Skill | Loaded in | Source Context |
|---|---|---|
| `agents` | 20/47 (43%) | Repo-level skill from agent-canvas/OpenHands repos |
| `custom-codereview-guide` | 18/47 (38%) | Custom code review standards |
| `gitlab` | 13/47 (28%) | GitLab integration (from OpenHands repos) |
| `feature-release-rollout` | 9/47 (19%) | Release management |
| `manage-evals` | 9/47 (19%) | Evaluation management |
| `cross-repo-testing` | 9/47 (19%) | Cross-repo test coordination |
| `sdk-release` | 9/47 (19%) | SDK release process |
| `design-principles` | 9/47 (19%) | Design principles reference |
| `debug-test-examples-workflow` | 9/47 (19%) | Test debugging workflow |
| `write-behavior-test` | 9/47 (19%) | Behavior test authoring |
| `run-eval` | 9/47 (19%) | Evaluation runner |

## Public Skills (Auto-Loaded in All Conversations)

These 38 skills are part of the OpenHands public skill set and load automatically in every conversation. They do not need migration — they are always available:

<details>
<summary>Full list of auto-loaded public skills (38)</summary>

`add-javadoc`, `add-skill`, `agent-creator`, `agent-sdk-builder`, `azure-devops`, `bitbucket`, `code-review`, `code-simplifier`, `datadog`, `deno`, `discord`, `docker`, `evidence-based-citations`, `flarglebargle`, `frontend-design`, `github`, `github-pr-review`, `github-repo-monitor`, `iterate`, `jupyter`, `kubernetes`, `learn-from-code-review`, `linear`, `npm`, `notion`, `openhands-api`, `openhands-automation`, `openhands-sdk`, `pdflatex`, `prd`, `qa-changes`, `release-notes`, `security`, `skill-creator`, `slack-channel-monitor`, `spark-version-upgrade`, `ssh`, `swift-linux`, `theme-factory`, `uv`, `vercel`

</details>

---

## Migration Recommendations

### 🔴 High Priority — Migrate to this repo

Skills you actively invoke and would benefit from personal customization:

#### 1. `notion` (custom version)
- **Why:** Most-invoked skill (5 calls across 5 conversations). Used for release highlights tracking, daily automation, and Notion page management.
- **Action:** Create a personalized `notion` skill with your specific Notion workspace patterns, page templates, and automation recipes. The public skill covers generic Notion API usage; a custom version could include your release-tracking workflow and daily check patterns.

#### 2. `openhands-api` (custom version)
- **Why:** Second most-invoked skill (2 calls). Used for delegating conversations and browser panel work.
- **Action:** Create a customized version with your frequently-used API patterns, preferred conversation delegation settings, and common workflows.

#### 3. `openhands-automation`
- **Why:** Invoked for setting up daily Notion checks. Combined with `notion` and `openhands-api`, this forms your automation workflow stack.
- **Action:** Create a personal version documenting your specific automation patterns (daily Notion checks, release roundups, etc.).

### 🟡 Medium Priority — Consider migrating

Skills that are loaded frequently and relevant to your workflows but not yet explicitly invoked often:

#### 4. `code-review` + `custom-codereview-guide`
- **Why:** `custom-codereview-guide` is loaded in 38% of conversations (from repos). You invoked both during a PR review. Having your own code review standards in this repo ensures consistency across all conversations.
- **Action:** Create a personal `code-review` skill that embeds your custom review guide, so it's available regardless of which repo you're working in.

#### 5. `frontend-design`
- **Why:** Loaded in 100% of conversations (public skill). Your work heavily involves agent-canvas UI development and browser panel implementation.
- **Action:** Consider a custom version that includes your preferred UI patterns, component libraries, and design tokens specific to the OpenHands ecosystem.

### 🟢 Lower Priority — Monitor usage

These are commonly loaded but may not need personal versions yet:

#### 6. `github` / `github-pr-review`
- **Why:** Loaded in all conversations. You work extensively with GitHub repos. The public versions are comprehensive, but a custom version could encode your PR workflow preferences.
- **Action:** Monitor whether you find yourself repeatedly providing the same GitHub context. If so, create a custom version.

#### 7. `skill-creator`
- **Why:** Loaded in all conversations. You create and maintain skills regularly.
- **Action:** Keep using the public version unless you develop skill-creation patterns worth codifying.

#### 8. `iterate`
- **Why:** Loaded in all conversations. Useful for PR iteration workflows.
- **Action:** The public version is likely sufficient. Consider customizing only if your iteration workflow has repo-specific patterns.

---

## Workflow Patterns Identified

From the conversation titles and skill usage, these are your primary workflow patterns:

1. **Notion-based release tracking** — Regular Notion page updates for release highlights and weekly roundups (notion + openhands-api + openhands-automation)
2. **Agent-canvas development** — UI implementation, bug investigation, and PR review (frontend-design + code-review + github)
3. **Skill syncing & maintenance** — Keeping skill docs up to date with source repos (openhands-agent-server-ui is already in this repo)
4. **Code review** — PR reviews with custom standards (code-review + custom-codereview-guide)

## Raw Data

<details>
<summary>Conversation list with skill invocations</summary>

| Date | Conversation | Skills Invoked |
|---|---|---|
| 2026-05-30 | 💄 Browser Panel Implementation in Agent Canvas | openhands-api |
| 2026-05-21 | 🔧 Update Notion Release Highlights check | notion |
| 2026-05-21 | ✅ Automate Daily Notion Release Check | notion, openhands-api, openhands-automation |
| 2026-05-21 | ✅ Verify Notion page and add Hello World | notion |
| 2026-05-21 | Conversation 044c6… | notion |
| 2026-05-20 | 🐛 Review agent-canvas bugfix PR #654 | code-review, custom-codereview-guide |
| 2026-05-20 | 📝 Weekly OpenHands Release Roundup Draft | notion |

All other conversations (40/47) had no explicit `invoke_skill` calls.

</details>
