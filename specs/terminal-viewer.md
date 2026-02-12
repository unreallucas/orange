# Terminal Viewer

Read-only preview of agent terminal output within the dashboard. View what an agent is doing without leaving the dashboard or attaching to its tmux session.

## Motivation

Currently, seeing agent output requires attaching to the tmux session (Enter key), which leaves the dashboard. This breaks monitoring flow when overseeing multiple agents. The terminal viewer provides a quick read-only preview inline.

## Keybinding

`t` — open terminal viewer for selected task.

| Context | Behavior |
|---------|----------|
| Task with live session | Open terminal viewer |
| Task without session | Show error "No active session" |
| Inside terminal viewer | Close viewer (toggle) |

## Layout

```
 coffee/login-fix  ●  working                                    [terminal]
──────────────────────────────────────────────────────────────────────────────
 $ bun test src/auth/login.test.ts
 ✓ validates email format (2ms)
 ✓ rejects empty password (1ms)
 ✓ creates session on success (5ms)

 3 tests passed

 $ git add src/auth/login.ts src/auth/login.test.ts
 $ git commit -m "feat(auth): add login form validation"
 [login-fix abc1234] feat(auth): add login form validation
  2 files changed, 89 insertions(+), 3 deletions(-)

 Agent thinking...
──────────────────────────────────────────────────────────────────────────────
 j/k:scroll  Enter:attach  Esc:close                             lines 45-67
```

**Header**: `project/branch  ● status` with `[terminal]` tag right-aligned.

**Body**: Captured terminal output, filling available height.

**Footer**: Keybindings + line position indicator (like view mode).

## Behavior

### Capture

Uses existing `capturePane(session, lines)` from tmux abstraction.

- Capture **500 lines** of scrollback (enough context without being excessive)
- Refresh every **2 seconds** while viewer is open
- Non-blocking — capture failures silently skip that refresh cycle

### Scrolling

- `j`/`k` (or arrow keys) scroll through captured output
- **Auto-scroll**: When scroll position is at the bottom, new output keeps the view pinned to the bottom (like `tail -f`)
- Scrolling up disengages auto-scroll; scrolling back to bottom re-engages
- `G` jumps to bottom (re-engages auto-scroll)
- `g` jumps to top

### Actions from Viewer

| Key | Action |
|-----|--------|
| `j`/`k`/`up`/`down` | Scroll |
| `g` | Jump to top |
| `G` | Jump to bottom |
| `Enter` | Attach to session (exit dashboard) |
| `Esc`/`t`/`q` | Close viewer, return to task list |

### Auto-Close

Viewer auto-closes when:
- Task status becomes terminal (done/cancelled)
- Session dies (show brief "Session ended" message, then close)

## State

```typescript
interface TerminalViewerData {
  active: boolean;
  taskId: string | null;
  content: string[];          // Captured lines
  scrollOffset: number;       // Current scroll position
  autoScroll: boolean;        // Pinned to bottom?
  refreshTimer: ReturnType<typeof setInterval> | null;
}
```

Added to `DashboardStateData` as `terminalViewer: TerminalViewerData`.

## Integration Points

### Dashboard State (`state.ts`)

- `enterTerminalViewer()` — validate session exists, start capture + refresh timer
- `exitTerminalViewer()` — clear timer, reset state
- `handleTerminalViewerInput(key)` — scroll, attach, close
- `refreshTerminalCapture()` — called by timer, updates content

### Dashboard Render (`index.ts`)

- New overlay (same pattern as `viewOverlay`) — hides task list, shows captured output
- Reuses the scrolling line rendering pattern from view mode

### Input Routing (`index.ts`)

- When `terminalViewer.active`, route keys to `handleTerminalViewerInput`
- Same priority as existing view/confirm/create mode checks

### Footer Keys (`getContextKeys`)

Add `t:terminal` to footer for tasks with live sessions:

| Task State | Addition |
|------------|----------|
| Working (alive) | `t:terminal` |
| Agent-review (alive) | `t:terminal` |
| Clarification (alive) | `t:terminal` |
| Reviewing (alive) | `t:terminal` |
| Stuck (alive) | `t:terminal` |

### Polling

The 2s refresh timer is independent of the dashboard's 30s poll cycle. It runs only while the terminal viewer is open and stops on close.

## Edge Cases

| Case | Behavior |
|------|----------|
| Session dies while viewing | Show "Session ended" message, auto-close after 1s |
| Task status changes while viewing | Refresh header (status/icon), keep viewing |
| Task becomes terminal while viewing | Auto-close |
| Capture returns empty | Show "(no output)" placeholder |
| Multiple rapid refreshes | Timer-based, no overlap (skip if previous capture still pending) |
| Terminal resize | Re-render with new dimensions, preserve scroll position |

## Testing

- Unit tests on state machine: enter/exit, scroll, auto-scroll logic
- Mock tmux `capturePane` returns predetermined output
- Test auto-close on session death
- Test scroll position persistence across refreshes
- Visual test: verify layout matches spec
