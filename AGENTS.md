# AGENTS.md - OpenCode Configuration Repository

This repository contains agent definitions, skills, and configuration for the OpenCode AI coding assistant. It specializes in Elixir/Phoenix and Python development.

## Project Overview

This is a **configuration repository** that defines AI agents and their knowledge bases. It contains:
- **Agent definitions**: Markdown files with YAML frontmatter defining agent behavior
- **Skill libraries**: Knowledge bases that agents can load for specialized guidance
- **Rules**: Global behavioral constraints and patterns

**Important**: This is NOT a software development project - no code compilation, testing, or traditional development workflows apply.

## Project Structure

```
/home/vircung/.config/opencode/
├── opencode.json           # Main configuration
├── package.json            # Node.js dependencies (Bun preferred, NPM fallback)
├── bun.lock               # Bun lockfile
├── agent/                 # Agent definitions (7 agents)
│   ├── elixir-architect.md
│   ├── elixir-developer.md
│   ├── elixir-phoenix-helper.md
│   ├── elixir-reviewer.md
│   ├── python-architect.md
│   ├── python-developer.md
│   └── python-reviewer.md
├── skill/                 # Knowledge bases (9 skills)
│   ├── elixir-*/SKILL.md
│   └── python-*/SKILL.md
└── rules/                 # Global behavioral rules
    ├── no-auto-commit.md
    └── agent-questions.md
```

## Package Management

```bash
# Preferred (if available)
bun install
bun add <package>

# Fallback (always available)
npm install
npm add <package>
```

## Configuration Validation

```bash
# Check OpenCode config validity
opencode config validate

# Check agent definitions
opencode agent list
opencode agent validate <agent-name>

# Check skill loading
opencode skill list
opencode skill validate <skill-name>
```

## File Format Standards

### Agent Definition Format
```yaml
---
description: Brief description of agent's role
mode: subagent                    # subagent | primary | all
temperature: 0.1-0.3             # LLM temperature
tools:
  write: true/false
  edit: true/false
  bash: true/false
  grep: true
  glob: true
  read: true
  task: true
  webfetch: true/false
permission:
  bash:
    "command pattern": "allow/ask"
  skill:
    "skill-name": "allow"
---

# Agent content in Markdown
```

### Skill Definition Format
```yaml
---
name: skill-name
description: Description of skill content
license: MIT
compatibility: opencode
metadata:
  audience: developers/architects/reviewers
  language: elixir/python
  focus: specific-area
---

# Skill content in Markdown
```

## Naming Conventions

### Files and Directories
- **Agent files**: `{language}-{role}.md` (kebab-case)
  - Examples: `python-developer.md`, `elixir-architect.md`
- **Skill directories**: `{language}-{topic}/` (kebab-case)
  - Examples: `elixir-ecto/`, `python-development/`
- **Skill files**: Always named `SKILL.md` (uppercase)
- **Rule files**: `{kebab-case}.md`

### Agent Types by Role
| Agent | Mode | Write Access | Temperature | Purpose |
|-------|------|--------------|-------------|---------|
| `*-architect` | subagent | No | 0.2 | System design, read-only analysis |
| `*-developer` | subagent | Yes | 0.3 | Implementation, full write access |
| `*-reviewer` | subagent | No | 0.1 | Code quality, security analysis |
| `elixir-phoenix-helper` | subagent | No | 0.2 | Generator execution only |

## Code Style Guidelines

### Agent Definition Structure
1. **YAML frontmatter** with required fields
2. **Core Responsibilities** section listing primary duties
3. **When to Engage** section for proactive invocation
4. **Approach** section with step-by-step methodology
5. **Key Principles** or patterns specific to the agent

### Markdown Formatting
- Use **bold** for section headers and important terms
- Use `code blocks` for commands, file names, and code
- Use bullet points for lists and guidelines
- Keep line length reasonable (no hard limit)

### Agent Content Guidelines
- **Be specific**: Include exact commands, file paths, patterns
- **Cross-reference**: Link to related agents and skills
- **Practical**: Focus on real-world, actionable guidance
- **Domain-focused**: Target the specific technology ecosystems the agents serve

## Error Handling

### Critical Rule: No Automatic Commits
**NEVER execute `git commit` in any form**

From `/rules/no-auto-commit.md`:
- ✅ Allowed: `git add`, `git status`, `git diff`, `git log`
- ❌ Forbidden: `git commit`, `git commit -m`, `git commit --amend`
- Always stage files and ask user to commit manually

### Clarifying Questions Pattern
From `/rules/agent-questions.md`:
- **Primary agents** can ask questions via `question` tool
- **Subagents** cannot ask questions directly
- Use **pre-ask pattern**: Primary → Questions → Answers → Subagent with context

### Missing Information Handling
When subagents lack information:
1. **State assumptions clearly** in comments
2. **Choose sensible defaults**
3. **Provide alternatives** for different scenarios
4. **Flag for review** with TODO comments

## Agent Collaboration Patterns

### Development Workflow
```
User Request → architect (design/planning)
                     ↓
            specialist-helper (framework-specific tasks)
                     ↓
            developer (implementation)
                     ↓
            reviewer (quality assurance)
```

### Architecture Review Workflow  
```
User Request → architect (system design)
                     ↓
            developer (implementation)
                     ↓
            reviewer (quality check)
```

### Skill Loading Strategy
- **Architects**: Load architecture + OTP + domain-specific skills
- **Developers**: Load development + implementation skills  
- **Reviewers**: Load review + security + performance skills

## Configuration Management

### Agent Configuration
- **Temperature settings**: 0.1 (reviewers) → 0.2 (architects) → 0.3 (developers)
- **Tool permissions**: Granular control via `tools` and `permission` sections
- **Skill loading**: Automatic via `permission.skill` declarations
- **Mode restrictions**: All agents are `subagent` mode (invoked by primary agents)

### Skill Organization
- **Language-specific skills**: Target modern patterns and frameworks for each technology
- **Cross-references**: Skills reference related skills and agents
- **Practical examples**: Real-world code patterns and anti-patterns
- **Best practices**: Include both good and bad examples with explanations

---

**Important**: This repository configures AI agents. Agents operating here should focus on maintaining agent definitions, skills, and rules rather than traditional software development tasks.