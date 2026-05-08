# tomenet-tmux

Run TomeNET's GCU (curses) client across multiple tmux panes, one pane per
ang_term. Each pane gets its own pty; the game writes to all of them through
a single client process via the `--term=IDX:tty` argument.

## Layout

```
+-----------------------+----------+
|                       | term-1   |   <- msg/chat
|                       +----------+
|   term-0 (main)       | term-2   |   <- inventory
|                       +----------+
|                       | term-3   |   <- recall/other
+-----------------------+----------+
```

50% main on the left, 50% right column split into three equal stripes.

## Requirements

- `tmux` (3.0+ recommended; tested on 3.5a).
- `tomenet` built from this repo. The wrapper looks for `tomenet` at the
  repo root first, then in `src/`.

## Usage

```
contrib/tmux-multiterm/tomenet-tmux [tomenet args...]
```

Pass-through examples:

```
contrib/tmux-multiterm/tomenet-tmux
contrib/tmux-multiterm/tomenet-tmux -lMyChar mypass europe.tomenet.eu
contrib/tmux-multiterm/tomenet-tmux -f.myrc -lOlorin_archer
```

The wrapper always passes `-c` (force GCU) to the client. Any args you
provide are appended after that and after the four `--term=...` args.

## Behaviour

- A new tmux session named `tomenet-<pid>` is created. You're attached
  automatically.
- Side panes (term-1/2/3) hold a `sleep 999999999` placeholder so their
  pty stays open and a shell prompt doesn't repaint over the game.
- **Keyboard input only registers in the main pane.** Keystrokes typed
  into a side pane go to its `sleep` and are dropped.
- Detach with `Ctrl-b d`. Reattach with `tmux attach -t tomenet-<pid>`
  (use `tmux ls` to find the pid).
- When the game exits cleanly, the wrapper kills the tmux session and the
  side-pane sleeps die with it.

## Limitations

- **Resize:** if you resize a side pane (e.g. drag a tmux border), the
  game will not see SIGWINCH on that pane and won't redraw. Resize
  handling is on the roadmap; for now, pick a layout and keep it.
- **Sound:** SDL audio still works (it doesn't depend on the terminal).
- **Side-pane input:** see above. Input multiplexing across panes is not
  implemented. The main pane is the keyboard.

## Customising the layout

Edit the `tomenet-tmux` script directly. The pane-creation block is
explicit `tmux split-window` calls; change percentages or add more panes
(up to 10 - `MAX_TERM_DATA_GCU` in `src/common/defines.h`). For each new
pane, capture its tty and add a matching `--term=IDX:$ttyN` to `GAME_CMD`.

## How it works

1. `tmux new-session -d` creates a detached session.
2. `tmux split-window` builds the pane geometry.
3. `tmux display-message -p '#{pane_tty}'` reads each pane's pty path.
4. `tmux send-keys` parks `sleep 999999999` in the side panes so they
   hold their ptys open without a shell prompt.
5. `tmux send-keys` runs `tomenet -c --term=0:tty0 --term=1:tty1 ...` in
   the main pane, followed by `tmux kill-session` so cleanup happens
   when the game exits.
6. `exec tmux attach` hands the terminal off to tmux.

When the client sees `--term=` args, `init_gcu()` takes the multi-SCREEN
path (`init_gcu_multiterm()` in `src/client/main-gcu.c`): one ncurses
`SCREEN` per tty via `newterm()`, with `set_term()` switching between
them as the term framework drives the hooks.
