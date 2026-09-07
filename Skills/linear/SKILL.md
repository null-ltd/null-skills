---
name: linear
description: Linear CLI mechanics. Use for any `linear` command: flags, file-based markdown bodies, issues, relations, projects, documents, initiatives, and the GraphQL fallback when no flag covers the operation.
disable-model-invocation: false
allowed-tools: Bash(linear:*), Bash(curl:*)
---

How to drive the `linear` CLI. What to track, when to move an issue, and where documents live are other skills' decisions; this file is flags and traps.

## The repo pin

A repo is Linear-tracked when its root holds `.linear.toml`, which supplies the default workspace and team for commands that use them:

```toml
workspace = "<workspace-slug>"
team_id = "<TEAM-KEY>"
```

`linear config` generates it interactively. `linear auth whoami` checks the login; if it fails, the user runs `linear auth login` themselves.

## Markdown goes through files

Any flag that takes markdown has a file form. Use it for anything longer than one line; inline flags mangle newlines and shell-escape badly.

| Command | File flag |
|---|---|
| `issue create`, `issue update` | `--description-file` |
| `issue comment add`, `issue comment update` | `--body-file` |
| `project create` | `--content-file` (overview) |
| `document create`, `document update` | `--content-file` |

## Recipes

Query issues. `issue list` is an alias of `issue mine` and shows only your issues; `issue query` covers a team or project:

```bash
linear issue query --project "<project>" --state unstarted --json
linear issue query --team WEB --label Bug --updated-after 2026-01-01
```

Create an issue, one label and a priority, into a project:

```bash
linear issue create --title "<imperative title>" --description-file ./spec.md \
  --project "<project>" --label Feature --priority 3 --state Todo --no-interactive
```

Move state, comment, wire a blocking edge:

```bash
linear issue update WEB-12 --state "In Progress"
linear issue comment add WEB-12 --body-file ./comment.md
linear issue relation add WEB-14 blocked-by WEB-12
linear issue relation list WEB-14
```

Create a project under an initiative with an overview:

```bash
linear project create --name "<name>" --team WEB --initiative "<initiative>" \
  --description "<one sentence, 255 chars max>" --content-file ./overview.md --status planned
linear project update <project> --status started
```

Attach a document to an issue or project:

```bash
linear document create --issue WEB-12 --title Plan --content-file ./plan.md
linear document update <document> --content-file ./plan.md
```

Inline image in a comment:

```bash
linear issue comment add WEB-12 --attach ./screenshot.png
```

## Writes

- **Confirm the target first.** With no `.linear.toml` at the repo root, pass `--workspace <slug>` explicitly on writes instead of trusting whatever workspace `linear auth default` last set. Add `--team <key>` only when the command's help documents it and its meaning matches the intended operation. `issue comment add` has no `--team`; `document update --team` changes the document's attachment, not its workspace scope. Then confirm the identifier in the intended workspace is the thing you mean: `linear issue view WEB-12 --json` and read the title before `issue update`, `comment add`, or `relation add` touches it.
- **Replace a body from its current version.** `document update --content-file` and the `projectUpdate(content:)` fallback replace the whole body, so a draft written from anything but the current body erases edits made in Linear. Fetch it (`linear document view <id> --json`; for a project overview, query `project(id:) { content }` through `linear api`), make the edit on that copy, write it, then fetch again and diff against what you sent. `document update` stops when inline comments would lose their anchors; `--force` overrides, and only after the user says so.

## Traps

- `issue comment create` does not exist; the verb is `add`.
- `issue start` creates and checks out a git branch as a side effect. Move state with `issue update --state`.
- `issue attach` makes a sidebar link and never renders an image inline; `comment add --attach` does.
- `project update` has no `--content` or `--content-file`. Changing an overview after creation is the GraphQL fallback below.
- `project view` has no `--json`; a project's overview is read through the GraphQL fallback.
- `project --description` is capped at 255 characters by the API; the overview (`--content-file`) is not.
- `issue query --state` and `issue list --state` filter by state type, never by name: `backlog`, `unstarted`, `started`, `completed`, `canceled`, `duplicate`. When a team has two states of one type, read `state.name` in the JSON to tell them apart.
- `--no-pager` exists only on `issue list` and `issue query`; other commands error on it.
- `issue list` needs `--team <key>` unless the repo pin supplies it.

## Reference

`linear <command> --help` lists a command's subcommands and `linear <command> <subcommand> --help` its flags; both are the installed version's own text, so nothing here repeats them. The commands are `api`, `auth`, `config`, `cycle`, `document`, `initiative`, `initiative-update`, `issue`, `label`, `milestone`, `project`, `project-update`, `schema`, `team`, and `user`. Curated examples for initiatives, labels, projects, and bulk operations are in [references/organization-features.md](references/organization-features.md).

## GraphQL fallback

Reach for `linear api` only when no command or flag covers the operation. Find the shape in the schema first:

```bash
linear schema -o "${TMPDIR:-/tmp}/linear-schema.graphql"
grep -A 30 "^input ProjectUpdateInput" "${TMPDIR:-/tmp}/linear-schema.graphql"
```

A query with non-null markers (`String!`) goes through a heredoc; inline quoting breaks on the `!`:

```bash
linear api '{ viewer { id name } }'

linear api --variable id=<project-id> --variable content="$(cat ./overview.md)" <<'GRAPHQL'
mutation($id: String!, $content: String!) {
  projectUpdate(id: $id, input: { content: $content }) { success }
}
GRAPHQL

linear api --variables-json '{"filter": {"state": {"name": {"eq": "In Progress"}}}}' <<'GRAPHQL'
query($filter: IssueFilter!) { issues(filter: $filter) { nodes { identifier title } } }
GRAPHQL
```

Raw HTTP, when the CLI's `api` command is not enough:

```bash
curl -s -X POST https://api.linear.app/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: $(linear auth token)" \
  -d '{"query": "{ viewer { id } }"}'
```
