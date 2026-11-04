---
name: gsd:workstreams
description: Manage parallel workstreams — list, create, switch, status, progress, complete, and resume
allowed-tools:
  - Read
  - Bash
  - Grep
requires: [new-milestone, phase, progress, resume-work]
---

# /gsd:workstreams

Manage parallel workstreams for concurrent milestone work.

## Usage

`/gsd:workstreams [subcommand] [args]`

### Subcommands

| Command | Description |
|---------|-------------|
| `list` | List all workstreams with status |
| `create <name>` | Create a new workstream |
| `status <name>` | Detailed status for one workstream |
| `switch <name>` | Set active workstream |
| `progress` | Progress summary across all workstreams |
| `complete <name>` | Archive a completed workstream |
| `resume <name>` | Resume work in a workstream |

## Step 1: Parse Subcommand

Parse the user input to find the workstream operation.
If the user gives no subcommand, use `list`.

## Step 2: Execute Operation

### list
Run: `gsd-tools query workstream.list --raw --cwd "$CWD"`
Show the workstreams in a table with these columns: name, status, current phase, and progress.

### create
Run: `gsd-tools query workstream.create <name> --raw --cwd "$CWD"`
After creation, show the new workstream path and suggest next steps:
- `/gsd:new-milestone --ws <name>` to set up the milestone

### status
Run: `gsd-tools query workstream.status <name> --raw --cwd "$CWD"`
Show the phase breakdown and the state information.

### switch
Run: `gsd-tools query workstream.set <name> --raw --cwd "$CWD"`
If the runtime supports it, also set `GSD_WORKSTREAM` for the current session.
If the runtime exposes a session identifier, GSD also stores the active workstream
for the session so concurrent sessions do not overwrite each other.

### progress
Run: `gsd-tools query workstream.progress --raw --cwd "$CWD"`
Show a progress overview across all workstreams.

### complete
Run: `gsd-tools query workstream.complete <name> --raw --cwd "$CWD"`
Archive the workstream to milestones/.

### resume
Set the workstream as active and suggest `/gsd:resume-work --ws <name>`.

## Step 3: Show Results

Show the JSON output from gsd-tools query in a human-readable format.
Include the `${GSD_WS}` flag in any routing suggestions.
