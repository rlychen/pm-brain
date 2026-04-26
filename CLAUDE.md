# Vault-level instructions for Claude Code

This file applies when Claude Code is operating on the PM-Brain Obsidian vault at `~/Documents/PM-Brain`. Read it before reading, writing, or moving notes. Project-specific `CLAUDE.md` files inside `01-Projects/[project-name]/` override anything here for work on that project.

The global `~/.claude/CLAUDE.md` covers communication style, engagement rules, and code preferences. This file covers the vault's organizing conventions.

## Folder structure

The vault uses a numbered PARA layout:

- `00-Inbox/` — capture, untriaged
- `01-Projects/` — active outcomes with end states
- `02-Areas/` — ongoing responsibilities
- `03-Resources/` — topical reference material
- `04-Archive/` — completed projects, deprecated areas, retired resources
- `05-Daily/` — daily notes, managed by the Periodic Notes plugin
- `06-Templates/` — note templates for Templater plugin
- `07-Prompts/` — reusable system prompts and evaluation prompts
- `08-Meta/` — vault config, conventions, this file

## PARA semantics

**Projects** have end states. Things you'd report *progress* on. They have deadlines or completion criteria. When a project is done, it moves to `04-Archive/`.

**Areas** are ongoing responsibilities. Things you'd report *status* on. They have no end state. They are maintained indefinitely.

**Resources** are topical reference material — articles, frameworks, code snippets, technical references — not tied to active work.

**Archive** is everything from the other three categories that is no longer active. Not deleted, just out of view.

When in doubt about whether something is a Project or an Area, prefer Project. Finite scope is more useful for action than ongoing topics.

## Where new notes go

**When ambiguous, default to `00-Inbox/`.** Capture first, categorize later. Do not guess at PARA placement when the user has not specified.

**When the user has clearly named a project, area, or resource**, write the note into the appropriate existing subfolder.

**When a project has no folder yet**, ask whether to create one before doing so.

## Folder creation rules

You may create new **subfolders** inside existing top-level PARA folders. For example, a new project folder under `01-Projects/` is fine.

You may **not** create new top-level numbered folders or modify the numbered scheme. The structure (`00-Inbox`, `01-Projects`, ..., `08-Meta`) is fixed.

## Hub notes

Each project and area folder has a `_hub.md` file. The underscore prefix sorts it to the top of the folder alphabetically. The hub orients — it explains what the folder is, current state, and links to the important notes inside.

**Read the hub first when entering a project or area folder.** It is your starting context for that domain.

**When adding a new note to a project or area folder**, append a link to the hub's notes section. Do not rewrite, summarize, or restructure existing hub content without explicit permission.

**When asked to update the hub itself**, do so. The hybrid is: passive maintenance (append-only) by default, active maintenance only when requested.

## Naming conventions

**Filenames in kebab-case.** Lowercase, hyphens between words, no spaces. Example: `network-design-prd.md`, not `Network Design PRD.md`.

**Note titles (H1 inside the file) in sentence case.** Example: `# Network design PRD`, not `# Network Design PRD`.

**Dates in `YYYY-MM-DD` format everywhere.** Daily notes, dated entries in journals, references to specific dates in note bodies. Do not mix formats.

## Conventions not yet adopted

These are visible in many Obsidian vaults but are deliberately not in use here. Do not add them to notes.

**Frontmatter (YAML at the top of notes).** Not in use. Do not add frontmatter blocks to new notes you create. May be added later when needed.

**Dataview queries.** Not in use. Do not write Dataview code blocks. Hubs are hand-maintained for now.

**Tags (`#tag-name`).** Use only when natural and useful. Do not tag aggressively or add tags as a categorization layer. Folder structure is the primary organization.

## Off-limits paths

**Never read or write:**

- `.obsidian/` — Obsidian's config and plugin state. Modifying breaks the application.
- `.smart-env/` — Smart Connections embedding cache. Auto-generated, regenerable.
- `.git/` — git's internal state.

**Read freely, but ask before writing:**

- `08-Meta/` — vault config, conventions, this file. Edits here change Claude Code's own operating rules. Always ask before modifying.
- `04-Archive/` — historical record. Treat as append-only or read-only. Ask before adding to it or modifying existing archive content.

## Daily notes

Daily notes live in `05-Daily/` and are managed by the Periodic Notes plugin. Filename format is `YYYY-MM-DD.md`.

When asked to add to today's daily note, write to the file with today's date. If it does not exist yet, create it. Do not create daily notes for past or future dates without explicit user instruction.

## When the user is not specific

If the user says "save this" or "make a note about this" without naming a destination, default to creating a note in `00-Inbox/` with a kebab-case filename derived from the content. Do not guess at PARA placement.

If the user says "add this to my [topic] notes" and there is more than one plausible destination, ask which one.

## Operating principles

**This file evolves with the vault.** As patterns become clear or break down, update this file. Edits take effect at the next Claude Code session start.

**When uncertain, ask.** The vault is a working space, not a sandbox — wrong-place notes and broken hub structure create real friction over time. Better to ask than to guess.
