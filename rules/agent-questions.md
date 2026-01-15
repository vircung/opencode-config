# Agent Clarifying Questions Pattern

## Overview

The primary agent can ask users clarifying questions before invoking subagents. This enables better outcomes by gathering requirements upfront and passing them to specialized subagents.

**Important**: Subagents cannot ask questions directly - answers are not propagated back to them. The primary agent must ask questions first and pass answers in the task prompt.

## Architecture

```
User Request
    ↓
Primary Agent (can ask questions)
    ↓
[Question Tool] ←→ User answers
    ↓
Primary Agent receives answers
    ↓
Task Tool (passes answers in prompt)
    ↓
Subagent (works with provided context)
    ↓
Result returns to Primary Agent
```

## Pre-Ask Pattern

### Step 1: Primary Agent Identifies Ambiguity

Before invoking a subagent, the primary agent should identify what clarification is needed.

### Step 2: Primary Agent Asks Questions

```
Use the question tool with:
- question: "How should we handle user authentication?"
- header: "Auth" (max 12 chars)
- options:
  - label: "Session-based", description: "Traditional cookies, simpler"
  - label: "JWT tokens", description: "Stateless, good for APIs"
  - label: "OAuth only", description: "Delegate to providers"
- multiple: false
```

### Step 3: Pass Answers to Subagent

Include the user's answers in the task prompt:

```
Task prompt to @elixir-architect:
"Design authentication for our Phoenix app.

User requirements:
- Authentication method: JWT tokens
- Must support API access
- Team size: 3 developers

Please provide architectural recommendations..."
```

## What to Clarify by Agent Type

### Before Invoking Architects
- Project scope and constraints
- Team size and experience level
- Scalability requirements
- Performance vs. maintainability priorities

### Before Invoking Developers
- Implementation approach preferences
- Edge case behavior
- API design choices
- Library/dependency preferences

### Before Invoking Reviewers
- Review scope (security, performance, maintainability)
- Priority areas
- Time constraints
- Production criticality

### Before Invoking Phoenix Helper
- Generator choice (html vs live vs json)
- Field specifications and types
- Context organization
- UI preferences

## Subagent Behavior

Subagents cannot ask questions. When information is missing, they should:

1. **State assumptions clearly** - Document what was assumed
2. **Choose sensible defaults** - Pick the most pragmatic approach
3. **Provide alternatives** - Offer different options for different scenarios
4. **Flag for follow-up** - Note what needs user confirmation

## Question Tool Reference

```typescript
type QuestionInfo = {
  question: string;      // Full question text
  header: string;        // Short label (max 12 chars)
  options: Array<{
    label: string;       // Choice label (1-5 words)
    description: string; // Explanation of choice
  }>;
  multiple?: boolean;    // Allow selecting multiple (default: false)
  custom?: boolean;      // Allow custom answer (default: true)
};
```

## Best Practices

1. **Ask early** - before invoking subagents, not after
2. **Be specific** - focused questions with clear options
3. **Explain trade-offs** - help users understand implications
4. **Group related questions** - ask related things together
5. **Keep headers short** - max 12 characters
6. **Include context in task prompt** - always pass answers to subagents
