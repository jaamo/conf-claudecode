---
name: tmux
description: Interact with tmux sessions, panes, and windows. Activates when users mention tmux, running processes, test runners, REPLs, SQL clients, panes, or want to send commands to or read output from interactive shell sessions.
version: 1.0.0
user-invocable: false
---

# Tmux Session Interaction

You have access to tmux for interacting with running processes, test runners, Node/tsx REPLs, SQL clients, and other interactive shell sessions.

## Core Principles

- Use `tmux` commands via the Bash tool to send keys and read output from tmux panes.
- Never blindly send commands — always check which sessions/windows/panes exist first.
- After sending a command, wait briefly then capture the output to verify the result.

## Available Operations

### 1. Discover Sessions
Before doing anything, list available tmux sessions and panes:
```bash
tmux list-sessions 2>/dev/null
tmux list-panes -a -F '#{session_name}:#{window_index}.#{pane_index} [#{pane_title}] (#{pane_current_command})' 2>/dev/null
```

### 2. Label Panes, Windows, and Sessions
Use semantic names so you can identify panes by purpose rather than memorizing numbers. Panes are always addressed by number, but titles serve as human-readable labels.

```bash
# Rename a session
tmux rename-session -t 0 myproject

# Rename a window
tmux rename-window -t myproject:0 dev

# Set a pane title (metadata — pane is still addressed by number)
tmux select-pane -t myproject:0.0 -T "tests"
tmux select-pane -t myproject:0.1 -T "node-repl"
tmux select-pane -t myproject:0.2 -T "tsc-watch"
tmux select-pane -t myproject:0.3 -T "psql"
```

When discovering panes, use the title to find the right pane by purpose:
```bash
# Find the pane labeled "tests"
tmux list-panes -a -F '#{session_name}:#{window_index}.#{pane_index} #{pane_title}' | grep tests
```

If the user refers to a pane by a semantic name (e.g., "the node pane", "the vitest pane", "the test runner"), match it against pane titles. If no titles are set, offer to label them.

### 3. Read Pane Output
Capture recent output from a tmux pane:
```bash
# Last 50 lines from a specific pane
tmux capture-pane -t <session>:<window>.<pane> -p -S -50

# Or capture the entire scrollback
tmux capture-pane -t <session>:<window>.<pane> -p -S -
```

### 4. Send Commands / Keys
Send a command to a tmux pane (press Enter to execute):
```bash
tmux send-keys -t <session>:<window>.<pane> '<command>' Enter
```

For special keys:
```bash
tmux send-keys -t <session>:<window>.<pane> C-c   # Ctrl+C
tmux send-keys -t <session>:<window>.<pane> C-d   # Ctrl+D (EOF)
tmux send-keys -t <session>:<window>.<pane> C-l   # Clear screen
```

### 5. Send and Read Pattern
The typical workflow is:
1. Send a command to the pane
2. Wait briefly for output (`sleep 1` or longer for slow commands)
3. Capture the pane output to read the result

```bash
tmux send-keys -t mysession:0.0 'node -e "console.log(1+1)"' Enter && sleep 1 && tmux capture-pane -t mysession:0.0 -p -S -20
```

### 6. Running Tests
When asked to run tests:
1. Find the appropriate tmux pane (or use the user-specified one)
2. Send the test command
3. Wait for completion (use longer sleeps for test suites)
4. Capture and analyze the output
5. Report results back to the user

### 7. Node REPL / SQL Client
When sending multi-line TypeScript or SQL:
- For TypeScript: a bare `node` REPL handles pasted multi-line input poorly. Prefer
  `node -e '...'` or `npx tsx -e '...'` for short snippets, and `npx tsx <file>` to run
  a TS file.
- For SQL: send the full query followed by Enter
- Always capture output after to verify the result

### 8. Watch Processes
Long-lived watchers (`vitest --watch`, `tsc --watch`, `next dev`, `nodemon`) re-run on
file save rather than on a command you send. This breaks the send → `sleep 1` → capture
pattern:
- After editing a file, the watcher may not have picked it up yet — wait longer
  (`sleep 3` or more) before the first capture.
- Capture twice if the first result looks stale or shows a run still in progress.
- Look for the watcher's completion marker (e.g. vitest's summary line, tsc's
  "Found N errors. Watching for file changes.") rather than assuming the last visible
  output is final.
- To force a re-run without editing a file, most watchers accept a key in the pane
  (vitest: `Enter` to rerun, `a` to run all).

## Important Notes

- If the user specifies a session/pane name, use it directly.
- If not specified, discover available sessions and ask which one to use (or use the most obvious one).
- For long-running commands, use longer sleep intervals before capturing output.
- If a command produces a lot of output, capture more scrollback lines (`-S -200`).
- When output seems incomplete, capture again with more scrollback.
- Use `C-c` to interrupt stuck/hanging commands if needed.
