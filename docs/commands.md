# Commands

The usage text in the [README](../README.md#commands) names the whole surface. This is the detail
behind it.

Every transcript below was produced from this config, in a home directory at `/home/you` that had
been applied once already — `doctor` says more before the first build than after one. Where a
transcript needed a fault to show, the text says what was broken first.

```
base    personal/base     git@github.com:you/dotfiles.git
wsl     personal/wsl
work    work/base         git@github-work:employer/dotfiles.git
local   local
```

## What a build says

Builds are quiet. Three things make one say anything beyond the `composed layer` line it prints per
layer, and nothing else does. A build that takes a path out of the tree and leaves a link in `$HOME`
pointing at nothing counts those links and reports them, which is *Links left behind after layers
change* in [migrating.md](migrating.md). A build that finds `current` holding a copy
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
is coming — see [layers.md](layers.md).

## Exit codes

Shale chooses between three exit codes, and no others:

- **0** — shale did what was asked.
- **1** — shale diagnosed a problem: a broken config, a missing layer, a build conflict, a stow
  failure, a `doctor` finding, or a path no layer provides.
- **2** — the invocation was wrong. Usage goes to stderr.

A run a signal ends is the exception, and exits on the signal in the usual way: 130 for a Ctrl-C,
143 for a `kill`, 137 for a `kill -9`.

## stdout and stderr

What a command produces goes to stdout — `doctor`'s report, `which`'s answer, `build`'s progress —
so it can be piped or redirected. Diagnostics and warnings go to stderr. That is the whole rule, and
it splits the two commands that say something alongside their answer: `doctor`'s notes are part of
its report and go to stdout with it, while `which`'s note that `build` would refuse the set is a
diagnostic, so `shale which PATH > file` keeps the answer and leaves the warning on the terminal.

## One run at a time

Two shale runs at once are not coordinated. Shale takes no lock, so two `build` or `apply` runs
against the same `~/.dotfiles` interleave their renames and the tree that results is undefined. Run
one at a time.

## The built tree

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

## Moving aside what stow would refuse

`apply --replace` moves the paths `doctor` reports as *holding something no apply put there* into
`~/.dotfiles/replaced/<timestamp>/`, keeping the tree each one had, and then applies. Here
`~/.bashrc` was the ordinary file `useradd` copies in from `/etc/skel`, at a path a layer provides:

```
$ shale apply --replace
shale: moved /home/you/.bashrc to /home/you/.dotfiles/replaced/20260825-104233/.bashrc
```

That report is the flag's exact scope, and it is narrower than "everything stow can refuse": a
symlink of your own whose target is spelled from the root looks applied, aborts the apply just the
same, and is a separate `doctor` finding with its own remedy. `--replace` leaves that one for you.

It moves and never deletes or reads: a file keeps its content and its mode, a symlink moves as a
link, and a directory moves whole. Nothing removes that directory afterwards — `doctor` notes it on
every run until you delete it, which is the point at which you have decided you no longer need what
is in it.

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

The tools are `bash`, `git`, GNU `stow` and `chkstow`, and the coreutils `chmod`, `cp`, `date`,
`ls`, `mkdir`, `mv`, `readlink`, `rm` and `rmdir`. Of those, `git` is used only to clone a layer
repository the machine does not have yet, so `doctor` reports it missing as a note where no
configured layer needs cloning and as a problem where one does. `chkstow` ships with GNU stow and is
what `doctor` audits `$HOME` with; without it `doctor` says so and skips that one check.

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
