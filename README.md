# Shale

Layered dotfiles: fine layers compacted into one tree.

Shale composes several directory trees into one and hands the result to GNU Stow. Each tree is a
*layer* mirroring your `$HOME` layout. Layers are stacked in a configured order and copied over each
other; when two layers provide the same path, the higher one wins, whole file. That is the entire
conflict-resolution mechanism — shale does not merge, template, or interpret any file it copies.

Because a layer is only a directory, layers can come from anywhere: a personal repo, a work repo on a
different account, or a directory that exists on one machine and is versioned nowhere. Combining them
is the machine's business, not the repos'.

## Status

Not yet implemented. Work is tracked in the issues.

## Requirements

`bash`, `git` and GNU `stow`, installed with your system package manager. Shale installs nothing,
including itself.

Tested on Linux and WSL. It is written to macOS's system bash 3.2 and POSIX utility flags, but is
untested there.

## Install

Copy the script to any directory on your `PATH`:

```sh
cp shale ~/.local/bin/shale
```

Shale never needs to know where it lives, and there is nothing else to install.

## Bootstrap on a new machine

```sh
mkdir -p ~/.dotfiles
cp examples/wsl-work.conf ~/.dotfiles/shale.conf
$EDITOR ~/.dotfiles/shale.conf     # list this machine's layers
shale apply                        # clones what is missing, builds, links
```

`examples/minimal.conf` is the smaller starting point — one repository layer and one local directory.

## Configuration

`~/.dotfiles/shale.conf` lists one layer per line, lowest precedence first:

```
name    path              url
```

`path` is relative to `~/.dotfiles`. `url` says how to clone the *first component* of `path` — the
clone root — because several layers commonly come from one repository. Give the url once per clone
root and leave it off the other lines that share it. Blank lines and `#` comments are ignored.

```
base    personal/base     git@github.com:you/dotfiles.git
wsl     personal/wsl
work    work              git@github-work:employer/dotfiles.git
local   local
```

Here `personal/base` and `personal/wsl` are two layers from one clone at `~/.dotfiles/personal`, and
`local` is this machine's own unversioned directory. Reordering the lines changes precedence;
nothing else does.

The config is machine-specific and belongs to the machine, not to any layer repo.

## Commands

| Command | What it does |
|---|---|
| `shale build` | Clones any missing layer, composes the layers into `~/.dotfiles/current` |
| `shale apply` | Builds, then stows `current` into `$HOME` |
| `shale doctor` | Checks prerequisites, the config, the layers, and `$HOME` for broken links |
| `shale which PATH` | Reports which layer provides `PATH`, and which layers it shadows |

Run `shale` with no arguments for usage.

`~/.dotfiles/current` is generated output. It is disposable — a bad build is fixed by fixing a layer
and building again — and it should never be edited or versioned. Edit the layer, not the result.

To update your layers, pull each clone root with git, then apply:

```sh
git -C ~/.dotfiles/personal pull
shale apply
```

## Adding to a config file instead of replacing it

Shale only replaces whole files. Anything additive is expressed in the config formats themselves: an
entry-point file in your base layer sources a fragment directory, and any layer can drop a file into
it. Git, SSH and tmux have native include mechanisms and should use those directly.

That convention lives in your layers, not in shale — shale has no knowledge of it and treats a
fragment as an ordinary file. A higher layer therefore *replaces* a fragment by reusing its exact
filename, or *adds* alongside it with a new one.

## Uninstalling

```sh
stow -D -d ~/.dotfiles -t ~ current
```

This is also the escape hatch if an apply goes wrong.

## Known limits

- Rebuilding replaces `~/.dotfiles/current` in place, so there is a brief window in which the
  symlinks in `$HOME` point at nothing. Apply relinks immediately afterwards.
- Two concurrent `shale apply` runs will interleave and produce an arbitrary tree. Don't.
- Git cannot store an empty directory, so a layer that needs one ships a `.gitkeep`, which will
  appear in `$HOME`.

## Licence

MIT. See `LICENSE`.
