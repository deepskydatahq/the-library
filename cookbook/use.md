# Use a Skill from the Library

## Context
Pull a skill, agent, prompt, expert, or hook from the catalog into the local environment. If already installed locally, overwrite with the latest from the source (refresh).

## Input
The user provides a skill name, description, or `all` to install everything.

## Steps

### 0. Check for "use all"
If the user said `/library use all`:
- Read `library.yaml` and collect every entry across all sections
- If user said "globally", use the `global` directory for all items
- For each entry, run steps 3–6 below (resolve deps, determine target, fetch, verify)
- Group items from the same GitHub repo into a single clone to avoid redundant fetches
- Show a summary table at the end with all installed items and their paths
- Skip step 2 (no need to search — we're installing everything)

### 1. Sync the Library Repo
Pull the latest catalog before reading:
```bash
cd <LIBRARY_SKILL_DIR>
git pull
```

### 2. Find the Entry
- Read `library.yaml`
- Search across `library.skills`, `library.agents`, `library.prompts`, `library.experts`, and `library.hooks`
- Match by name (exact) or description (fuzzy/keyword match)
- If multiple matches, show them and ask the user to pick one
- If no match, tell the user and suggest `/library search`

### 3. Resolve Dependencies
If the entry has a `requires` field:
- For each typed reference (`skill:name`, `agent:name`, `prompt:name`):
  - Look it up in `library.yaml`
  - If found, recursively run the `use` workflow for that dependency first
  - If not found, warn the user: "Dependency <ref> not found in library catalog"
- Process all dependencies before the requested item

### 4. Determine Target Directory
- Read `default_dirs` from `library.yaml`
- If user said "global" or "globally" → use the `global` path
- If user specified a custom path → use that path
- Otherwise → use the `default` path
- Select the correct section based on type (skills/agents/prompts)

### 5. Fetch from Source

**If source is a local path** (starts with `/` or `~`):
- Resolve `~` to the home directory
- Get the parent directory of the referenced file
- For skills: copy the entire parent directory to the target:
  ```bash
  cp -R <parent_directory>/ <target_directory>/<name>/
  ```
- For agents: copy just the agent file to the target:
  ```bash
  cp <agent_file> <target_directory>/<agent_name>.md
  ```
- For prompts: copy just the prompt file to the target:
  ```bash
  cp <prompt_file> <target_directory>/<prompt_name>.md
  ```
- For experts: copy just the expert file to the target:
  ```bash
  cp <expert_file> <target_directory>/<expert_name>.md
  ```
- For hooks: copy just the hook file to the target and preserve execute permissions:
  ```bash
  cp <hook_file> <target_directory>/<hook_name>.sh
  chmod +x <target_directory>/<hook_name>.sh
  ```
- If the agent, prompt, or expert is nested in a subdirectory, copy the subdirectory to the target as well, creating the subdir if it doesn't exist. This is useful because it keeps items grouped together.

**If source is a GitHub URL**:
- Parse the URL to extract: `org`, `repo`, `branch`, `file_path`
  - Browser URL pattern: `https://github.com/<org>/<repo>/blob/<branch>/<path>`
  - Raw URL pattern: `https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path>`
- Determine the clone URL: `https://github.com/<org>/<repo>.git`
- Determine the parent directory path within the repo (everything before the filename)
- Clone into a temporary directory:
  ```bash
  tmp_dir=$(mktemp -d)
  git clone --depth 1 --branch <branch> https://github.com/<org>/<repo>.git "$tmp_dir"
  ```
- Copy the parent directory of the file to the target:
  ```bash
  cp -R "$tmp_dir/<parent_path>/" <target_directory>/<name>/
  ```
- Clean up:
  ```bash
  rm -rf "$tmp_dir"
  ```

**If clone fails (private repo)**, try SSH:
  ```bash
  git clone --depth 1 --branch <branch> git@github.com:<org>/<repo>.git "$tmp_dir"
  ```

### 6. Verify Installation
- Confirm the target directory exists
- Confirm the main file (SKILL.md, AGENT.md, expert .md, hook .sh, or prompt file) exists in it
- Report success with the installed path

### 7. Confirm
Tell the user:
- What was installed and where
- Any dependencies that were also installed
- If this was a refresh (overwrite), mention that
