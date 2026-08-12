# Shale

Layered dotfiles: fine layers compacted into one tree.

Shale composes several directory trees into one and hands the result to GNU Stow. Each tree is a
*layer* mirroring your `$HOME` layout. Layers are stacked in a configured order and copied over each
other; when two layers provide the same path, the higher one wins, whole file. That is the entire
conflict-resolution mechanism — shale does not merge, template, or interpret any file it copies.

Because a layer is only a directory, layers can come from anywhere: a personal repo, a work repo on a
different account, or a directory that exists on one machine and is versioned nowhere. Combining them
is the machine's business, not the repos'.

## Requirements

`bash`, `git` and GNU `stow`, installed with your system package manager, and the coreutils any
system already has — `cp`, `mv`, `rm`, `mkdir`, `rmdir` and `readlink`. Shale installs nothing,
including itself. `git` is used only to clone a layer repository the machine does not have yet.
`chkstow` ships with GNU stow and is what `doctor` audits `$HOME` with; without it `doctor` says so
and skips that one check. `shale doctor` names every one of these it cannot find, so it is the
fastest answer to a machine where something will not run.

Shale is written to a floor of bash 3.2 and POSIX utility flags, which is what macOS ships. The test
suite runs in CI on Linux and on macOS, the macOS job under `/bin/bash` so that the floor is
exercised rather than assumed. WSL is Linux for shale's purposes. No other platform is exercised.

## Install

Copy the script to any directory on your `PATH`:

```sh
mkdir -p ~/.local/bin
cp shale ~/.local/bin/shale
```

Shale never needs to know where it lives, and there is nothing else to install.

## Bootstrap on a new machine

```sh
mkdir -p ~/.dotfiles
cp examples/wsl-work.conf ~/.dotfiles/shale.conf
$EDITOR ~/.dotfiles/shale.conf     # list this machine's layers
mkdir -p ~/.dotfiles/local         # a layer with no url is yours to create
shale apply                        # clones what is missing, builds, links
```

Shale clones a layer that has a url and creates no layer that has not, so every url-less layer has
to exist before the first apply — `local`, in both shipped examples. `examples/minimal.conf` is the
smaller starting point: one repository layer and that same local directory.

Dotfiles already in `$HOME`? Whether they are ordinary files or stow packages you stow by hand, they
have to stop being files at those paths before the first apply, and anything of yours worth keeping
has to be copied into a layer by hand — shale merges nothing.
[docs/migrating.md](docs/migrating.md) takes both starting points step by step.

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
work    work/base         git@github-work:employer/dotfiles.git
local   local
```

Here `personal/base` and `personal/wsl` are two layers from one clone at `~/.dotfiles/personal`,
`work/base` is a layer from a second clone at `~/.dotfiles/work` on a second GitHub account, and
`local` is this machine's own unversioned directory. Reordering the lines changes precedence;
nothing else does.

The config is machine-specific and belongs to the machine, not to any layer repo.

Every transcript below was produced from that config, in a home directory at `/home/you` that had
been applied once already — `doctor` says more before the first build than after one. Where a
transcript needed a fault to show, the text says what was broken first.

## Commands

```
$ shale
shale - layered dotfiles builder

usage:
  shale build          compose the configured layers into ~/.dotfiles/current
  shale apply          build, then stow current into your home directory
  shale doctor         check prerequisites, config, and broken links
  shale which PATH     show which layer provides PATH, and what it shadows

~/.dotfiles/shale.conf lists one layer per line, lowest precedence first:

  NAME  PATH  [URL]

PATH is relative to ~/.dotfiles.  URL says how to clone PATH's top-level
directory; give it once per directory and leave it off the other lines.
```

`shale help`, `shale -h` and `shale --help` print that same text, as does `shale` with no arguments
above. The four commands in it are the whole surface: there are no others, and no options.

| Command | What it does |
|---|---|
| `shale build` | Clones any missing clone root that has a url, then composes the layers into `~/.dotfiles/current` |
| `shale apply` | Builds, then stows `current` over `$HOME`, replacing the last apply's links |
| `shale doctor` | Checks the prerequisites, the config, the layers and `$HOME`, and reports what it finds |
| `shale which PATH` | Names every layer providing `PATH`, winner first, and when `build` would refuse the set |

After editing a file a layer already has, `build` is the whole loop: `~/.zshrc` is a link into
`current`, so rebuilding the tree it points at makes the edit live at that instant. `apply` is for
when a *path* appears or disappears — a file or directory added to a layer has nothing in `$HOME`
pointing at it until stow makes the link, and one deleted from every layer leaves a link behind
until stow prunes it. Apply when unsure: it builds first, so it is never less than a build.
[docs/layers.md](docs/layers.md) has the boundary case by case.

Every build after an edit also reports the file as one the built tree held an older copy of, and
names `stow --adopt` while doing so; that is the built tree having been out of date, not an
accusation, and [docs/layers.md](docs/layers.md) says why shale cannot tell the two apart.

Shale chooses between three exit codes, and no others:

- **0** — shale did what was asked.
- **1** — shale diagnosed a problem: a broken config, a missing layer, a build conflict, a stow
  failure, a `doctor` finding, or a path no layer provides.
- **2** — the invocation was wrong. Usage goes to stderr.

A run a signal ends is the exception, and exits on the signal in the usual way: 130 for a Ctrl-C,
143 for a `kill`, 137 for a `kill -9`.

What a command produces goes to stdout — `doctor`'s report, `which`'s answer, `build`'s progress —
so it can be piped or redirected. Diagnostics and warnings go to stderr. That is the whole rule, and
it splits the two commands that say something alongside their answer: `doctor`'s notes are part of
its report and go to stdout with it, while `which`'s note that `build` would refuse the set is a
diagnostic, so `shale which PATH > file` keeps the answer and leaves the warning on the terminal.

`build` and `apply` take a lock at `~/.dotfiles/.lock` for the whole build and, for `apply`, the
stow that follows it. A second shale run refuses immediately rather than waiting, and touches
nothing:

```
shale: another shale has the lock at /home/you/.dotfiles/.lock
shale:   build and apply take it so that two of them cannot rebuild /home/you/.dotfiles/current at once
shale:   if no other shale is running, this lock is stale: remove it with: rmdir '/home/you/.dotfiles/.lock'
```

`~/.dotfiles/current` is generated output. It is disposable — a bad build is fixed by fixing a layer
and building again — and it should never be edited or versioned. Edit the layer, not the result. A
build that finds a regular file in `current` differing in age from the layer's copy says so and keeps
the tree it replaced at `~/.dotfiles/current.old`, so what was there is recoverable until the next
build, which removes it. Newer means something edited the built tree; older means either that
something moved a file into it, which is what `stow --adopt` does, or that a layer has moved on and
this is the first build since. A symlink is never itself compared: a build copies the link rather
than what it points at, so an age comparison there would be about a file the build never touches. A
build composes into `~/.dotfiles/current.new` and renames that into place, so a `current.new` left
lying about is a run that died mid-build and is safe to delete.

One file in there is shale's own rather than a copy: `current/.stow-local-ignore`, which keeps a
`README`, `LICENSE` or `COPYING` at the top of the built tree out of `$HOME`. A layer that ships a
`.stow-local-ignore` at its own top level fails the build, naming the layer; one further down is an
ordinary file and is copied like any other.

To update your layers, pull each clone root with git, then apply:

```sh
git -C ~/.dotfiles/personal pull
shale apply
```

## Which layer wins

```
$ shale which .zshrc
winner    work   /home/you/.dotfiles/work/base/.zshrc
shadowed  base   /home/you/.dotfiles/personal/base/.zshrc
```

Spell the path however you have it to hand: `.zshrc`, `./.zshrc`, `~/.zshrc`, an absolute path and a
trailing slash all name the same thing, as do the copies of it in the built tree and inside a layer
when those are spelled from `~` or absolutely — `~/.dotfiles/current/.zshrc` answers for `.zshrc`,
while `.dotfiles/current/.zshrc` is just a path no layer provides. A path outside `$HOME`, a path
containing `..`, and `$HOME` or the built tree itself are refused with a diagnosis and exit 1.

Directories merge rather than shadow, and the report says so instead of naming a winner:

```
$ shale which .config/profile.d
merged    work   /home/you/.dotfiles/work/base/.config/profile.d/
merged    wsl    /home/you/.dotfiles/personal/wsl/.config/profile.d/
merged    base   /home/you/.dotfiles/personal/base/.config/profile.d/
```

`which` reads the layers, not the built tree, so it answers the same before the first build as after
one. A path no layer provides is a diagnosis, and exits 1:

```
$ shale which .vimrc
shale: no layer provides '.vimrc'
```

## When two layers disagree about a path

A file replaces a file and a directory merges into a directory, but neither may replace the other.
Here the `local` layer provides `.config/profile.d` as a file where the layers below it provide a
directory; `build` names both layers and rebuilds nothing:

```
$ shale build
shale: composed layer 'base' from /home/you/.dotfiles/personal/base
shale: composed layer 'wsl' from /home/you/.dotfiles/personal/wsl
shale: composed layer 'work' from /home/you/.dotfiles/work/base
shale: composed layer 'local' from /home/you/.dotfiles/local
shale: conflict at .config/profile.d
shale:   layer 'local' provides a file        /home/you/.dotfiles/local/.config/profile.d
shale:   layer 'work'  provides a directory   /home/you/.dotfiles/work/base/.config/profile.d
shale:   a layer cannot replace a directory with a file, or a file with a directory
shale:   remove or rename one of them
shale: 1 conflict; /home/you/.dotfiles/current not rebuilt
```

There is no flag to override that. `shale which` on the path names the same pair before you run the
build.

## Checking the setup

`shale doctor` checks that every tool shale runs is installed, that the config parses, that every
configured layer directory is there, that the build lock is neither held nor uncreatable, that
nothing in `~/.dotfiles` is a stray, that no layer ships shale itself, and that no link in `$HOME`
points into the built tree at a path it no longer provides. Where a `.stowrc` exists it names the
stow options that file sets, because several of them change an apply and three of them make it do
nothing at all. It changes nothing itself.

```
$ shale doctor
shale: no problems found
```

Doctor reports two kinds of line. A *problem* is a defect in the setup: it counts towards the total
and makes doctor exit 1. Everything else is a note about something worth knowing, and neither counts
nor changes the exit code. Below, a `vim  vim` line was added to the config for a layer that is not
there, and a stray `~/.dotfiles/old-tmux` directory was created — the first is the problem, the
second the note:

```
$ shale doctor
shale: layer 'vim' has no directory at /home/you/.dotfiles/vim and no url for 'vim'
shale:   add a url on that line, or create the directory
shale: /home/you/.dotfiles/old-tmux is not a configured layer
shale:   it may be a leftover stow package, a stray clone, or a layer you removed from the config
shale: 1 problem found
```

## Adding to a config file instead of replacing it

Shale only replaces whole files. Anything additive is expressed in the config formats themselves: an
entry-point file in your base layer sources a fragment directory, and any layer can drop a file into
it. Git, SSH and tmux have native include mechanisms and should use those directly.

That convention lives in your layers, not in shale — shale has no knowledge of it and treats a
fragment as an ordinary file. A higher layer therefore *replaces* a fragment by reusing its exact
filename, or *adds* alongside it with a new one.

[docs/layers.md](docs/layers.md) writes the convention out in full, alongside the rest of what goes
into authoring a layer.

## Uninstalling

Removing the links leaves `$HOME` with no dotfiles at all, which is what you want between an apply
you regret and the next one:

```sh
stow -D -d ~/.dotfiles -t ~ current
```

It leaves your layers, your config and the built tree alone; `shale apply` puts the links back. To
leave shale altogether and keep your dotfiles as ordinary files, copy the built tree back over them
first:

```sh
stow -D -d ~/.dotfiles -t ~ current    # every link into current/ goes
cp -RL ~/.dotfiles/current/. ~/        # the built tree lands as real files
rm ~/.stow-local-ignore                # the one file in that tree that is shale's own
rm -rf ~/.dotfiles                     # layers, config and built tree, once the repos are pushed
```

What the copy deposits is exactly what `ls -a ~/.dotfiles/current` lists. The order matters, and
[docs/migrating.md](docs/migrating.md) says what `cp` here does and does not preserve.

## Known limits

- Rebuilding replaces `~/.dotfiles/current` in place, so there is a brief window in which the
  symlinks in `$HOME` point at nothing. Apply relinks immediately afterwards.
- No build is incremental: every one composes every layer from scratch, so a one-character edit
  costs what a first build costs. Roughly two milliseconds a file — 4.1 seconds for a 2000-file
  tree, 31 milliseconds for a seven-file one, on the container these were measured in.
- When a whole directory leaves every layer, stow does not visit it and the links inside it are left
  dangling by an apply that still exits 0. `shale doctor` finds them and prints what to do about
  them; [docs/migrating.md](docs/migrating.md) covers the case in full.
- A shale that is killed outright — `kill -9`, or the machine losing power — leaves its lock
  directory behind, and a `~/.dotfiles/current.new` as well if it died mid-build. Shale never
  reclaims a lock, because nothing it can read tells a live holder from a dead one. `doctor` reports
  the lock and names the command that clears it; it says nothing about `current.new`, which the next
  build removes once the lock is gone.
- Git cannot store an empty directory, so a layer that needs one ships a `.gitkeep`, which will
  appear in `$HOME`.

## Licence

MIT. See `LICENSE`.
