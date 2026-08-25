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
system already has — `cp`, `mv`, `rm`, `mkdir` and `readlink`. Shale installs nothing,
including itself. `git` is used only to clone a layer repository the machine does not have yet, so
`doctor` reports it missing as a note where no configured layer needs cloning and as a problem where
one does.
`chkstow` ships with GNU stow and is what `doctor` audits `$HOME` with; without it `doctor` says so
and skips that one check. `shale doctor` names every one of these it cannot find, so it is the
fastest answer to a machine where something will not run.

Shale is written to a floor of bash 3.2 and POSIX utility flags, which is what macOS ships. The test
suite runs in CI on Linux and on macOS, the macOS job under `/bin/bash` so that the floor is
exercised rather than assumed. WSL is Linux for shale's purposes. No other platform is exercised.

## Install

Clone this repository, and copy the script from the clone to any directory on your `PATH`. These
commands and the bootstrap below run from the clone's root, which is where `shale` and the example
configs are:

```sh
mkdir -p ~/.local/bin
cp shale ~/.local/bin/shale
command -v shale
```

`~/.local/bin` is the usual choice, and it is often not on `PATH` on the machine you are
bootstrapping, because putting it there is a thing your dotfiles do and they are not applied yet.
That is what the third line is for: nothing printed means this shell cannot find shale, and
`export PATH="$HOME/.local/bin:$PATH"` is enough to get through the bootstrap below. Applying your
layers is what makes it permanent.

Shale never needs to know where it lives, and there is nothing else to install. There is no version
command: shale is one file, and the copy you have is whatever the repository held when you copied it.

## Bootstrap on a new machine

```sh
mkdir -p ~/.dotfiles && chmod 0700 ~/.dotfiles
cp examples/wsl-work.conf ~/.dotfiles/shale.conf   # examples/ is in this repository
$EDITOR ~/.dotfiles/shale.conf                     # list this machine's layers
mkdir -p ~/.dotfiles/local                         # a layer with no url is yours to create
```

Declare any mode that matters before that first apply. Git records no permission bits for a
directory and only the executable bit for a file, so an `.ssh`, a credentials file or anything else
private is composed and linked at the cloning machine's umask — typically `755` and `644` — with
every command exiting 0, `doctor` included: a mode nobody declared is not a fault shale can see.
Declarations go in a `.shale-modes` at the top of the clone root, which for a url-less layer is the
layer directory itself, and which a layer from a repository carries committed with it:

```
# ~/.dotfiles/local/.shale-modes
700  .ssh
600  .ssh/config
```

```sh
shale apply                        # clones what is missing, builds, links
```

[Declaring the mode of a path](#declaring-the-mode-of-a-path) is the whole of the format.

Shale clones a layer that has a url and creates no layer that has not, so every url-less layer has
to exist before the first apply — `local`, in both shipped examples. `examples/minimal.conf` is the
smaller starting point: one repository layer and that same local directory. A repository layer's
clone root does not exist until that apply creates it, so a mode not already committed with the
layer cannot be declared ahead of it: apply once, put the `.shale-modes` in the clone now on disk,
commit it, and apply again. The second apply narrows what you declared and widens nothing, so the
first apply's open mode does not stay open.

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
root and leave it off the other lines that share it. Blank lines and `#` comments are ignored. The
three fields are separated by whitespace and there is no quoting, so none of them can contain a
space; [docs/layers.md](docs/layers.md) says what shale can and cannot tell you when one does. A
`name`, and each component of a `path`, is ASCII: it starts with an ASCII letter, a digit or `_`,
and may go on with those, `.` and `-`. A directory with an accent in its name cannot be a layer —
rename it. A `url` is not restricted.

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

`~/.dotfiles` is the default and not the only choice: set `SHALE_DIR` to an absolute path and that
directory holds `shale.conf`, the layers and the built tree instead. Every path `shale` prints names
the directory in force, the usage text above included. Trailing slashes are ignored; an unset or
empty `SHALE_DIR` is `~/.dotfiles` exactly as before; and a message naming the variable refuses a
relative path, a path starting with a literal `~`, `/` itself, and `$HOME` or any directory above it
— shale composes into that directory and stows the result into `$HOME`, and stow will not link a
directory into itself. What `apply` links into is `$HOME` whatever the root is: the variable moves
where the material is kept, not where it lands. Every command reads it, so export it from the
environment rather than setting it on one command line — a build under one root and an apply under
another leave `$HOME` linked into a tree nothing rebuilds.

## Blocking a file from every apply

`.shale-ignore`, at the top of a clone root, lists the files no apply should link:

```
~/.dotfiles/
  shale.conf          this machine's
  personal/           the clone root
    .shale-ignore     committed, and shared by every machine that clones it
    base/  wsl/
```

```
# ~/.dotfiles/personal/.shale-ignore
.DS_Store
*.swp
Thumbs.db
```

Junk like that lands in a layer through ordinary use, and the repository's `.gitignore` does not stop
it: shale composes from the filesystem, not from git's index, so a `.DS_Store` git never sees is
copied and linked on every machine that clones the layer. Every build reads one such file per clone
root, concatenates them, and writes them into `current/.stow-local-ignore` below stow's own list — so
they are in force on the *first* build, before anything has been applied, which a `~/.stowrc` a layer
ships cannot be. Shale reads these files and copies none of them; one beside the config or inside a
layer is read by nothing, and `shale doctor` names it.

The patterns are globs, in the familiar subset of a `.gitignore`'s: `*` and `?` match inside one path
component, `[abc]` and `[!abc]` match one character, and everything else is literal. A pattern with
no `/` matches a path component at any depth; one with a `/` anywhere is anchored at the top of the
tree. A directory that matches takes everything under it. Anything shale cannot translate — a
backslash, a `**`, a trailing `/`, an unclosed `[` — is refused by name, with the file and line, and
stops the build rather than reaching stow. Nothing about what shale does changes with the shell you
run it from: a shell that exports `SHELLOPTS`, `BASH_ENV`, or `BASHOPTS` on bash 4.1 and later
passes its own options to every script it starts, and shale turns off the ones that would change an
answer — the `extglob` and `nocasematch` that rewrite a pattern, and the `keyword` and `errexit`
that rewrite what the script itself means — before it does anything. `set -v` and `set -x` are left
alone, so `bash -x shale build` still works.

The file is still composed into `current/`; the pattern changes only what stow links. So a pattern
that matches more than it was meant to leaves a real file unlinked with `shale apply` exiting 0, and
two commands report that: `shale doctor` names the patterns in force, with the file and line each
came from, where the built tree holds a file they cover, and `shale which PATH` says when a path is
on the list and which pattern put it there. Adding a pattern does not unlink what an earlier apply
already linked — stow skips an ignored path rather than unstowing it — so `doctor` names those
leftover links too, and removing them is the whole of the fix.
[docs/layers.md](docs/layers.md#blocking-a-file-without-removing-it) has the detail.

## Declaring the mode of a path

Git records no permission bits for a directory and only the executable bit for a file, so a cloned
layer has whatever the clone's umask gave it — `755` on most machines, `775` on Debian and Ubuntu,
and `644` for a `config` you committed at `600`. A mode you want is a mode you declare, in a
`.shale-modes` beside the `.shale-ignore` at the top of the clone root:

```
# ~/.dotfiles/personal/.shale-modes
700  .ssh
600  .ssh/config
700  .gnupg
```

The mode first, in `chmod` order, then one exact path relative to the layer root — no globs, and
three octal digits, setuid and setgid and sticky being refused rather than set on a path in your
home directory. `build` puts that mode on the path in `current/`, and `apply` puts it on the
directory stow makes in `$HOME`, which is the only way one reaches `$HOME` at all: stow creates each
directory with `mkdir` and the caller's umask. A directory nobody declares is `755` in `current/`
and stow's business in `$HOME`. A path no layer provides, a path with a symlink anywhere on it, and
any line shale cannot read all stop the build with the file and the line. Two clone roots declaring
one path resolve by config order, and `shale which` names the line that wins.
[docs/layers.md](docs/layers.md) has the whole of it.

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

SHALE_DIR, an absolute path, names that directory; ~/.dotfiles without it.

A .shale-ignore file at the top of a clone root blocks paths from every apply.
A .shale-modes file there declares modes, one "700  .ssh" per line.
```

`shale help`, `shale -h` and `shale --help` print that same text, as does `shale` with no arguments
above. The four commands in it are the whole surface: there are no others, and no options.

| Command | What it does |
|---|---|
| `shale build` | Clones any missing clone root that has a url, then composes the layers into `~/.dotfiles/current` |
| `shale apply` | Builds, then stows `current` over `$HOME`, replacing the last apply's links |
| `shale doctor` | Checks the prerequisites, the config, the layers, the built tree and `$HOME`; it will not exit 0 on a setup `build` or `apply` is certain to refuse |
| `shale which PATH` | Names every layer providing `PATH`, winner first, and when `build` would refuse the set |

After editing a file a layer already has, `build` is the whole loop: `~/.zshrc` is a link into
`current`, so rebuilding the tree it points at makes the edit live at that instant. `apply` is for
when a *path* appears or disappears — a file or directory added to a layer has nothing in `$HOME`
pointing at it until stow makes the link, and one deleted from every layer leaves a link behind
until stow prunes it. Apply when unsure: it builds first, so it is never less than a build.
[docs/layers.md](docs/layers.md) has the boundary case by case.

Builds are quiet. Three things make one say anything beyond the `composed layer` line it prints per
layer, and nothing else does. A build that takes a path out of the tree and leaves a link in `$HOME`
pointing at nothing counts those links and reports them, which is *Links left behind after layers
change* in [docs/migrating.md](docs/migrating.md). A build that finds `current` holding a copy
*newer* than the layer file replacing it reports that per file, naming where it kept what it
replaced. And a build about to remove a `~/.dotfiles/current.old` reports that removal where the
tree holds something the layers cannot produce again — a file newer than its layer copy, one whose
kind the layers have changed, or one at a path no layer provides — where anything that is not a tree
shale composed is sitting at the name, or where the state is one an interrupted build leaves. A
build that finds `current` holding a copy *older* than the layer file replacing it keeps the
previous tree and says nothing, because an edited layer leaves exactly that state and a message
there would be a message on every edit; `shale doctor` is what reports that mismatch, as a note
rather than a problem, and only until the build that overwrites it. `doctor` names a `current.old`
on the same conditions the removal is announced on, and also where a problem it found means no build
is coming — see [docs/layers.md](docs/layers.md).

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

Two shale runs at once are not coordinated. Shale takes no lock, so two `build` or `apply` runs
against the same `~/.dotfiles` interleave their renames and the tree that results is undefined. Run
one at a time.

`~/.dotfiles/current` is generated output. It is disposable — a bad build is fixed by fixing a layer
and building again — and it should never be edited or versioned. Edit the layer, not the result. A
build that finds a regular file in `current` differing in age from the layer's copy keeps the tree it
replaced at `~/.dotfiles/current.old`, so what was there is recoverable until the next build, which
removes it — and says so as it goes, where that tree holds anything the layers cannot produce again,
or is not a tree shale composed at all. Newer means either that something wrote or moved a file into the built tree — which is
what `stow --adopt` does with a file newer than the layer's copy — or that a different layer now
provides the path with an older copy; the build says so per file, stating both. Older means
either that something moved a file into it, which is what `stow --adopt` does, or that a layer has
moved on and there has been no build since; the build is silent on that direction, and `shale doctor`
names the files instead, up to ten of them with a count above that. A symlink is never itself
compared: a build copies the link rather than what it points at, so an age comparison there would be
about a file the build never touches. A
build composes into `~/.dotfiles/current.new` and renames that into place, so a `current.new` left
lying about is a run that died mid-build and is safe to delete.

One file in there is shale's own rather than a copy: `current/.stow-local-ignore`, which is GNU Stow
2.4.1's default ignore list. It keeps a `README`, `LICENSE` or `COPYING` at the top of the built tree
out of `$HOME`, and at any depth a `.gitignore`, a `.gitmodules`, an editor backup like `.zshrc~`, an
emacs autosave or lock file like `#notes#` or `.#lockfile`, and
the furniture of RCS, CVS, Subversion, Darcs and Mercurial. Stow reads a package's own list *instead
of* its built-in one rather than as well, so the file has to carry the whole of it; writing 2.4.1's
is also what makes stow 2.3.1 apply the same tree. That list applies with nothing configured, so a
file it covers reaches `current/` and stops there with every command exiting 0; `shale which` on the
path says so, and says the list is stow's own rather than one of yours — there is no line to edit,
and only a different name gets the path linked. The patterns from each clone root's
`.shale-ignore` are appended below that list, translated into stow's syntax with the glob they came
from beside each one. A layer that ships a `.stow-local-ignore` at its own top level fails the build,
naming the layer; one further down is an ordinary file and is copied like any other.

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

`which` also answers the question that brings most people to it — why a file it names is not in
`$HOME` — for the reasons shale knows: an ignore pattern of yours, stow's own default list, a `.git`
no build composes, a pair of layers `build` would refuse. Where it knows none it says nothing, and
what is left to check is `~/.stowrc` and `~/.dotfiles/.stowrc`: stow reads both, shale reads neither,
and an `--ignore` in one takes a file out of every apply with each command exiting 0. `shale doctor`
names such a file where one exists.

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
configured layer directory is there and that nothing in one is a directory it cannot search or a
file it cannot open, that no two layers disagree about whether a path is a file or a directory, that
no layer ships a symlink whose target is spelled from the root, which stow will not link, that
nothing in `$HOME` at a path the layers provide is something no apply put there — a file, a
directory, or a link of your own, none of which stow will write over — that no link in `$HOME` at
such a path is spelled from the root, which reads as applied and which stow refuses, that no layer
ships `.stow-local-ignore`, which shale writes itself, or shale itself, that `~/.dotfiles/current`
is a directory rather than a file or a link to one, that a build could create the tree it composes
into, that nothing in `~/.dotfiles` whose name does not begin with a dot is a stray, that no
`current.new` has been left beside `current` and no `current.old` holding anything the layers cannot
produce again, and that no link in `$HOME` points into the built tree at a path it no longer
provides. Where a `.stowrc` exists it says so, because stow appends its `--ignore` patterns to the
ones shale passes and one of them can drop a file from an apply without a word. It reads nothing out
of that file, and changes nothing itself.

```
$ shale doctor
shale: no problems found
```

Doctor reports two kinds of line. A *problem* is a defect in the setup: it counts towards the total
and makes doctor exit 1. Everything else is a note about something worth knowing, and neither counts
nor changes the exit code. Every state doctor can see that stops a build or an apply outright is a
problem, so `no problems found` is a strong signal that both will run rather than a guarantee that
they must. What it does not see: whether `$HOME` is writable, which fails an apply rather than a
build, nor whether a leftover `current.old` can be removed.

That verdict is about the checks that ran. The audit of `$HOME` for broken links is the only one that
looks outside `~/.dotfiles`, and it needs a built tree, `chkstow`, `readlink` and a `$HOME` to walk.
Without any one of them doctor says which in a note and closes `no problems found, but /home/you was
not audited for broken links: read the note above` rather than the bare verdict.

Below, a `vim  vim` line
was added to the config for a layer that is not
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

That scan reads the names in `~/.dotfiles` that do not begin with a dot, and only those. The
directory is usually a git clone, so it is full of dot names that are exactly where they belong —
`.git`, `.gitignore`, `.github` — and naming those would bury the strays worth reading under a list
of files there is nothing to do about. What that costs is the dot-named stray: put a `.tmux.conf`,
or a leftover `.old-layer` directory, straight into `~/.dotfiles` rather than into a layer and
doctor says nothing about it, where the same two without the dot are both reported. `shale which
.tmux.conf` will not lead you to it either — it answers about layers, and no layer provides it.
Dotfiles belong inside a layer, at the path they take in `$HOME`.

The dot names doctor does report come from checks of their own rather than from that scan, and each
knows one thing: a `.stowrc` gets the note that stow reads it, and `.shale-ignore` and
`.shale-modes` are looked for by name beside the config and inside every configured layer, so a
misplaced one is reported at either, up to the ten doctor names before it counts the rest. A copy
anywhere else — beside a clone root's layers rather than at the top of it, or under a directory no
config line makes a layer — is in neither place, and is as silent as any other dot name:

```
$ shale doctor
shale: /home/you/.dotfiles/.shale-ignore is read by nothing
shale:   shale reads one .shale-ignore per clone root, at /home/you/.dotfiles/<root>/.shale-ignore
shale:   move it into the clone root whose layers it is about
shale: no problems found, but read the note above
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
first. Run this before you do, while the links are still there — it names every file in `$HOME` the
copy would overwrite, and on a `$HOME` only shale writes to it prints nothing:

```sh
( cd ~/.dotfiles/current && find . ! -type d -exec sh -c 'for p; do p=${p#./}; [ -L "$HOME/$p" ] || [ ! -e "$HOME/$p" ] || printf "%s\n" "$HOME/$p"; done' sh {} + )
```

Then the copy, as one chained block so a step that fails stops the ones after it:

```sh
stow -D -d ~/.dotfiles -t ~ current &&
cp -RL ~/.dotfiles/current/. ~/ &&
rm ~/.stow-local-ignore &&
rm -rf ~/.dotfiles
```

Every link into `current/` goes, the built tree lands as real files, shale's own
`.stow-local-ignore` goes with the rest, and the layers, config and built tree follow once the repos
are pushed. Those notes are here rather than at the ends of the lines because a `#` pasted into zsh
is not a comment, and `&&` would bind to it instead of to the next command.

The step that stops the block is usually the first: one ordinary file at a path the built tree also
provides makes GNU Stow 2.3.1 refuse the unstow, and unchained the last line runs anyway and deletes
`~/.dotfiles/local`, the layer that is nowhere else.

The chain cannot help where nothing fails, and on stow 2.4.1 nothing does: it unstows past the file
2.3.1 refuses, and `cp -RL` then overwrites whatever `$HOME` holds at a path the built tree provides.
That is only ever a file shale never managed — one you edited in place so that it replaced the link,
or one stow was never allowed to link, which is what a repository-root layer's own `README.md` is.
Both are on the list the check above prints, which is the only warning either gets.

What the copy deposits is exactly what `ls -a ~/.dotfiles/current` lists. The order matters, and
[docs/migrating.md](docs/migrating.md) says what `cp` here does and does not preserve, what it
overwrites, and how to carry on from a block that stopped.

## Known limits

- Rebuilding replaces `~/.dotfiles/current` in place, so there is a brief window in which the
  symlinks in `$HOME` point at nothing. Apply relinks immediately afterwards.
- No build is incremental: every one composes every layer from scratch, so a one-character edit
  costs what a first build costs. Roughly two and a half milliseconds a file — 5.0 seconds for a
  2000-file tree spread over 400 directories, 40 milliseconds for a seven-file one, on the container
  these were measured in. A directory
  is one `mkdir` and costs less than a file: no layer's directory mode is read or copied. What costs
  instead is each line of a `.shale-modes` — one `chmod` in the build, and a read and a `chmod` in
  the apply — so the price is the length of that file rather than the shape of the tree. Against a
  400-directory tree, five lines were not measurable, twenty added 13 milliseconds to the build and
  72 to the apply, and four hundred added 0.9 seconds and 2.4.
- When a whole directory leaves every layer, stow does not visit it and the links inside it are left
  dangling by an apply that still exits 0. The run that took those paths out of the tree counts them
  and says so in one line — the `build` where you ran it on its own, the `apply` otherwise, never
  both; `shale doctor` names them and prints what to do about them, and goes on finding them on every
  later run, which neither does — each speaks only for the run you just made.
  [docs/migrating.md](docs/migrating.md) covers the case in full.
- A shale that is killed outright — `kill -9`, or the machine losing power — leaves a
  `~/.dotfiles/current.new` behind if it died mid-build. `doctor` notes it, and any `current.old`
  beside it, saying what each is; the next build removes both, and where doctor found a problem that
  stops that build it says so instead.
- Git cannot store an empty directory, so a layer that needs one ships a `.gitkeep`, which will
  appear in `$HOME`.
- Git records no permission bits for a directory, and only the executable bit for a file, so a
  freshly cloned layer has whatever modes the clone's umask gave it — `755` on most machines, `775`
  on Debian and Ubuntu, and `644` for a file you committed at `600`. Shale carries a file's mode
  across faithfully, and the working tree after a clone is not what was committed; a directory's
  mode it does not carry at all. Where a mode matters,
  declare it in a `.shale-modes` at the top of the clone root: that file is plain text, which is the
  one thing about a mode git reproduces exactly. `doctor` says nothing about a clone root's own mode,
  group-writable or not, and that is deliberate: the next re-clone puts the mode straight back, so
  the note would return every time it was acted on. Narrow them yourself if the machine has other
  people on it.
- The built tree is `700`, so nothing but you can walk it. Nothing shale runs needs to — stow reads
  it as you — but if your `$HOME` is group-readable by design and something else reads a file shale
  linked, it resolves the link into `~/.dotfiles/current` and stops there. There is no option to
  relax it, and a `chmod` does not hold: a build renames a fresh tree over the old one rather than
  chmodding it. `shale doctor` notes a `current` whose permission bits are not `700`. Only those
  bits are shale's — a setgid bit inherited from `~/.dotfiles` gives you `2700` on every build, and
  `doctor` says nothing about that.
- That `700` only holds as far as the directory above it. `~/.dotfiles` is yours — shale never
  creates it and never chmods it — and anyone who can write there can rename `current` aside and
  leave a tree of their own at the name, which every link in `$HOME` then resolves into. So `doctor`
  notes a `~/.dotfiles` that its group or the world can write, and prints the `chmod` that closes it;
  `700` and `750` are silent, and no build or apply clears the note, because only you can.
- Shale sets the mode of a directory it creates in `$HOME`, and never widens one that was already
  there. So a `~/.ssh` you keep at `700` survives a layer that declares `755`. `apply` says in one
  line that it left such a directory alone, `doctor` names each one and both ways to settle it —
  change the declaration, or `chmod` the directory yourself — and `doctor` stays green, since
  nothing is broken and the directory shale kept is the safer of the two.

## Licence

MIT. See `LICENSE`.
