---
title: 'GitHub Copilot in VS Code'
description: 'Explore the latest GitHub Copilot features in Visual Studio Code, including multi-window sessions, /btw side chats, live status pills, browser element commenting, and dictation customization.'
authors:
  - GitHub Copilot Learning Hub Team
lastUpdated: 2026-08-06
estimatedReadingTime: '8 minutes'
tags:
  - vs-code
  - copilot-app
  - agents
  - fundamentals
relatedArticles:
  - ./github-copilot-app.md
  - ./using-copilot-coding-agent.md
  - ./understanding-mcp-servers.md
prerequisites:
  - Visual Studio Code installed
  - GitHub Copilot extension installed (stable or Insiders)
---

GitHub Copilot's VS Code extension continues to expand its agent-centric capabilities. This article covers the key features introduced in VS Code 1.132 (August 2026) and explains how to get the most out of Copilot directly inside your editor.

## Multi-Window Session Support

VS Code 1.132 introduces **multi-window session support**: you can now connect to the same Copilot agent session from multiple VS Code windows simultaneously. This is powered by the **agent host** introduced in earlier releases, which runs the session as a shared service that any VS Code window can attach to.

### Why This Matters

- **Split-screen workflows**: Keep a planning window open on one monitor and a coding window on another — both talk to the same agent and see the same session state.
- **Remote + local**: Connect to a session running on a remote machine from a local VS Code window (or vice versa) without losing context.
- **Team handoffs**: Share a session identifier with a colleague so they can join the same running session.

### How to Use It

Open the same session from any VS Code window by selecting it from the session picker (accessible via the Copilot status bar item → **Manage Sessions**). Each window shows the session's live state and can send messages to the shared agent.

## `/btw` Side Chats

The `/btw` command creates a **side chat** — a lightweight, independent conversation thread you can open alongside your main session. Side chats are useful when you want to ask a quick question or explore an idea without derailing the main session's context.

```
/btw What's the difference between useEffect and useLayoutEffect?
```

Side chats appear as a collapsible panel next to the main conversation. They share the same model and session settings but maintain a separate conversation history. Close a side chat when you're done; the main session is unaffected.

### When to Use Side Chats

| Scenario | Use Main Chat | Use `/btw` Side Chat |
|----------|--------------|---------------------|
| Multi-step implementation task | ✅ | |
| Quick factual question | | ✅ |
| Exploring an alternative approach mid-task | | ✅ |
| Reviewing code the agent just wrote | ✅ | |
| Looking up a documentation reference | | ✅ |

## Live Status Pills

VS Code 1.132 adds **live status pills** — small inline indicators that appear in the Copilot panel header to show what the agent is actively doing. Pills appear for:

- **Changes**: Files the agent has modified in the current turn
- **Previews**: Open browser or document previews the agent is working with
- **Subagents**: Delegated work items running in parallel (e.g., a background research task)
- **Browsers**: Active browser sessions the agent is controlling

Clicking a pill opens the corresponding view — for example, clicking the Changes pill opens a diff of the current turn's modifications, and clicking a Browser pill opens the browser session panel.

Status pills give you at-a-glance awareness of what the agent is doing in complex multi-step tasks, reducing the need to scroll through the conversation to understand the current state.

## Browser Element Commenting

When the agent has a browser session open (for example, during web development or testing tasks), you can now **click directly on page elements to attach comments** that the agent sees as context.

### How It Works

1. The agent opens a browser preview (e.g., `localhost:3000`)
2. You see the live page in the Browser panel
3. Click any element — a comment bubble appears
4. Type your feedback: "This button should be primary blue" or "This layout breaks on mobile"
5. The agent reads your element-specific comment and applies the change

This creates a tighter design-to-code feedback loop: instead of describing UI issues in text, you point directly at the element that needs changing. The agent receives both the element's location in the DOM and your comment text.

## Dictation Customization

VS Code 1.132 ships experimental **multilingual dictation** support with file-based customization. You can configure dictation behavior using Markdown files at two levels:

### User-level dictation preferences

```
~/.copilot/dictation.md
```

This file configures personal dictation preferences that apply across all projects — preferred language, custom vocabulary, abbreviation expansions, and voice-specific corrections.

**Example `~/.copilot/dictation.md`**:

```markdown
# Dictation Preferences

## Language
Primary: English (en-US)
Secondary: Spanish (es-MX)

## Custom Vocabulary
- "GH" → "GitHub"
- "Co-pee-lot" → "Copilot"
- "YAML" → "YAML" (preserve spelling, don't expand)

## Abbreviations
- "fn" → "function"
- "cls" → "class"
```

### Repository-level dictation preferences

```
.github/dictation.md
```

This file lets you define project-specific dictation overrides — useful for repositories with domain-specific terminology that the general dictation model might get wrong (e.g., proprietary product names, technical jargon).

**Example `.github/dictation.md`**:

```markdown
# Project Dictation

## Domain Terms
- "Kuzco" → "Kuzco" (our internal caching library, not a name)
- "preact" → "Preact"
- "vite" → "Vite"
```

Repository-level preferences layer on top of user-level preferences — both files are read, with the repository file taking precedence for any overlapping terms.

> **Note**: Dictation support is experimental in VS Code 1.132. Enable it via **Settings → GitHub Copilot → Experimental → Dictation** or by adding `"github.copilot.experimental.dictation": true` to your VS Code settings.

## Deprecations: `ChatAgentHostEnabled` Policy Removed

The `ChatAgentHostEnabled` policy (used to control whether VS Code ran the Copilot agent host) has been **removed** in VS Code 1.132. The agent host is now always enabled and cannot be turned off via this policy.

If you are managing VS Code deployments through MDM or Group Policy and have this setting configured, remove it — it is silently ignored as of this release. To control Copilot access more broadly, use your GitHub organization's Copilot policy settings instead.

## Further Reading

- **Official VS Code release notes**: [VS Code 1.132 (August 2026)](https://code.visualstudio.com/updates/v1_132) — full release notes with all Copilot changes
- **GitHub Copilot app**: [Getting Started with the GitHub Copilot app](../github-copilot-app/) — the standalone desktop experience for multi-agent work
- **Using the Coding Agent**: [Using the Copilot Coding Agent](../using-copilot-coding-agent/) — assign issues to Copilot and let it work autonomously
- **MCP Servers**: [Understanding MCP Servers](../understanding-mcp-servers/) — extend Copilot with external tools

---
