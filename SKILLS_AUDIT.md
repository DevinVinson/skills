# Skills Usage Audit & Migration Recommendations

> **Audit date:** 2026-06-03
> **Data source:** 47 conversations from OpenHands Cloud (agent-canvas)
> **Method:** Scanned conversation events for activated skills (auto-injected by keyword), explicitly invoked skills (`invoke_skill`), and skills loaded in agent context.

## Summary

Across 47 recent conversations, **15 distinct skills were actively used** (activated by keyword or explicitly invoked). Every conversation also loads the full public skills catalog (~40+ skills) into the agent context, but only a subset are actually triggered during use.

### Already in this repo

| Skill | Status |
|---|---|
| `standup` | ✅ Migrated |
| `openhands-agent-server-ui` | ✅ Migrated |
| `openhands-design` | ✅ Migrated |
| `use-git-worktrees` | ✅ Migrated |

---

## Skills Usage Ranking

Skills ranked by actual usage across conversations (activated + invoked).

| Rank | Skill | Conversations | Activated | Invoked | Notes |
|------|-------|:---:|:---:|:---:|-------|
| 1 | `github` | 22 | 148 | 0 | Auto-injects on git/GitHub keywords |
| 2 | `bitbucket` | 22 | 148 | 0 | Auto-injects alongside `github` |
| 3 | `openhands-automation` | 13 | 112 | 10 | Used for cron jobs, scheduling |
| 4 | `openhands-sdk` | 11 | 101 | 0 | Auto-injects on SDK keywords |
| 5 | `gitlab` | 10 | 91 | 0 | Auto-injects alongside other git skills |
| 6 | `security` | 9 | 90 | 0 | Auto-injects on security keywords |
| 7 | `notion` | 9 | 27 | 14 | Heavily invoked for Notion workflows |
| 8 | `npm` | 3 | 30 | 0 | Auto-injects for Node.js/npm work |
| 9 | `docker` | 3 | 21 | 0 | Auto-injects for container work |
| 10 | `linear` | 3 | 12 | 0 | Auto-injects for project management |
| 11 | `vercel` | 2 | 11 | 0 | Auto-injects for deployment work |
| 12 | `uv` | 1 | 10 | 0 | Auto-injects for Python packaging |
| 13 | `custom-codereview-guide` | 1 | 0 | 10 | Explicitly invoked for code review |
| 14 | `code-review` | 1 | 0 | 10 | Explicitly invoked for code review |
| 15 | `github-pr-review` | 1 | 1 | 0 | Auto-injects for PR review |

---

## Migration Recommendations

### Tier 1 — Strongly recommend migrating (high personal usage, customization value)

These skills are frequently used and would benefit from personal customization or version control.

| Skill | Reason | Priority |
|---|---|---|
| `notion` | Actively invoked in 9 conversations (14 explicit invocations). Used for Notion page creation, release roundups, and automation. Personal Notion workflows are highly specific to your workspace and would benefit from customization. | 🔴 High |
| `openhands-automation` | Invoked in 13 conversations. Used to set up cron jobs and automations. Personalizing this with your common automation patterns would save time. | 🔴 High |
| `code-review` | Explicitly invoked for PR reviews. A personal version could encode your review style, priorities, and project-specific standards. | 🟡 Medium |
| `github-pr-review` | Complements `code-review` for posting inline PR comments. Migrating both together creates a complete review workflow. | 🟡 Medium |
| `skill-creator` | Loaded in every conversation. As you build more personal skills, having your own version with your conventions and directory structure would be valuable. | 🟡 Medium |

### Tier 2 — Consider migrating (moderate usage, some customization value)

These skills are auto-activated frequently but work well as-is from the public catalog.

| Skill | Reason | Priority |
|---|---|---|
| `github` | Activated in 22 conversations, but the public version is generic and works well. Only migrate if you want to add personal GitHub workflow patterns. | 🟢 Low |
| `npm` | Activated in 3 conversations for Node.js work. Only migrate if you have custom npm configurations or private registries. | 🟢 Low |
| `docker` | Activated in 3 conversations. Public version is sufficient unless you have custom Docker workflows. | 🟢 Low |
| `openhands-api` | Loaded in every conversation. You already depend on it for automation scripts. Migrate only if you want to extend it with custom helpers. | 🟢 Low |

### Tier 3 — No need to migrate (auto-injected, generic)

These skills activate automatically based on keywords and provide generic platform support. They work well from the public catalog and don't benefit from personalization.

| Skill | Reason |
|---|---|
| `bitbucket` | Auto-injects alongside `github`. Only useful if you use Bitbucket. |
| `gitlab` | Auto-injects alongside `github`. Only useful if you use GitLab. |
| `security` | Generic security guidelines. Auto-injects on security keywords. |
| `openhands-sdk` | Auto-injects for SDK work. Public version tracks upstream. |
| `linear` | Auto-injects for project management. Public version is sufficient. |
| `vercel` | Auto-injects for deployment. Public version is sufficient. |
| `uv` | Auto-injects for Python packaging. Public version is sufficient. |

---

## Key Conversations by Skill Usage

| Conversation | Skills Used |
|---|---|
| 📝 Sync Skills Repo with Agent-Server | github, bitbucket, openhands-automation, gitlab, security, openhands-sdk |
| ✅ Automate Daily Notion Release Check | notion (invoked ×10), openhands-automation (invoked ×10), vercel |
| 🐛 Review agent-canvas bugfix PR #654 | code-review (invoked ×10), custom-codereview-guide (invoked ×10), github, bitbucket, linear |
| ✅ Test Notion MCP automation | notion, openhands-automation |
| 📝 Weekly OpenHands Release Roundup Draft | notion (invoked), github, bitbucket, gitlab, openhands-sdk |
| 🐛 Agent-Canvas npm run dev Error | npm, uv, github, bitbucket, openhands-sdk |
| 📝 Plan Agent Canvas docs migration | github, bitbucket, docker, npm |

---

## Methodology Notes

- **Activated skills** are auto-injected by the system when user messages contain matching keywords. High activation counts in long conversations reflect repeated keyword matches across multiple user messages — not necessarily intentional skill use.
- **Invoked skills** are explicitly called via `invoke_skill` by the agent, indicating the skill's content was directly loaded and used.
- **Conversations with 1000 events** hit the API pagination limit; actual activation counts may be higher.
- Skills like `agent-memory`, `skill-creator`, and all other public skills are loaded into every conversation's available skills list, but are only counted as "used" when activated or invoked.
- Some skills (e.g., `bitbucket`, `gitlab`) co-activate with `github` because they share keyword triggers; their high numbers don't necessarily indicate independent usage.
