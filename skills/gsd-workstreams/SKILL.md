---
name: gsd-workstreams
description: "Manage parallel workstreams — list, create, switch, status, progress, complete, and resume"
allowed-tools:
  - Read
  - Bash
  - Grep
---


# /gsd-workstreams

Manage parallel workstreams for concurrent milestone work.

## Usage

`/gsd-workstreams [subcommand] [args]`

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

Read the user's input to find the workstream operation to do.
If the input has no subcommand, use `list` by default.


## Step 2: Execute Operation

### list
Run: `gsd_run query workstream.list --raw --cwd "$CWD"`
Show the workstreams in a table with name, status, current phase, and progress.


### create
Run: `gsd_run query workstream.create <name> --raw --cwd "$CWD"`
After creation, show the path of the new workstream.
Suggest this next step:
- Use `/gsd-new-milestone --ws <name>` to set up the milestone.


### status
Run: `gsd_run query workstream.status <name> --raw --cwd "$CWD"`
Show a detailed breakdown of the phases and information about the state.


### switch
Run: `gsd_run query workstream.set <name> --raw --cwd "$CWD"`
If the runtime supports it, also set `GSD_WORKSTREAM` for the current session.
If the runtime exposes a session identifier, GSD also stores the active workstream for that session.
This prevents concurrent sessions from overwriting each other.


### progress
Run: `gsd_run query workstream.progress --raw --cwd "$CWD"`
Show an overview of progress across all workstreams.


### complete
Run: `gsd_run query workstream.complete <name> --raw --cwd "$CWD"`
Archive the workstream to milestones/.

### resume
Set the workstream as active.
Suggest `/gsd-resume-work --ws <name>`.


## Step 3: Display Results

Format the JSON output from gsd-tools query so a person can read it.
Include the `${GSD_WS}` flag in any routing suggestions.
