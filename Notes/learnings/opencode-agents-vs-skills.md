# Agents vs Skills in OpenCode

## Core Difference

| | Agent | Skill |
|---|---|---|
| What it is | A specialized AI assistant | Reusable instructions or a playbook |
| Controls | Model, prompt, tools, permissions, and execution mode | Domain knowledge, conventions, and workflow |
| Runs independently | Yes, as a primary agent or subagent | No, it is loaded by an agent |
| Best for | Different roles, access levels, or workflows | Reusing task-specific knowledge |
| Example | A read-only security reviewer | An OWASP security checklist |

A useful analogy:

- **Agent:** the person doing the work
- **Skill:** the reference guide that person uses

## Using Agents

OpenCode has two agent types:

- **Primary agents:** The main conversation agent. Use `Tab` to switch between agents such as `Build` and `Plan`.
- **Subagents:** Specialized agents invoked with `@`, or automatically delegated to by a primary agent.

Examples:

```text
@explore Find where authentication tokens are validated
```

```text
@general Investigate why the integration tests are failing
```

Create a custom agent with:

```bash
opencode agent create
```

Or create `.opencode/agents/reviewer.md`:

```yaml
---
description: Reviews code without making changes
mode: subagent
permission:
  edit: deny
  bash: deny
---

Review code for bugs, security issues, and maintainability problems.
Do not modify files.
```

You can then invoke it with:

```text
@reviewer Review the payment-service changes
```

### Agent Scenarios

- A read-only planning agent
- A security auditor that cannot edit files
- A debugging agent with shell access
- A documentation agent allowed to edit Markdown but not run commands
- A deployment agent configured with a specific model and stricter permissions

## Using Skills

Create a skill at:

```text
.opencode/skills/api-conventions/SKILL.md
```

Example:

```yaml
---
name: api-conventions
description: Apply the project's REST API naming, error, authentication, and pagination conventions
---

## Rules

- Use plural resource names.
- Return consistent error objects.
- Use cursor pagination for collection endpoints.
- Require authentication for write operations.
```

Skills can be project-specific or global:

```text
.opencode/skills/<name>/SKILL.md
~/.config/opencode/skills/<name>/SKILL.md
```

OpenCode discovers the skill and exposes its name and description to the agent. The agent loads it on demand using the native skill tool. You can also request it explicitly:

```text
Implement this endpoint. Use the api-conventions skill first.
```

A skill does not grant additional permissions. If it instructs the agent to edit files or run commands, the selected agent must still have permission to do so.

### Skill Scenarios

- Company coding standards
- React or framework conventions
- Database migration checklist
- Release and changelog procedure
- Incident-response runbook
- API design rules
- Cloud deployment instructions
- A testing strategy used across multiple agents

## When to Choose Which

Use an **agent** when you need:

- A different role or behavior
- Different model selection
- Different tool access or permissions
- A separate subtask or child session
- A persistent workflow such as planning, reviewing, or debugging

Use a **skill** when you need:

- Reusable project knowledge
- A checklist or procedure
- Framework or company conventions
- Instructions that multiple agents should share

They work well together: a read-only `security-reviewer` agent can load a reusable `threat-modeling` skill.

## Documentation

- [OpenCode Agents](https://opencode.ai/docs/agents/)
- [OpenCode Agent Skills](https://opencode.ai/docs/skills/)
