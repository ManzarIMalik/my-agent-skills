<!-- <div align="center">
  <a href="https://agentskills.io">
    <img src="assets/cover-image.png" alt="Agent Skills" width="90%">
  </a>
</div> -->

# My Agent Skills [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

A living notebook of the Agent Skills formats, platforms, and personal workflows I rely on every time I build or steer an agent session. This repo surfaces the resources I bookmark, the tools I trust, and the skills I publish for others to load.

## Contents
- [What Are Agent Skills](#what-are-agent-skills)
- [Why I Curate This List](#why-i-curate-this-list)
- [Start Here](#start-here)
- [Phase 1: Learn the Fundamentals](#phase-1-learn-the-fundamentals)
- [Phase 2: Use Existing Skills](#phase-2-use-existing-skills)
- [Phase 3: Build and Integrate](#phase-3-build-and-integrate)
- [Phase 4: Benchmarks and Research](#phase-4-benchmarks-and-research)
- [Skills I Use](#skills-i-use)
- [Frequently Asked Questions](#frequently-asked-questions)
- [Contributing](#contributing)

## What Are Agent Skills

Agent Skills are modular `SKILL.md` packages that expose workflows, guardrails, and metadata to agents only when they are needed. Because the format is file-based, you can version, review, and reuse the same skills across Claude, Copilot, or any system that understands the progressive-disclosure pattern.

`agent-skills` · `progressive-disclosure` · `skill-md` · `context-management` · `guardrails`.

## Why I Curate This List

This is not a generic "awesome" list; it mirrors the signals I follow, the APIs I call (Claude, GitHub Copilot, OpenAI Codex, and others), and the specialized skills I either host or want to audit. Think of it as a mix of canonical references, emerging research, and the skills I currently recommend to my own team.

## Start Here

If you are new to Agent Skills or just want a quick tour, these primers are where I always begin.

- [What are skills](https://agentskills.io/what-are-skills) — Beginner-friendly explanation of the SKILL.md format and agent-capability plumbing.
- [Using skills in Claude](https://support.claude.com/en/articles/12512180-using-skills-in-claude) — Practical guide for enabling Skills inside Claude’s UI.

## Phase 1: Learn the Fundamentals

### Key Articles
- [Equipping agents for the real world with Agent Skills](https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills) — Original Anthropic announcement.
- [Claude Skills vs MCP: Complete Guide](https://dev.to/jimquote/claude-skills-vs-mcp-complete-guide-to-token-efficient-ai-agent-architecture-4mkf) — Comparison of Skills and Model Context Protocol.
- [The Great AI Agent Configuration Confusion](https://medium.com/@satinath.mondal/the-great-ai-agent-configuration-confusion-agents-md-skill-md-and-whats-next-12345) — Analysis of SKILL.md, AGENTS.md, and related conventions.
- [Using skills with Deep Agents](https://blog.langchain.com/using-skills-with-deep-agents/) — How LangChain’s Deep Agents layer adopts this pattern.

### Video Introductions
- [Claude's new Agent Skills](https://www.youtube.com/watch?v=VRzkafNIdgI) — One-minute overview.
- [Don't Build Agents, Build Skills Instead](https://www.youtube.com/watch?v=CEvIs9y1uog) — Anthropic talk on skills as a scalable abstraction.
- [Agent Skills Explained: Why This Changes Everything](https://www.youtube.com/watch?v=Ihoxov5x66k) — Why skills matter for agent development.
- [Claude Agent Skills Tutorial and Demo](https://www.youtube.com/watch?v=mxZqEduwyFk) — Hands-on walkthrough with Claude Code.
- [Claude Code Skills built me an AI Agent Team](https://www.youtube.com/watch?v=OdtGN27LchE) — Extended beginner guide.

### Courses
- [Agent Skills with Anthropic](https://learn.deeplearning.ai/courses/agent-skills-with-anthropic/) — Short DeepLearning.AI course.

## Phase 2: Use Existing Skills

### Supported Platforms and IDEs I Watch Closely
- [Claude Code](https://claude.ai/code) — Anthropic’s coding environment with Skill activation ([docs](https://code.claude.com/docs/en/skills)).
- [OpenAI Codex](https://developers.openai.com/codex/skills/) — CLI-first Codex experience for skills ([docs](https://developers.openai.com/codex/skills/)).
- [Gemini CLI](https://geminicli.com) — Terminal-first agent with skills ([docs](https://geminicli.com/docs/cli/skills/)).
- [Cursor](https://cursor.com/) — Editor with baked-in Skill awareness ([docs](https://cursor.com/docs/context/skills)).
- [VS Code](https://code.visualstudio.com/) — Official support for agent skills via Copilot customization ([docs](https://code.visualstudio.com/docs/copilot/customization/agent-skills)).
- [GitHub Copilot](https://github.com/features/copilot) — Skills shipped alongside GitHub’s AI pair programmer ([docs](https://docs.github.com/copilot/concepts/agents/about-agent-skills)).
- [Manus](https://manus.im/features/agent-skills) — Autonomous AI agent and its skill marketplace ([blog](https://manus.im/blog/manus-skills)).
- [OpenCode](https://opencode.ai/) — AI studio with built-in skills loader ([docs](https://opencode.ai/docs/skills/)).
- [Amp](https://ampcode.com/) — Companion agent with hosted skills ([docs](https://ampcode.com/manual#agent-skills)).
- [Goose](https://block.github.io/goose/) — Open-source framework with skill extensions ([docs](https://block.github.io/goose/extensions)).
- [Letta](https://www.letta.com/) — Stateful LLM agents with skill-memory integration ([docs](https://docs.letta.com/letta-code)).
- [Roo Code](https://roocode.com/) — Extension + cloud setup for VS Code skills ([docs](https://docs.roocode.com/features/skills)).
- [antigravity](https://antigravity.google/docs/skills) — Google Labs showcase of experimental agent skills.

### Ready-to-Use Skill Libraries
#### Top Picks
- [Anthropic skills](https://github.com/anthropics/skills) — Official catalog from Anthropic.
- [OpenAI skills](https://github.com/openai/skills) — OpenAI’s curated skill bundles.
- [Vercel skills](https://skills.sh/vercel-labs/agent-skills) — Web-focused skills by Vercel Labs.
- [Hugging Face skills](https://github.com/huggingface/skills) — Community-driven repository of cross-platform skills.
- [OpenClaw skills](https://www.clawhub.ai/skills) — Skills targeting the OpenClaw ecosystem.

#### More Collections Worth My Time
- [Orchestra-Research/AI-research-SKILLs](https://github.com/Orchestra-Research/AI-research-SKILLs)
- [karanb192/awesome-claude-skills](https://github.com/karanb192/awesome-claude-skills)
- [shajith003/awesome-claude-skills](https://github.com/shajith003/awesome-claude-skills)
- [GuDaStudio/skills](https://github.com/GuDaStudio/skills)
- [DougTrajano/pydantic-ai-skills](https://github.com/DougTrajano/pydantic-ai-skills)
- [OmidZamani/dspy-skills](https://github.com/OmidZamani/dspy-skills)
- [ponderous-dustiness314/awesome-claude-skills](https://github.com/ponderous-dustiness314/awesome-claude-skills)
- [hikanner/agent-skills](https://github.com/hikanner/agent-skills)
- [gradion-ai/freeact-skills](https://github.com/gradion-ai/freeact-skills)

### Skill Marketplaces & Directories
- [SkillsMP](https://skillsmp.com/)
- [Skillstore](https://skillstore.io/)
- [SkillsDirectory](https://www.skillsdirectory.org/)
- [skills.sh](https://skills.sh/)

## Phase 3: Build and Integrate

### How to Build Skills
- [How to create custom skills](https://support.claude.com/en/articles/12512198-creating-custom-skills)
- [Skills API Quickstart](https://docs.claude.com/en/api/skills-guide#creating-a-skill)
- [How I Built Agent Skills for Claude Code](https://dev.to/nunc/how-i-built-agent-skills-for-claude-code-oj4)
- [Claude Agent Skills Tutorial](https://www.youtube.com/watch?v=fOxC44g8vig)

### Developer Tools I Inspect
- [LangChain Multi-Agent Skills](https://docs.langchain.com/oss/python/langchain/multi-agent/skills)
- [SkillCheck](https://github.com/agentigy/skillcheck)
- [OpenSkills](https://github.com/numman-ali/openskills)
- [LangChain Deep Agents](https://github.com/langchain-ai/deepagents)
- [IntentKit](https://github.com/crestalnetwork/intentkit)
- [Agentica](https://github.com/wrtnlabs/agentica)

### Reference Implementations I Bookmark
#### Development and Programming
- [kylehughes/the-unofficial-swift-concurrency-migration-skill](https://github.com/kylehughes/the-unofficial-swift-concurrency-migration-skill)
- [gapmiss/obsidian-plugin-skill](https://github.com/gapmiss/obsidian-plugin-skill)
- [frmoretto/stream-coding](https://github.com/frmoretto/stream-coding)
- [remotion-dev/remotion](https://github.com/remotion-dev/remotion/tree/main/packages/skills)

#### Integration and Automation
- [SawyerHood/dev-browser](https://github.com/SawyerHood/dev-browser)
- [gotalab/skillport](https://github.com/gotalab/skillport)
- [gmickel/sheets-cli](https://github.com/gmickel/sheets-cli)
- [fabioc-aloha/spotify-skill](https://github.com/fabioc-aloha/spotify-skill)

## Phase 4: Benchmarks and Research

### Benchmarks and Evaluation
- [huggingface/upskill](https://github.com/huggingface/upskill)
- [benchflow-ai/SkillsBench](https://github.com/benchflow-ai/SkillsBench)

### Advanced Engineering
- [Claude Agent Skills: A First Principles Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/)
- [I finally CRACKED Claude Agent Skills](https://www.youtube.com/watch?v=kFpLzCVLA20)
- [Claude Agent Skills](https://www.youtube.com/watch?v=9XaprFRNTlc)
- [muratcankoylan/Agent-Skills-for-Context-Engineering](https://github.com/muratcankoylan/Agent-Skills-for-Context-Engineering)
- [jakedahn/pomodoro](https://github.com/jakedahn/pomodoro)
- [yzfly/Mind-Cloning-Engineering](https://github.com/yzfly/Mind-Cloning-Engineering)

### Academic Papers
- [Agent Skills Enable a New Class of Realistic and Trivially Simple Prompt Injections](https://arxiv.org/abs/2510.26328)
- [A survey of agent interoperability protocols](https://arxiv.org/abs/2505.02279)
- [Reinforcement Learning for Self-Improving Agent with Skill Library](https://arxiv.org/abs/2512.17102)
- [PolySkill: Learning Generalizable Skills Through Polymorphic Abstraction](https://arxiv.org/abs/2510.15863)

## Skills I Use

This section will showcase the `SKILL.md` packages that I build and use. Expect a mix of Claude-aware workflows, Copilot helpers, and automation scaffolds. (Coming soon.)

## Frequently Asked Questions

### What are Agent Skills

Agent Skills are modular `SKILL.md` packages that provide on-demand capabilities without loading all knowledge up front.

### How do Agent Skills differ from fine-tuning

Fine-tuning changes model weights, while skills provide runtime knowledge and workflows that you can update instantly.

### What is the difference between Agent Skills and MCP

Agent Skills focus on workflows and capabilities, while MCP focuses on secure, structured data and tool access.

### How do I create my first Agent Skill

See the [How to Build Skills](#phase-3-build-and-integrate) section for a step-by-step authoring guide.

### Which AI platforms support Agent Skills

Support varies by platform, but major tools include Claude (Claude.ai and Claude Code), OpenAI Codex, GitHub Copilot, Cursor, VS Code, and more.

### Can I use Agent Skills with ChatGPT or other LLMs

If a platform does not support the format natively, you can often integrate skills via a loader or by adapting the `SKILL.md` instructions into that platform's prompt workflow.

### Are Agent Skills secure

Treat skills like code: review them before using, avoid installing from untrusted sources, and prefer audited skill libraries.

### How do I share Agent Skills with my team

The most common approach is to version skills in Git (in a shared repo) and let your supported tools discover them from a standard directory.

## Contributing

Please read [CONTRIBUTING.md](CONTRIBUTING.md) before submitting changes.

I will share agent skills here—the skills I build and use.
