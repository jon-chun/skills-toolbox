# Manual: `skill-creator` → `skills-toolbox` marketplace

End-to-end workflow for authoring a skill with Claude Code's built-in
`skill-creator` skill and shipping it through the `jon-chun/skills-toolbox`
marketplace (or any marketplace that follows the same layout).

This manual encodes hard-won lessons from real installation bugs that the
original test suite missed. **Read §3 ("Three invisible traps") before your
first PR** — every trap there shipped to `main` once, passed all unit
tests, and only surfaced under `claude /doctor` or in users' install logs.

---

## 0. Prerequisites

- Claude Code CLI: `claude --version` returns ≥ 2.x
- `uv` package manager: `uv --version`
- Local clone:
  ```bash
  git clone https://github.com/jon-chun/skills-toolbox
  cd skills-toolbox && uv sync --group dev
  ```
- A working `claude plugin validate .` against `main` (sanity check before
  you start changing anything).

---

## 1. Author the skill with `skill-creator`

`skill-creator` is the official Anthropic skill that scaffolds new skills.
It is loaded automatically when you install either marketplace:

| Source | Skill ID |
|---|---|
| `claude-plugins-official` | `skill-creator:skill-creator` |
| `anthropic-agent-skills` (example-skills plugin) | `example-skills:skill-creator` |

In a Claude Code session **inside the `skills-toolbox` repo**, ask:

> Use the `skill-creator` skill to create a new skill called `<name>` that
> handles `<one-line problem statement>`. Place it under `skills/<name>/`.

`skill-creator` will:

1. Ask clarifying questions about scope, triggering conditions, and which
   downstream methods belong in `references/`.
2. Generate the canonical skill folder.
3. Iterate with you on the description (the single highest-ROI decision —
   see §1.1).
4. Generate a `evals/trigger-tests.json` with should/shouldn't cases.

Expected output layout:

```
skills/<name>/
├── SKILL.md                  # YAML frontmatter + instructions
├── README.md
├── CHANGELOG.md
├── references/               # Progressive-disclosure method docs
├── scripts/                  # Optional helpers
├── assets/                   # Optional templates / fixtures
└── evals/trigger-tests.json  # ≥ 20 should/shouldn't cases
```

> If `skill-creator` writes the folder somewhere else (e.g. your home
> directory), `mv` it into `skills/<name>/` before continuing.

### 1.1 Description engineering (1024-char hard cap)

The description is the **only signal** Claude uses to decide whether to
load the skill. Hard rules enforced by
`tests/e2e/test_description_heuristics.py`:

- Starts with `Use when …`
- Third person (`the user wants …`, never `I help with …`)
- ≥ 4 distinct trigger phrases, drawn from multiple personas (engineer,
  designer, researcher, exec)
- ≤ 1024 chars (registries truncate longer)
- No XML tags, no markdown syntax, no placeholders

If the description summarises *what the skill does* instead of *when it
fires*, Claude under-triggers — the skill loads on disk but is never
invoked.

---

## 2. Integrate into the marketplace

Make exactly **four** edits. Do **not** add a per-skill
`.claude-plugin/plugin.json` — see §3 trap 3.

### 2.1 `.claude-plugin/marketplace.json`

Append a plugin entry. The shape below is load-bearing — every field
matters. See §3 for what happens if you deviate.

```json
{
  "name": "<name>",
  "source": "./",
  "skills": ["./skills/<name>"],
  "strict": false,
  "version": "0.1.0",
  "description": "Use when …",
  "author": { "name": "your-handle", "url": "https://github.com/your-handle" },
  "license": "MIT",
  "homepage": "https://github.com/jon-chun/skills-toolbox",
  "repository": "https://github.com/jon-chun/skills-toolbox",
  "keywords": ["...", "...", "...", "..."]
}
```

**Why this exact shape:**

- `source: "./"` — Claude Code copies the entire marketplace root into the
  plugin cache. Pointing `source` directly at the skill folder breaks
  loader discovery (trap 1).
- `skills: ["./skills/<name>"]` — explicit override of the default
  `<source>/skills/<name>/SKILL.md` discovery path. Required because each
  plugin in this monorepo ships exactly one skill out of many.
- `strict: false` — tells Claude Code the marketplace entry is the entire
  plugin definition. Without it, any `plugin.json` Claude Code finds
  inside the source tree (including under `skills/<name>/`) becomes a
  competing source of truth and `/doctor` reports a conflict.
- `version` — without it, Claude Code falls back to the commit SHA and
  user updates are opaque.
- `repository` — must be a **string URL**, never the npm-style
  `{ type, url, directory }` object (trap 2).

### 2.2 `skills.json` (skills.sh bundle manifest)

Append:

```json
{
  "name": "<name>",
  "path": "skills/<name>",
  "description": "…",
  "version": "0.1.0"
}
```

### 2.3 `README.md` catalog table

Add one row under `## Catalog`:

```
| [`<name>`](skills/<name>/) | One-line description | 0.1.0 |
```

### 2.4 `CHANGELOG.md` (repo-level)

Add a top entry under the next unreleased version:
`- Add <name> skill (#PR)`.

---

## 3. Three invisible traps (lessons from real bugs)

Every trap below was once shipped to `main`, passed all existing tests,
and only surfaced when a user tried to install. The repo now has
regression tests for each — do not delete or weaken them.

### Trap 1 — Wrong `source` path

❌ `"source": "./skills/<name>"` (no `skills` array)
- Claude Code's loader expects skills at `<source>/skills/<name>/SKILL.md`
  (default discovery).
- Finds zero skills inside `./skills/<name>/skills/`. Plugin installs
  successfully but loads with no skill content.

✅ `"source": "./", "skills": ["./skills/<name>"]`
- Mirrors `anthropic-agent-skills` exactly.

Regression test:
`tests/integration/test_plugin_loader_simulation.py::test_every_marketplace_plugin_resolves_to_at_least_one_skill_md`

### Trap 2 — `repository` as an object

❌ `"repository": { "type": "git", "url": "...", "directory": "..." }`
- Fails `claude plugin validate`:
  `repository: Invalid input: expected string, received object`

✅ `"repository": "https://github.com/jon-chun/skills-toolbox"`

Regression test:
`tests/unit/test_plugin_manifest.py::test_marketplace_entry_repository_is_string_if_present`

### Trap 3 — Per-skill `.claude-plugin/plugin.json`

❌ Adding `skills/<name>/.claude-plugin/plugin.json` — even if it only
declares metadata (no `skills`/`commands`/`agents`/`hooks` keys).
- `claude /doctor` reports:

  > Plugin `<name>` has conflicting manifests: both plugin.json and
  > marketplace entry specify components. Set strict: true in marketplace
  > entry or remove component specs from one location.

- Claude Code's loader treats a plugin.json sitting next to a SKILL.md as
  a second component-declaring manifest, conflicting with the
  marketplace's `skills: [...]` array.

✅ Do not create per-skill `plugin.json`. The proven-good pattern
(`anthropic-agent-skills`) ships **zero** per-skill plugin.json files.
Marketplace.json is the sole source of plugin metadata. SKILL.md
frontmatter is the source of truth for skill identity.

Regression tests:
- `tests/integration/test_plugin_loader_simulation.py::test_strict_false_skills_paths_have_no_sibling_plugin_json`
- `tests/unit/test_skill_structure.py::test_skill_has_no_per_skill_plugin_manifest`
- `tests/e2e/test_official_validator.py::test_no_per_skill_plugin_json_files_remain`

---

## 4. Verify locally

Run all checks before pushing. Each one catches a different class of bug.

```bash
# 1. Static + structural (catches all three traps and the description rules)
uv run pytest tests/ -v

# 2. Official Claude Code validator (last line of defence on schema)
claude plugin validate .

# 3. Real install round-trip from a local path
claude plugin marketplace add /absolute/path/to/skills-toolbox
claude plugin install <name>@skills-toolbox
claude plugin list | grep <name>           # expect: ✔ enabled

# 4. Verify SKILL.md ended up where the loader expects
find ~/.claude/plugins/cache/skills-toolbox -name SKILL.md
# Expect: …/<name>/<version>/skills/<name>/SKILL.md
# A path WITHOUT skills/<name>/ in it means trap 1 returned.

# 5. Cleanup
claude plugin uninstall <name>@skills-toolbox
claude plugin marketplace remove skills-toolbox

# 6. Lint + format
uv run ruff check . --fix && uv run ruff format .
```

What CI verifies on every push (`.github/workflows/ci.yml`):

- Frontmatter parses; `name` matches `^[a-z][a-z0-9-]{1,63}$`; description
  ≤ 1024 chars and starts with `Use when`.
- Marketplace + skills.json are internally consistent.
- Loader simulation: every plugin entry resolves to ≥ 1 real SKILL.md.
- `claude plugin validate .` exits 0.
- No `.claude-plugin/plugin.json` exists under `skills/`.
- `evals/trigger-tests.json` has ≥ 20 cases with ≥ 30% of each polarity.

---

## 5. Ship

```bash
git add .
git commit -m "feat: add <name> skill"
git tag -s v0.<minor>.0 -m "Release v0.<minor>.0"
git push --follow-tags
```

`git tag -s` requires a configured GPG signing key. Drop the `-s` only if
you cannot sign — note this in the PR.

---

## 6. Recover from a broken install

When a user reports `/doctor` errors or "skill not loading", walk this
list **top to bottom** and stop at the first command that fixes it.

| # | Symptom | Recovery command |
|---|---|---|
| 1 | Anything weird, first thing to try | `/reload-plugins` (rescans cache, no network) |
| 2 | Latest `marketplace.json` not picked up after a push | `/plugin marketplace update <marketplace>` |
| 3 | Plugin installed but skill not visible after a marketplace update | `/plugin uninstall <name>@<marketplace>` then `/plugin install <name>@<marketplace>` then `/reload-plugins` |
| 4 | `/doctor` reports conflicting manifests | Verify §3 trap 3 (no per-skill `plugin.json`); fix the repo, push, then steps 2 + 3 |
| 5 | Validator rejects the marketplace | `claude plugin validate .` locally; fix until ✔; push; then steps 2 + 3 |
| 6 | Stale cache after structural changes (e.g. moving SKILL.md) | `rm -rf ~/.claude/plugins/cache/<marketplace>` then `/plugin marketplace add <owner>/<repo>` to re-fetch |
| 7 | Marketplace declaration itself broken in user settings | `/plugin marketplace remove <marketplace>` then `/plugin marketplace add <owner>/<repo>` |
| 8 | Total scorched-earth reset | Steps 7 + 6 + reinstall plugin |

Reference paths:

- Marketplace declarations: `~/.claude/settings.json` (`plugins.marketplaces`)
- Cached plugins: `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`
- Cached marketplace.json: `~/.claude/plugins/cache/<marketplace>/.claude-plugin/marketplace.json`

---

## 7. Reference: known-good layout

```
skills-toolbox/
├── .claude-plugin/
│   └── marketplace.json          # Sole source of plugin metadata
├── skills/
│   └── <name>/
│       ├── SKILL.md              # YAML frontmatter + body
│       ├── README.md
│       ├── CHANGELOG.md
│       ├── references/
│       ├── scripts/
│       ├── assets/
│       └── evals/trigger-tests.json
│       # NO .claude-plugin/ directory here (trap 3)
├── skills.json                   # skills.sh bundle manifest
├── tests/                        # unit / integration / e2e
├── pyproject.toml
└── README.md
```

This matches `anthropic-agent-skills` exactly. If your layout differs in
any way, you have likely hit a trap from §3.

---

## 8. See also

- `docs/tech-spec.md` — full authoring + publishing spec (read once)
- `CONTRIBUTING.md` — first-time-contributor checklist
- `.github/workflows/ci.yml` — exact commands CI runs
- Upstream marketplace docs: <https://code.claude.com/docs/en/plugin-marketplaces>
- Upstream skill docs: <https://code.claude.com/docs/en/skills>
- Reference repo: <https://github.com/anthropics/anthropic-agent-skills>
