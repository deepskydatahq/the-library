---
name: library
description: Private skill distribution system. Use when the user wants to install, use, add, push, remove, sync, list, or search for skills, agents, or prompts from their private library catalog. Triggers on /library commands or mentions of library, skill distribution, or agentic management.
argument-hint: [command or prompt] [name or details]
---

# The Library

A meta-skill for private-first distribution of agentics (skills, agents, and prompts) across agents, devices, and teams.

## Variables

> Update these after forking and cloning the library repo.

- **LIBRARY_REPO_URL**: `git@github.com:deepskydatahq/the-library.git`
- **LIBRARY_YAML_PATH**: `~/.claude/skills/library/library.yaml`
- **LIBRARY_SKILL_DIR**: `~/.claude/skills/library/`

## How It Works

The Library is a catalog of references to your agentics. The `library.yaml` file points to where skills, agents, and prompts live (local filesystem or GitHub repos). Nothing is fetched until you ask for it.

**The `library.yaml` is a catalog, not a manifest.** Entries define what's *available* — not what gets installed. You pull specific items on demand with `/library use <name>`.

## Commands

| Command                     | Purpose                                  |
| --------------------------- | ---------------------------------------- |
| `/library install`          | First-time setup: fork, clone, configure |
| `/library add <details>`    | Register a new entry in the catalog      |
| `/library use <name>`       | Pull from source (install or refresh)    |
| `/library push <name>`      | Push local changes back to source        |
| `/library remove <name>`    | Remove from catalog and optionally local |
| `/library list`             | Show full catalog with install status    |
| `/library sync`             | Re-pull all installed items from source   |
| `/library search <keyword>` | Find entries by keyword                  |

## Agentic Types

The library manages five types of agentics:

| Type | Directory | Description |
| ---- | --------- | ----------- |
| **skill** | `.claude/skills/` | Capabilities — what an agent can do (testing patterns, crawler pipelines) |
| **agent** | `.claude/agents/` | Autonomous agents with specialization and scale |
| **prompt** | `.claude/commands/` | Orchestration commands that coordinate skills and agents |
| **expert** | `.claude/experts/` | Specialized reviewers and advisors (product strategy, architecture) |
| **hook** | `.claude/hooks/` | Lifecycle scripts triggered on session events (start, stop) |

## Cookbook

Each command has a detailed step-by-step guide. **Read the relevant cookbook file before executing a command.**

| Command | Cookbook                                 | Use When                                                    |
| ------- | --------------------------------------- | ----------------------------------------------------------- |
| install | [cookbook/install.md](cookbook/install.md) | First-time setup on a new device                            |
| add     | [cookbook/add.md](cookbook/add.md)         | User wants to register a new skill/agent/prompt in catalog  |
| use     | [cookbook/use.md](cookbook/use.md)         | User wants to pull or refresh a skill from the catalog      |
| push    | [cookbook/push.md](cookbook/push.md)       | User improved a skill locally and wants to update the source |
| remove  | [cookbook/remove.md](cookbook/remove.md)   | User wants to remove an entry from the catalog               |
| list    | [cookbook/list.md](cookbook/list.md)       | User wants to see what's available and what's installed      |
| sync    | [cookbook/sync.md](cookbook/sync.md)       | User wants to refresh all installed items at once            |
| search  | [cookbook/search.md](cookbook/search.md)   | User is looking for a skill but doesn't know the exact name |

**When a user invokes a `/library` command, read the matching cookbook file first, then execute the steps.**

## Source Format

The `source` field in `library.yaml` supports these formats (auto-detected):

- `/absolute/path/to/SKILL.md` — local filesystem
- `https://github.com/org/repo/blob/main/path/to/SKILL.md` — GitHub browser URL
- `https://raw.githubusercontent.com/org/repo/main/path/to/SKILL.md` — GitHub raw URL

Both GitHub URL formats are supported. Parse org, repo, branch, and file path from the URL structure. For private repos, use SSH or `GITHUB_TOKEN` for auth automatically.

**Important:** The source points to a specific file (SKILL.md, AGENT.md, expert .md, hook .sh, or prompt file). For skills, we pull the entire parent directory. For agents, experts, prompts, and hooks, we copy just the file.

## Source Parsing Rules

**Local paths** start with `/` or `~`:
- Use the path directly. Copy the parent directory of the referenced file.

**GitHub browser URLs** match `https://github.com/<org>/<repo>/blob/<branch>/<path>`:
- Parse: `org`, `repo`, `branch`, `file_path`
- Clone URL: `https://github.com/<org>/<repo>.git`
- File location within repo: `<path>`

**GitHub raw URLs** match `https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path>`:
- Parse: `org`, `repo`, `branch`, `file_path`
- Clone URL: `https://github.com/<org>/<repo>.git`
- File location within repo: `<path>`

## GitHub Workflow

When working with GitHub sources, prefer `gh api` for accessing single files (e.g., reading a SKILL.md to check metadata). For pulling entire skill directories, clone into a temp dir per the steps below.

**Fetching (use):**
1. Clone the repo with `git clone --depth 1 <clone_url>` into a temporary directory
2. Navigate to the parent directory of the referenced file
3. Copy that entire directory to the target local directory
4. The temporary directory is cleaned up automatically

**Pushing (push):**
1. Clone the repo with `git clone --depth 1 <clone_url>` into a temporary directory
2. Overwrite the skill directory in the clone with the local version
3. Stage only the relevant changes: `git add <skill_directory_path>`
4. Commit with message: `library: updated <skill name> <what changed>`
5. Push to remote
6. The temporary directory is cleaned up automatically

## Typed Dependencies

The `requires` field uses typed references to avoid ambiguity:
- `skill:name` — references a skill in the library catalog
- `agent:name` — references an agent in the library catalog
- `prompt:name` — references a prompt in the library catalog
- `expert:name` — references an expert in the library catalog
- `hook:name` — references a hook in the library catalog

When resolving dependencies: look up each reference in `library.yaml`, fetch all dependencies first (recursively), then fetch the requested item.

## Target Directories

By default, items are installed to the **default** directory from `library.yaml`:

```yaml
default_dirs:
    skills:
        - default: .claude/skills/
        - global: ~/.claude/skills/
    agents:
        - default: .claude/agents/
        - global: ~/.claude/agents/
    prompts:
        - default: .claude/commands/
        - global: ~/.claude/commands/
    experts:
        - default: .claude/experts/
        - global: ~/.claude/experts/
    hooks:
        - default: .claude/hooks/
        - global: ~/.claude/hooks/
```

- If the user says "global" or "globally", use the `global` directory.
- If the user specifies a custom path, use that path.
- Otherwise, use the `default` directory.

## Library Repo Sync

The library skill itself lives in `<LIBRARY_SKILL_DIR>` as a cloned git repo. When running `add` (which modifies `library.yaml`), always:
1. `git pull` in the library directory first to get latest
2. Make the changes
3. `git add library.yaml && git commit && git push`

This keeps the catalog in sync across devices.

## Example Filled Library File

```yaml
default_dirs:
  skills:
    - default: .claude/skills/
    - global: ~/.claude/skills/
  agents:
    - default: .claude/agents/
    - global: ~/.claude/agents/
  prompts:
    - default: .claude/commands/
    - global: ~/.claude/commands/
  experts:
    - default: .claude/experts/
    - global: ~/.claude/experts/
  hooks:
    - default: .claude/hooks/
    - global: ~/.claude/hooks/

library:
  skills:
    - name: testing
      description: Kent C. Dodds testing patterns with convex-test and RTL
      source: https://github.com/deepskydatahq/the-library/blob/main/skills/testing.md

    - name: test-crawler
      description: Test website crawler pipeline - Firecrawl map, classification, content quality
      source: https://github.com/deepskydatahq/the-library/blob/main/skills/test-crawler.md

  agents: []

  prompts:
    - name: execute-mission
      description: Full autopilot from mission to PRs - breakdown, triage, parallel sub-agents
      source: https://github.com/deepskydatahq/the-library/blob/main/commands/execute-mission.md

    - name: fix-pr-feedback
      description: Fix automated PR review comments - classify, fix actionable, max 2 rounds
      source: https://github.com/deepskydatahq/the-library/blob/main/commands/fix-pr-feedback.md

    - name: retro
      description: Post-implementation retrospective - discover issues, create Beads tasks
      source: https://github.com/deepskydatahq/the-library/blob/main/commands/retro.md

  experts:
    - name: product-strategist
      description: Product design advisor - user value, prioritization, jobs-to-be-done
      source: https://github.com/deepskydatahq/the-library/blob/main/experts/product-strategist.md

    - name: simplification-reviewer
      description: Reviews designs for unnecessary complexity and bloat
      source: https://github.com/deepskydatahq/the-library/blob/main/experts/simplification-reviewer.md

    - name: technical-architect
      description: Architecture advisor - patterns, API design, composability
      source: https://github.com/deepskydatahq/the-library/blob/main/experts/technical-architect.md

  hooks:
    - name: huginn-bootstrap
      description: SessionStart hook - loads project context from Huginn Memory
      source: https://github.com/deepskydatahq/the-library/blob/main/hooks/huginn-bootstrap.sh

    - name: huginn-save
      description: Stop hook - persists session learnings to Huginn Memory
      source: https://github.com/deepskydatahq/the-library/blob/main/hooks/huginn-save.sh
```
