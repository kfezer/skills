---
name: obsidian
description: "Read, search, create, and edit notes in the Obsidian vault using the official Obsidian CLI via terminal."
version: 2.0.0
author: kfezer
license: MIT
platforms: [macos]
metadata:
  hermes:
    tags: [Obsidian, Notes, Markdown, Vault, Graph]
    related_skills: []
---

# Obsidian Vault

Use the **official Obsidian CLI**, invoked with the `terminal` tool, for all
Obsidian work — reading, searching, creating, editing, backlinks/graph
queries, canvas files, tags, tasks, and live-app state (active file, open
tabs, daily note). Do not use generic file tools (`read_file`/`write_file`)
or raw shell `cat`/`grep`/`find` against the vault directory directly —
the CLI is aware of Obsidian's link graph, frontmatter, and sync state in
ways raw file access is not, and it works correctly regardless of the
`terminal` tool's working directory.

**There is no tool named `obsidian`.** Call the `terminal` tool itself,
passing it a `command` string that runs the CLI as a shell command — do not
call a function/tool literally named `obsidian`, it does not exist.

## Invocation

Call the binary by its full path — do not assume it's on PATH:

```
/usr/local/bin/obsidian <command> key=value ...
```

Example, via `terminal`:

```
terminal(command="/usr/local/bin/obsidian read path=\"Inbox/Note.md\"")
```

There is exactly one vault on this machine, named `Documents`, so `vault=`
targeting is not needed (adjust the `vault=<name>` option if you have
multiple vaults). Options with spaces need quoting: `name="My Note"`. Run
`/usr/local/bin/obsidian help <command>` for a specific command's full
option list if unsure — if the CLI errors on an unrecognized command with a
"did you mean" suggestion, check `help <suggested-command>` before trusting
it; the suggestion can be a real but semantically different command (e.g.
`outline` shows headings, not links).

**Content argument gotcha**: `content=<text>` is a literal CLI argument, not
a shell heredoc — the CLI has its own `\n`/`\t` escape convention for
newlines/tabs *within* that value (see `\n` for newline, not a real
line break). Wrap the whole value in double quotes and backslash-escape any
literal double quotes inside it. For longer or multi-section notes, prefer
several `append`/`prepend` calls over one giant `create` call — shorter
argument strings are far less error-prone than one large escaped blob.

## Never dump loose notes at the vault root

Every new note must go inside a real folder:

- Use an existing topic folder when the content clearly belongs there —
  check with `files folder=<name>` or `folders` first if unsure what exists.
- For anything uncategorized or a quick capture, default to `Inbox/` (e.g.
  `path="Inbox/My New Note.md"`) if that folder exists in your vault, or
  whatever your vault's actual unsorted-notes convention is.
- Never call `create` with a bare `path=SomeNote.md` (no folder segment).

This matters beyond tidiness: notes living loose at the vault's iCloud
container root may not be exposed correctly to the Obsidian mobile app,
which expects proper vault structure — so this is a correctness rule, not
just a style preference.

## Common commands

- **Read**: `read path="Folder/Note.md"` (or `file="Note Name"` to resolve
  by name like a wikilink, when the path is unknown).
- **List**: `files` (whole vault), `files folder="Inbox"` (one folder),
  `folders` (list folders).
- **Search**: `search query="text"` (filenames+content), `search:context
  query="text"` for matching line context.
- **Create**: `create path="Inbox/New Note.md" content="..."` — see the
  vault-root rule above. Add `overwrite` to replace an existing file
  (be careful — confirm intent before overwriting real content).
- **Append/Prepend**: `append path="..." content="..."`, `prepend
  path="..." content="..."`.
- **Targeted property edit**: `property:set name=<field> value=<value>
  path="..."` rather than rewriting a whole note to change frontmatter.
- **Delete**: `delete path="..." permanent` (omit `permanent` to use trash
  instead, which is safer for anything not clearly disposable).
- **Move/rename**: `move path="old.md" to="NewFolder/new.md"`.

## Graph and links

This is the CLI's real advantage over raw file access — link-graph queries
are computed by Obsidian itself, not by grepping for `[[wikilinks]]`:

- `backlinks path="Note.md"` — notes that link *to* this one.
- `links path="Note.md"` — notes this one links *to*.
- `orphans` — notes nothing links to.
- `deadends` — notes with no outgoing links.
- `unresolved` — links pointing at notes that don't exist yet.
- `tags` / `tag name=<tag>` — tag inventory and per-tag file lists.

Use `[[Note Name]]` wikilink syntax in note content to create links; the
graph commands above pick them up automatically once the note is written.

## Canvas files

`.canvas` files are just JSON. Use `read`/`create` as with any other file,
treating `content` as minified JSON (no embedded newlines to escape) —
single-quote the whole value in the shell command, since JSON itself only
uses double quotes internally, so no escaping is needed that way. Validate
the JSON structure before writing.

## Live-app state

- `vault info=files` — vault summary (file/folder counts, size).
- Omit `file=`/`path=` on most commands (e.g. plain `read`, `tags active`)
  to default to the file currently open in the Obsidian app.
- `daily:read` / `daily:append content="..."` — today's daily note.
- `recents`, `tabs`, `bookmarks` — what the user has open/saved right now.
- `sync:status` — whether iCloud sync is current (useful if a write seems
  to not be showing up elsewhere).

## Reliability note for small/local models

If your agent runs a small local model, don't rely solely on this file
being loaded on demand (many agent frameworks index skills lazily — a
one-line name+description summary in the system prompt, with the full body
fetched only via a second tool call). Small models can be unreliable at
that two-hop retrieval and will fall back to a generic file tool instead,
which is silently wrong here (see the vault-root note above — file tools
resolving vault-relative paths against the wrong base directory is a real,
observed failure, not hypothetical). If you see this happening, put the
load-bearing minimum (CLI path, the "no tool named `obsidian`" note, and
the vault-root rule) directly into whatever context is *always* injected
for your agent (a system prompt / always-on user-profile file), and keep
this file as the fuller reference.
