+++
date = '2026-01-17T14:00:00+01:00'
draft = false
title = 'OpenCode Knowledge: Stop Re-Explaining Your Context Every AI Session'
description = 'Stop re-explaining your context every session. Build an on-demand knowledge vault where AI intelligently loads your specifics (from coding standards to personal preferences) exactly when needed.'
tags = ['opencode', 'ai', 'knowledge-management', 'developer-productivity', 'open-source', 'typescript', 'llm']
categories = ['projects']
authors = ['msegoviadev']
showHero = true
heroStyle = "basic"
+++

I've explained our API design patterns to Claude seventeen times this month. Each new session, same conversation: "Remember, we prefer composition over inheritance. Here are our naming conventions. Again." 

This isn't just about code. Monday morning I'm explaining coding standards. Tuesday afternoon I'm re-describing my blog's voice and structure preferences. Wednesday evening I'm explaining personal finance categorization rules again. The "context repetition tax" compounds with every domain where I have specialized knowledge.

The cognitive overhead is real: Did I already mention this? Which version did I explain last time? Am I being consistent across sessions?

This shouldn't be necessary. We solved this decades ago with documentation, wikis, and style guides. Why are we manually re-inventing context every single AI session?

What if AI could learn your specifics once, then intelligently load exactly what it needs, when it needs it?

## What It Is

opencode-knowledge is an OpenCode plugin that creates an intelligent knowledge vault where AI assistants search and load your specialized knowledge (coding standards, writing guidelines, personal preferences) only when relevant to the current task.

Here's how it works: You organize knowledge in markdown files with tags (a familiar format). AI automatically indexes your vault on session start. When working on a task, AI searches by tags and loads only relevant knowledge. Session state tracks what's been loaded to avoid redundancy.

The key insight: instead of dumping everything upfront (context explosion) or explaining manually every session (repetition tax), AI intelligently loads on-demand. A vast vault where knowledge lives, accessed precisely when needed.

Think of it as a smart filing cabinet where AI knows exactly which drawer to open based on what you're working on.

This works for development teams maintaining coding standards, writers with style guides and brand voice, or anyone with specialized knowledge they're tired of re-explaining.

## The Journey

### The Context Window Problem

Static context files (CLAUDE.md, AGENTS.md, rules/) seemed logical at first: dump all your standards upfront and let AI remember everything. But reality doesn't scale that way.

A small project with 50KB of coding standards consumes 12,000 tokens immediately. Most sessions only need 10-15% of that context. The waste compounds across every session. Add more standards and your context window explodes faster.

I kept running into this: I'm fixing a CSS bug, but the AI has already loaded database migration guidelines, API security policies, and deployment procedures. Why? It doesn't need any of that right now.

The insight hit me: we need selective loading, not wholesale dumping. Quality over quantity. Precision over coverage.

### Collaboration: The Origin Story

This started with my friend [@canyavall](https://github.com/canyavall) and me working through the context repetition problem together. After long discussions about how AI should load knowledge, what inference could and couldn't do, and what developers actually need, we landed on the core design: tag-based search for deterministic context loading. He built the initial implementation as a Claude Code plugin. I took the concept and created this OpenCode version.

### Design Decision: Tags Over AI Inference

Our initial temptation was to just let AI "figure out" what knowledge it needs based on the task. Sounds smart, right? That's exactly how Skills systems work - most LLM platforms (Claude, OpenAI, others) let you define skills that activate through inference when the AI thinks they're relevant.

We explored that approach first. The problems surfaced quickly: unpredictable results (same task, different knowledge loaded each time), hard to debug (why did it load this but not that?), inconsistent behavior across sessions, and a complete black box with no transparency into reasoning. You'd be writing a React component and the testing skill wouldn't activate. You'd discuss API patterns and nothing would load.

Here's a concrete example of the difference: you type "help me test this React component." With Skills inference, the AI might load testing knowledge, or it might not - there's no guarantee. With explicit tag-based search, the AI searches for packages tagged `[react, testing, component]` and gets deterministic ranked results. Same input, predictable output, every time.

Our breakthrough came with explicit tags. Developers create frontmatter: `tags: [react, testing, component-architecture]`. When AI works on a React test, it searches for packages tagged `react,testing` and finds relevant matches. Simple relevance scoring: matched tags divided by the maximum of search tags and package tags.

```typescript
// Simple but effective relevance scoring
const matchedTags = tags.filter(t => packageTags.has(t));
const relevance = matchedTags.length / Math.max(searchTags.size, packageTags.size);
```

The AI is explicitly prompted to search by tags for every task, reinforcing consistent behavior. Description matching adds a second signal. When something doesn't load, you can see exactly why: the tags didn't match, or the relevance score was too low.

Yes, this requires manual tagging. But that's a feature, not a bug: you control exactly what knowledge exists and when it's discoverable. The matching logic is transparent, debuggable, and improvable.

It's trading AI "intelligence" for engineering reliability. Less clever inference, more predictable outcomes.

### Format Choice: Markdown + Frontmatter

Why not a database? Too much overhead, complexity, and requires special tooling. Not git-friendly.

Why not JSON? Not human-friendly for long-form content. Hard to write and edit knowledge packages.

Why markdown with YAML frontmatter? We chose it because developers already know this format from Hugo, Jekyll, Docusaurus, and other static site generators. It's git-friendly (version control, diffs, pull requests), works in any text editor, supports preview, and has zero learning curve.

```markdown
---
tags: [api-design, rest, standards]
description: Core API design principles
category: backend
---
# API Design Standards

[your knowledge here...]
```

## Under the Hood

Three core components power the system:

**Catalog Builder** scans `.opencode/knowledge/vault/` on session start, parses YAML frontmatter from markdown files, builds a searchable index saved as `knowledge.json`, and organizes everything by category, tags, and descriptions.

**Tag-based Search** implements the simple but effective relevance scoring shown earlier. It returns ranked results with highest relevance first.

**Session State Tracker** uses JSONL logs to track which packages have been loaded, preventing redundant loading and persisting state across tool invocations.

The smart injection flow works like this: On first message, AI receives a knowledge map showing categories, top tags, and available tools. During work, AI uses the `knowledge_search` tool to find packages by tags. When needed, AI uses the `knowledge_load` tool to inject specific packages into context. Only what's needed, when it's needed.

The efficiency gain is significant. Instead of 12,000 tokens loaded upfront, a typical session loads 2,000-3,000 tokens precisely when relevant.

## Proof of Concept: Real-World Use

The architecture works beyond code. I'm already using it across multiple domains.

My personal vault includes software development knowledge (code conventions, component architecture patterns, testing strategies, deployment procedures). That was the obvious starting point.

But it also includes content creation guidelines. Blog writing structure and voice, X/Twitter copywriting best practices, brand voice and tone preferences, author context and background. The post you're reading right now? Written with AI that loaded my blog-writing standards on-demand. It knows my voice, my structure preferences, my style guidelines because it intelligently loaded `personal/blog-writing.md` when I said "write a blog post."

I even use it for personal finance. Transaction categorization rules, budget allocation preferences, financial goals and constraints, spending pattern analysis. When I ask AI to categorize expenses, it loads my rules: "Coffee shops under $6 = daily expenses, over $6 = dining out." My specifics, automatically available.

Other domains work well too. Business and project management (decision-making frameworks, stakeholder communication preferences, meeting templates, planning methodologies). Research and learning (note-taking standards, citation preferences, resource organization, topic connections).

The pattern is universal: anywhere you have specialized knowledge you repeatedly explain, create a knowledge package.

## Try It Out

Installation takes about two minutes.

Add the plugin to your OpenCode config at `~/.config/opencode/opencode.json`:

```json
{
  "plugin": ["opencode-knowledge"]
}
```

Create your vault structure:

```bash
mkdir -p .opencode/knowledge/vault/standards
```

Create your first knowledge package at `.opencode/knowledge/vault/standards/code-conventions.md`:

```markdown
---
tags: [standards, typescript, conventions]
description: Core code conventions and style guide
category: standards
---
# Code Conventions

[your standards here]
```

Start an OpenCode session. Automatic indexing happens on start, AI receives the knowledge map, and you use knowledge naturally during work.

The project is on [GitHub](https://github.com/msegoviadev/opencode-knowledge) under MIT license.

Start small. Create 2-3 knowledge packages for your most commonly re-explained concepts. Expand organically as you discover patterns.

## Closing Thoughts

Building this taught me that simple beats complex. Tag-based search feels almost too simple, but it works beautifully. Resisting the urge to over-engineer paid off. Developer-friendly formats matter - markdown plus frontmatter equals zero learning curve. If I'd built a custom format or required special tooling, adoption would've suffered.

My friend @canyavall and I developed this concept together through long hours discussing how AI should load knowledge, what inference could and couldn't do, and what developers actually need. He built it for Claude Code, I built it for OpenCode. The collaboration made it better than either of us would've created alone.

The architecture is completely domain-agnostic. I'm already using it for software development, content creation, and personal finance. Same tag-based search, same markdown vault, same on-demand loading - just different knowledge packages for different domains. When I'm writing code, AI loads architecture patterns. When I'm writing blog posts, it loads my writing guidelines. When I'm categorizing expenses, it loads my finance rules.

Next steps: making it even more deterministic. File patterns for auto-loading, mandatory packages for specific contexts, extension-based triggers - more ways to enforce what loads when, reducing any remaining inference.

If you're tired of re-explaining coding standards, writing guidelines, or any specialized knowledge every AI session, try opencode-knowledge. Start with one domain. Build a small vault. See how it changes your workflow.

The best AI tools get out of your way. Your knowledge vault sits there quietly until needed, then surfaces exactly what's relevant. That's the kind of AI assistance worth building.
