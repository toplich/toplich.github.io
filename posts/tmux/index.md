# tmux Quick Start Guide


# 🧪 Quick Guide to `tmux`

`tmux` (terminal multiplexer) lets you run multiple terminal sessions in one window — detach and reattach them at will. It's ideal for remote work and multitasking in the terminal.

---

## 🚀 Start a New Session

```bash
tmux new -s mysession
```

This creates a new tmux session named `mysession`.

---

## 🔁 Reattach to a Session

List all available sessions:

```bash
tmux ls
```

Attach to the most recent session:

```bash
tmux attach
```

Attach to a specific session:

```bash
tmux attach -t mysession
```

---

## ⌨️ Common Keybindings

> Press `Ctrl + b` first, then:

| Key | Action                  |
| --- | ----------------------- |
| `d` | Detach from session     |
| `c` | Create a new window     |
| `n` | Next window             |
| `"` | Split pane horizontally |
| `%` | Split pane vertically   |
| `x` | Close the current pane  |
| `[` | Enter scrollback mode   |
| `:` | Open command prompt     |

---

## 🧹 Exit a Session

Kill a specific tmux session:

```bash
tmux kill-session -t mysession
```

Or exit from inside `tmux`:

```bash
exit
```

---

## 🔧 Useful Aliases (`.bashrc` or `.zshrc`)

```bash
alias ta="tmux attach"
alias tn="tmux new -s"
alias tls="tmux ls"
```

{{< admonition type=tip title="Pro tip" open=true >}} Tmux is perfect for long-running processes on servers. Detach with `Ctrl + b`, then `d`, log out, and come back later! {{< /admonition >}}


