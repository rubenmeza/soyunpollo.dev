---
title: "Godot with Neovim as External Editor"
date: 2026-04-27T12:00:00-06:00
tags: ["godot", "neovim", "gdscript", "lsp"]
categories: ["tools"]
ShowToc: false
draft: false
---

I'm starting my journey into game development. I love games, and for a long time I wanted to learn how to build them. So here I am, opening Godot for the first time as a real tool, not just to play around.

But there is a problem. I am very used to Neovim. I have my own config, my own keybindings, and the vim motions are part of how I think when I write code. Working without them feels slow and uncomfortable. I did not want to give that up just because I am writing GDScript now.

The good news: I don't have to. Godot can use Neovim as the external editor, and Godot's language server still gives me autocomplete, go-to-definition, and diagnostics inside Neovim. Here is how I set it up.

## Tell Godot to use Neovim

Open Godot and go to **Editor → Editor Settings → Text Editor → External**. Check **Use External Editor**, then fill in these two fields:

- **Exec Path**: the full path to your terminal. For Alacritty, that is `/usr/bin/alacritty`.
- **Exec Flags**: `-e nvim {file} +{line}`

The `-e` flag is important. Without it, Alacritty does not know that the rest of the line is the command to run. I lost some minutes on this one.

Now click any `.gd` script in Godot. A new terminal opens with Neovim on the right file and the right line.

## Add GDScript to your LSP config

Godot ships its own language server. It listens on TCP port `6005` while the editor is open. You do not install anything with Mason — Godot is the server.

In your `nvim-lspconfig` setup, register the server:

```lua
vim.lsp.config['gdscript'] = {
  capabilities = capabilities,
  cmd = vim.lsp.rpc.connect('127.0.0.1', 6005),
  filetypes = { 'gd', 'gdscript', 'gdscript3' },
  root_markers = { 'project.godot', '.git' },
}

vim.lsp.enable('gdscript')
```

If `.gd` files are not detected, add this once at startup:

```lua
vim.filetype.add({ extension = { gd = 'gdscript' } })
```

## Try it

Keep Godot open. Open a `.gd` file in Neovim and run `:LspInfo`. You should see `gdscript` attached. Now `K` shows documentation, `gd` jumps to definitions, and completion works through your normal `nvim-cmp` setup.

One small thing to remember: the LSP only runs while Godot is open. Close the editor and the server stops. That is fine for me — I always have Godot open when I write game code anyway.

## Reuse the same Neovim instance

After using this for a while, one thing started to annoy me. Every time I clicked a `.gd` script in Godot, a new terminal opened with a new Neovim. Five scripts, five terminals. Not what I wanted.

The fix: have one Neovim listen on a socket, and send new files to it instead of starting a fresh one.

First, tell Neovim to start a server when it opens. I added this to my config:

```lua
local godot_socket = "/tmp/godot.pipe"
if vim.uv.fs_stat(godot_socket) == nil then
  pcall(vim.fn.serverstart, godot_socket)
end
```

Then a small wrapper script at `~/.local/bin/godot-nvim`:

```bash
#!/usr/bin/env bash
set -e

SOCK=/tmp/godot.pipe
file="$1"
line="${2:-1}"
col="${3:-1}"

if [ -S "$SOCK" ]; then
  nvim --server "$SOCK" --remote-send "<C-\\><C-n>:edit ${file}<CR>:call cursor(${line}, ${col})<CR>"
else
  rm -f "$SOCK"
  alacritty -e nvim --listen "$SOCK" "${file}" "+call cursor(${line}, ${col})" &
fi
```

Make it executable:

```bash
chmod +x ~/.local/bin/godot-nvim
```

Now update Godot's external editor settings:

- **Exec Path**: `/home/<you>/.local/bin/godot-nvim`
- **Exec Flags**: `"{file}" "{line}" "{col}"`

The first click opens a terminal with Neovim, and the socket starts. Every click after that jumps to the file inside the same Neovim. No more terminal spam.

If Neovim is closed but the socket file is still there, the wrapper removes it and opens a fresh terminal. Stale socket case handled.

Small setup, big win.
