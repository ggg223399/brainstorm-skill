# Brainstorm — Multi-Agent Discussion Skill for Claude Code

A topic-adaptive multi-agent discussion system for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Dynamically generates expert roles based on your topic, runs structured discussions with configurable execution patterns, and produces synthesized reports.

## Features

- **Dynamic role generation** — Automatically creates 3-5 expert roles with built-in tension (analytical, critical, practical, divergent, empathetic)
- **Flexible execution model** — Two orthogonal parameters (`order` grouping + `rounds`) naturally produce parallel, staged, debate, or staged-debate behaviors
- **Preset reuse** — Save and reuse role configurations for recurring topics
- **Evolution** — Presets improve over time based on user feedback
- **Cross-preset wisdom** — Learnings from one preset inform future role generation

## Execution Model

Discussions are controlled by two parameters:

| `order` distribution | `rounds` | Behavior |
|---------------------|----------|----------|
| All same | 1 | **Parallel** — Independent perspectives |
| Multiple groups | 1 | **Staged** — Propose → Review → Refine |
| All same | 2-3 | **Debate** — Multi-round convergence |
| Multiple groups | 2-3 | **Staged debate** — Staged + multi-round |

## Installation

### Claude Code

Copy the command file to your project:

```bash
# Create the commands directory if it doesn't exist
mkdir -p .claude/commands

# Copy the brainstorm command
cp .claude/commands/brainstorm.md <your-project>/.claude/commands/brainstorm.md
```

Then use it with `/brainstorm` in Claude Code.

### Setup presets directory

```bash
# Create a directory for your presets (the skill looks here by default)
mkdir -p Apps/brainstorm/presets

# Optionally copy example presets
cp examples/presets/*.json Apps/brainstorm/presets/
```

> **Note:** You may need to adjust the preset directory path in `brainstorm.md` if your project structure differs.

## Usage

In Claude Code, type:

```
/brainstorm <your topic>
```

Examples:
- `/brainstorm fitness plan for beginners`
- `/brainstorm tech stack selection for a new SaaS`
- `/brainstorm product naming for a crypto wallet`

The system will:
1. Check for existing presets or generate new roles
2. Collect your personal context for customization
3. Execute the discussion (parallel, staged, or multi-round)
4. Produce a synthesized report with consensus, disagreements, and action items

## File Structure

```
.claude/commands/brainstorm.md  # Main skill file (Claude Code command)
schema.json                     # Preset JSON schema
EVOLUTION-REFERENCE.md          # Detailed evolution mechanism docs
examples/
  presets/
    _wisdom.json                # Cross-preset best practices
    fitness-plan.json           # Example: fitness planning preset
    schedule-management.json    # Example: schedule management preset
```

## Creating Custom Presets

Presets are JSON files following `schema.json`. Key fields per agent:

```json
{
  "id": "unique-slug",
  "name": "Role Name",
  "responsibility": "One-line description",
  "thinking_style": "analytical | divergent | critical | practical | empathetic",
  "system_prompt": "Full system prompt with constraints and output requirements",
  "subagent_type": "oracle | metis",
  "order": 1
}
```

Set `order` to group agents into stages. Agents with the same `order` run in parallel within their stage. Set `rounds` in `discussion_config` for multi-round discussions.

## License

MIT
