---
layout: post
title: "Persistent Workspaces in Terminal Multiplexers"
date: 2026-09-02
categories: terminal productivity architecture
series: "AI coding agent productivity"
---

When you type `exit` in the last shell of a workspace, the multiplexer destroys the workspace record, drops its metadata, and throws your cursor into whichever space happens to be next.

In traditional terminal multiplexers (tmux, screen, zellij) and modern agent-first multiplexers like [Herdr](https://github.com/herdrdev/herdr), workspace lifecycle is strictly derived from active pane and tab counts:

$$\text{Workspace Lifespan} = \min \{ t \mid \text{active\_panes}(t) = 0 \}$$

This creates a fundamental friction when working across several concurrent projects. A workspace is not just a collection of shell processes; it is a **mental anchor** and a **spatial context** — tied to a project name, a repository root, and a dedicated role. When you finish a shell session or terminate a run, you want the space to stay in place with a clean prompt in that working directory, ready for the next task.

This post documents how to decouple workspace identity from ephemeral process life using event-driven multiplexer plugins, the distinction between process exit hooks and workspace teardown, and how to maintain focus stability across shell lifecycles.

## The Architectural Mismatch: Containers vs. Identities

In most multiplexer architectures, the hierarchy is hierarchical and bottom-up:

| Concept | Lifetime Model | What Happens When Empty |
|---|---|---|
| **Server / Daemon** | Outlives all clients | Persists across disconnects |
| **Workspace (Space)** | Bounded by child tabs | Purged immediately on zero panes |
| **Tab / Window** | Bounded by child panes | Destroyed on zero panes |
| **Pane / PTY** | Bounded by OS process | Terminates on process exit |

When the last child process inside a workspace dies (e.g., `exit`, Ctrl-D, or a terminated script), the process lifecycle cascades upward:

```text
Process Exit (PID dies)
   └── Pane Removed
         └── Tab Collapses (0 panes left)
               └── Workspace Deleted (0 tabs left)
                     └── Focus Diverts to Adjacent Workspace
```

By the time the daemon records `workspaces.remove(index)`, the space's human identity — its name, public slot, and working directory — is already gone.

## The Event Boundary Trap: `workspace.closed` vs. `pane.exited`

The obvious first approach is to listen to the multiplexer's `workspace.closed` event.

In Herdr's [socket and plugin API](https://herdr.dev/docs/plugins/), plugins can subscribe to lifecycle hooks in `herdr-plugin.toml`. However, inspecting the core event dispatch reveals a subtle distinction:

| Event | Dispatched When | Payload Data |
|---|---|---|
| `workspace.closed` | User explicitly issues `workspace close` / kill | Full workspace info (id, label) |
| `pane.exited` | A child process inside a terminal pane terminates | Pane ID and Workspace ID only |

When a shell exits naturally, the engine dispatches `pane.exited`, observes that the pane count has dropped to zero, and quietly frees the workspace structure **without** emitting `workspace.closed`.

Subscribing only to `workspace.closed` therefore leaves interactive terminal exits completely unhandled. A reliable lifecycle manager must subscribe to both:

```toml
[[events]]
on = "workspace.closed"
command = ["node", "bin/on-closed.js"]

[[events]]
on = "pane.exited"
command = ["node", "bin/on-pane-exited.js"]

[[events]]
on = "workspace.created"
command = ["node", "bin/track.js"]
```

## Reconciling Ephemeral IDs to Human Spaces

Inside the multiplexer IPC bus, workspaces are identified by transient runtime handles (e.g., `w0`, `w1A`, `wY`). These change on every creation and restart.

To keep spaces truly persistent, the configuration layer must deal strictly with stable human attributes:
- **`label`**: The project or domain name (`"backend"`, `"frontend"`, `"infra"`, `"research"`).
- **`cwd`**: The repository root path.

A declarative state registry bridges the two:

```json
{
  "autoPersist": true,
  "spaces": [
    {
      "label": "infra",
      "cwd": "C:/projects/infra"
    },
    {
      "label": "backend",
      "cwd": "C:/projects/backend"
    }
  ],
  "ignored": []
}
```

### The State Synchronization Loop

When a new workspace or pane is created, `track.js` queries the live pane metadata to resolve the real current working directory dynamically:

```javascript
const workspaces = listWorkspaces();
const panes = listPanes();

const cwdByWs = new Map();
for (const p of panes) {
  if (p.workspace_id && p.cwd && (!cwdByWs.has(p.workspace_id) || p.focused)) {
    cwdByWs.set(p.workspace_id, p.cwd);
  }
}
```

When `pane.exited` fires, the handler:
1. Waits a single tick for the daemon to clean up dead process handles;
2. Checks whether the workspace still exists in the active workspace list;
3. If the workspace has collapsed, matches its prior label against the persistent registry;
4. Issues a workspace recreation request with the original `--cwd` and `--label`.

```javascript
function onPaneExited(eventData) {
  const wsId = eventData.data.workspace_id;

  setTimeout(() => {
    const active = listWorkspaces();
    if (active.some(w => w.workspace_id === wsId)) {
      return; // Workspace still has other active panes
    }

    const target = registry.spaces.find(s => s.workspace_id === wsId);
    if (target && !registry.ignored.includes(target.label)) {
      recreateWorkspace(target);
    }
  }, 100);
}
```

## Preventing Focus Drift

When a workspace collapses, the multiplexer daemon immediately moves focus to the adjacent surviving workspace. If the plugin recreates the space in the background without focus management, the user's focus remains stranded in the adjacent space.

To provide an unbroken terminal workflow, the recreation sequence captures the new workspace identifier and explicitly requests a focus switch:

```javascript
const res = runHerdr(["workspace", "create", "--cwd", space.cwd, "--label", space.label]);
const newWs = res.data?.result?.workspace;

if (newWs && newWs.workspace_id) {
  runHerdr(["workspace", "focus", newWs.workspace_id]);
}
```

From the user's perspective, typing `exit` behaves like a prompt reset: the scrollback clears, the working directory is preserved, focus never flickers to an unrelated space, and the workspace remains anchored.

## Summary

Decoupling workspace identity from process lifespan turns ephemeral multiplexer tabs into durable project workbenches:

1. **Trap the right layer**: Hook both `pane.exited` and `workspace.closed` to catch natural shell exits alongside explicit closes.
2. **Track human attributes, not runtime handles**: Keep state files centered on project labels and filesystem roots.
3. **Reclaim focus explicitly**: Always refocus the recreated workspace to prevent disruptive focus jumps.

## Source

- [Herdr Multiplexer](https://github.com/herdrdev/herdr)
- [Herdr Plugin System Documentation](https://herdr.dev/docs/plugins/)
