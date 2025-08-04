# Vim Quick Start Guide


# ✍️ Quick Guide to `vim`

`vim` is a powerful, keyboard-driven text editor found on almost every Unix-like system. It's fast, lightweight, and essential for sysadmins and developers.

---

## 🧭 Understanding Vim Modes

Vim has **3 primary modes**:

| Mode       | Description             |
| ---------- | ----------------------- |
| **Normal** | Navigation and commands |
| **Insert** | Typing/editing text     |
| **Visual** | Selecting text          |

You start in **Normal mode** by default.

---

## 📂 Opening a File

```bash
vim myfile.txt
```

This opens the file in **Normal mode**.

---

## ✍️ Entering Insert Mode

| Key | Description          |
| --- | -------------------- |
| `i` | Insert before cursor |
| `a` | Append after cursor  |
| `o` | Open new line below  |

💡 Tip: You’ll see `-- INSERT --` at the bottom of the screen when in Insert mode.

---

## ⎋ Returning to Normal Mode

- Press `Esc` to exit Insert or Visual mode and return to Normal mode.

---

## 💾 Saving and Quitting

| Command      | Description         |
| ------------ | ------------------- |
| `:w`         | Save (write)        |
| `:q`         | Quit                |
| `:wq` / `ZZ` | Save and quit       |
| `:q!`        | Quit without saving |

{{< admonition type=tip title="Tip for Beginners" open=true >}} To exit Vim quickly, press `Esc`, then type `:q!` and hit `Enter`. {{< /admonition >}}

---

## 🔁 Navigation Shortcuts (Normal Mode)

| Key                | Action                           |
| ------------------ | -------------------------------- |
| `h`, `j`, `k`, `l` | Move left, down, up, right       |
| `gg`               | Go to top of file                |
| `G`                | Go to end of file                |
| `0` / `^` / `$`    | Start / first char / end of line |
| `w`, `b`           | Next / previous word             |
| `Ctrl+d/u`         | Scroll half page down/up         |

---

## 📌 Editing Text (Normal Mode)

| Key            | Action                        |
| -------------- | ----------------------------- |
| `dd`           | Delete (cut) current line     |
| `yy`           | Copy current line             |
| `p`            | Paste after cursor            |
| `u` / `Ctrl+r` | Undo / redo                   |
| `x`            | Delete character under cursor |
| `r<char>`      | Replace character             |

---

## 🔧 Useful Vim Settings (`.vimrc`)

```vim
set number           " show line numbers
set tabstop=4        " 4 spaces per tab
set expandtab        " convert tabs to spaces
set autoindent       " smart indentation
set mouse=a          " enable mouse support
syntax on            " enable syntax highlighting
```

---

## 📚 Resources

- Run `vimtutor` in terminal for an interactive tutorial.
- Official site: [https://www.vim.org](https://www.vim.org)
- Cheatsheet: [https://vim.rtorr.com](https://vim.rtorr.com)


