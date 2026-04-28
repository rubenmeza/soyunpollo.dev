---
title: "Godot with Neovim as External Editor"
date: 2026-04-27T12:00:00-06:00
tags: ["godot", "neovim", "gdscript", "lsp"]
categories: ["tools"]
ShowToc: false
draft: false
---

I like Godot, and I like Neovim. So I wanted to use both at the same time — open my GDScript files in Neovim, but keep all the help that Godot's language server gives me, like autocomplete and go-to-definition.

The good news: it works, and the setup is small.

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

Small setup, big win.
