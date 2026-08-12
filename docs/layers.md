# Authoring a layer

A layer is a directory tree that mirrors your home directory. Everything in it is copied, path for
path, into `~/.dotfiles/current`, and almost everything in `current` is linked into `$HOME`. A file
at `personal/base/.config/nvim/init.lua` becomes `~/.config/nvim/init.lua`. The exception is the
handful of names stow never links — version-control furniture, editor backups and emacs autosaves —
which *Where a layer lives* below sets out; apart from those there is nothing else to learn about
the mapping.

Shale copies files. It never merges them, never templates them and never reads what is inside one.
Fragment directories, numeric prefixes and shared helper files are conventions for *your* layers,
expressed in the config formats themselves and implemented by the files you write. Shale has no
knowledge of any of them and treats every one as an ordinary file.

## Where a layer lives

Give a layer a subdirectory of its repository rather than the repository root:

```
base    personal/base     git@github.com:you/dotfiles.git
```

Shale skips `.git` at every depth, and the built tree carries a `.stow-local-ignore` holding GNU
Stow's own default ignore list, which keeps a top-level `README*`, `LICENSE*` and `COPYING` out of
`$HOME`, and at any depth a `.gitignore`, a `.gitmodules`, an editor backup like `.zshrc~`, an emacs
autosave like `#notes#` or lock file like `.#init.el`, and the furniture of RCS, CVS, Subversion,
Darcs and Mercurial. Nothing else is filtered — `.gitconfig`,
`.gitkeep` and `~/.config/git/ignore` are matched by none of it, and a `README.md` below the top of
the layer becomes a real link. Name the repository root as the layer and `~/.github/`, `~/Makefile`
and `~/CONTRIBUTING.md` join them. The subdirectory is what keeps the repository's own furniture out
of the image.

Several layers can come from one repository — `personal/base` and `personal/wsl` are two
directories in one clone at `~/.dotfiles/personal`, so the url is given once, on the first of them.
A layer needs no repository at all: `~/.dotfiles/local` with no url is this machine's own directory.
Shale clones only what has a url, so create that one yourself — `mkdir -p ~/.dotfiles/local` —
before the first build, which otherwise stops on it. An empty layer directory composes to nothing
and is fine.

## Replacing a file, and adding one

Precedence is per path and whole file. A higher layer *replaces* a file by reusing its exact path,
and *adds* alongside it by choosing a filename no lower layer uses. Directories are not replaced:
where two layers both provide `.config/nvim/`, the directory merges and the contest happens file by
file inside it.

`shale which` answers who wins, before any build and without one:

```
$ shale which .zshrc
winner    local  /home/you/.dotfiles/local/.zshrc
shadowed  base   /home/you/.dotfiles/personal/base/.zshrc
```

Give it the path however you have it to hand: `.zshrc`, `./.zshrc`, `~/.zshrc`, `/home/you/.zshrc`
and a trailing slash all name the same thing, and so do the copies of it in the built tree and in a
layer — `~/.dotfiles/current/.zshrc` and `~/.dotfiles/personal/base/.zshrc` both answer for `.zshrc`,
spelled from `~` or absolutely. It refuses, with a diagnosis and exit 1, a path outside `$HOME`, a
path containing `..`, and `$HOME` or the built tree itself rather than something in it.

Whole file is worth taking literally, because the loss is silent. A four-line `.gitconfig` in `local`
displaces every line of the one below it, and the apply that installs it says nothing, since nothing
is wrong:

```
$ shale which .gitconfig
winner    local  /home/you/.dotfiles/local/.gitconfig
shadowed  base   /home/you/.dotfiles/personal/base/.gitconfig

$ diff -u /home/you/.dotfiles/personal/base/.gitconfig /home/you/.dotfiles/local/.gitconfig
--- /home/you/.dotfiles/personal/base/.gitconfig
+++ /home/you/.dotfiles/local/.gitconfig
@@ -1,7 +1,2 @@
-[alias]
-	st = status
-	lg = log --oneline
-[core]
-	excludesfile = ~/.config/git/ignore
-[include]
-	path = ~/.config/git/conf.d/50-work.conf
+[user]
+	email = me@example.com
```

The include, both aliases and the excludesfile are gone, and only that diff says so. Run it between
the two paths `which` printed before adding a small override, not after. Where you wanted the four
lines *as well*, use the tool's own include mechanism below rather than a second copy of the file.

A layer cannot replace a directory with a file, or a file with a directory. `build` calls that a
conflict, names both layers and the two paths, and rebuilds nothing; `which` reports the same pair
as a note, and `shale doctor` reports it as a problem without your having to guess which path to ask
about. Rename one of them.

```
shale: 'build' would fail at .config/profile.d — it is a directory in layer 'base' and a file in layer 'work'
shale:   /home/you/.dotfiles/base/.config/profile.d
shale:   /home/you/.dotfiles/work/.config/profile.d
shale:   a layer cannot replace a directory with a file, or a file with a directory
shale:   remove or rename one of them
```

More than two layers can disagree at one path, and how many conflicts `build` reports then depends on
the order rather than on the count: three layers holding directory, directory, file give it one,
where file, directory, directory give it two. `doctor` reports the lowest of them and names the two
layers `build` names it against — the nearest provider below, not the first one — and adds a line
saying how many layers provide the path when it is more than two, because removing one of the pair
it named can leave another that disagrees.

## A layer restored under another user

A layer cloned or restored under `sudo` keeps its owner, and `build` stops at the first directory it
cannot search or file it cannot open — `cp` reports the path and shale reports the copy that failed.
`shale doctor` finds these ahead of the build and names up to ten of them, counting the rest.

Only a regular file matters. A build copies a symlink as a symlink and recreates a FIFO without
opening either, so neither a link whose target you cannot read nor an unreadable FIFO stops
anything, and `doctor` says nothing about them.

## Fragment directories

This is the convention, not a feature. The entry point in your base layer does the sourcing, so
every fragment is an ordinary file that any layer can add, and shale is not involved.

`~/.profile` in the base layer, POSIX `sh` only. Whichever login shell the machine hands you reads
this file — `dash`, `ksh`, or `bash` where there is no `~/.bash_profile`, and zsh only through the
`~/.zprofile` below — and the one thing they all understand is POSIX:

```sh
[ -r "$HOME/.config/shell/helpers.sh" ] && . "$HOME/.config/shell/helpers.sh"

for _f in "$HOME"/.config/profile.d/*.sh; do
  [ -r "$_f" ] || continue
  . "$_f"
done
unset _f
```

The `[ -r "$_f" ] || continue` is what handles an empty `profile.d`: with no match the glob arrives
as the literal pattern, which is not readable, and the loop does nothing.

A login zsh does not read `~/.profile` at all: it reads `~/.zprofile`, then `~/.zshrc`. So the base
layer ships a `~/.zprofile` too, and it sources `~/.profile` in sh emulation:

```zsh
[ -r "$HOME/.profile" ] && emulate sh -c '. "$HOME/.profile"'
```

`emulate sh -c` runs that one command under zsh's sh-compatible options and leaves the rest of the
shell alone. It is what makes a POSIX file behave. Native zsh has `NOMATCH` set, so an empty
`profile.d` turns the loop's glob into an error — `no matches found`, return 126 — and everything
after it in `~/.profile` is skipped. Word splitting differs too: zsh does not split unquoted
expansions, and a file written expecting it goes wrong without saying so.

`~/.zshrc` in the base layer, where the whole of zsh is available and the `(N)` glob qualifier does
the same job:

```zsh
[ -r "$HOME/.config/shell/helpers.sh" ] && . "$HOME/.config/shell/helpers.sh"

for _f in "$HOME"/.config/zsh/rc.d/*.zsh(N); do
  source "$_f"
done
unset _f
```

Any layer then drops a file into `.config/profile.d/` or `.config/zsh/rc.d/` and it is picked up.
A higher layer replaces a fragment by reusing its exact filename and adds one by choosing a new
filename, exactly as for any other file, because to shale that is all a fragment is.

## Ordering, and the inversion PATH puts on it

Zero-padded numeric prefixes fix the order the fragments run in: `10-path.sh` before `20-editor.sh`
before `90-corp.sh`. Pad them, because `100-x.sh` sorts before `20-x.sh` otherwise.

A fragment that *prepends* to `PATH` inverts that order into lookup precedence. The fragment that
runs last leaves its directory at the front of `PATH` and so wins lookups. Give base layers low
numbers and higher layers high ones, and the two orders agree: last to run, first to be found.

```sh
# .config/profile.d/10-path.sh, in the base layer
prepend_path "$HOME/.local/bin"

# .config/profile.d/90-corp.sh, in the work layer
prepend_path /opt/corp/bin
```

leaves `PATH` as `/opt/corp/bin:$HOME/.local/bin:…`.

## Shared helper functions

Fragments that need the same function share one file rather than each defining it. It is sourced by
both entry points before their loops, as above, so it has to be POSIX `sh` — `~/.profile` sources it
too.

```sh
# .config/shell/helpers.sh, in the base layer
prepend_path() {
  case ":$PATH:" in
    *":$1:"*) ;;
    *) PATH="$1${PATH:+:$PATH}" ;;
  esac
}
```

## Tools that include natively

Where a tool has its own include mechanism, use it instead of inventing a fragment directory for it.
They do not all behave alike, and what decides the recipe is which value a tool keeps when it reads
one keyword twice. Where the **last** value wins, the include goes at the *bottom* of the base
layer's file, below everything a higher layer might override; where the **first** wins, it goes at
the *top*. At the wrong end a fragment can still add a key the base file never sets, so the recipe
looks like it works and only the overrides are lost, silently. Establish which rule a tool follows
by asking the tool for the value it settled on rather than by reading the files — that is
`git config --get alias.lg` for git and `ssh -G example.com` for ssh. For tmux it is
`tmux show-options -g`, which answers only against a server that is already running: with none it
prints nothing and exits 1, which is not the fragment failing to load. A window option needs `-gw`,
because a value set with `setw -g` is invisible to plain `-g`.

Git keeps the last value, so the `[include]` goes at the bottom. Its `include.path` takes one path
and does not glob: `path = ~/.config/git/conf.d/*.conf` includes nothing at all, silently. The base
layer therefore names each file it expects, and a higher layer supplies it; an include whose file
does not exist is ignored without error, so naming a file no layer ships costs nothing.

```
[alias]
  lg = log --oneline
[include]
  path = ~/.config/git/conf.d/50-work.conf
```

Put those two blocks the other way round and a `50-work.conf` setting `alias.lg` is read first and
beaten by the base layer's line below it. Nothing reports that, and `shale which .gitconfig` cannot
help, because the base layer genuinely does provide the file and that answer is correct:

```
$ git config --get alias.lg
log --oneline
```

SSH is the other way about: it keeps the first value it obtains for each keyword, so `Include` goes
at the *top* of `~/.ssh/config`, above any `Host` block — a `Host *` default written above the
`Include` would otherwise win over everything the included files say. `Include` does glob, and
`Include ~/.ssh/config.d/*.conf` over a directory that does not exist is not an error.

tmux's `source-file` keeps the last value, as git's include does, so it goes at the bottom of
`tmux.conf`, below the `set -g` lines a fragment might replace. It does glob, so one line covers a
directory; the base layer's `tmux.conf` still names what it sources, and the layers supply those
files.

Whichever tool it is, a higher layer must not replace the file that carries the include. Reconciling
`~/.gitconfig` the ordinary way — a whole `.gitconfig` in `local`, per *Replacing a file, and adding
one* above — takes the `[include]` line with the rest of the file, and every fragment it named stops
being read. Nothing reports it, and `which` is the wrong question to ask: the fragment is still in
the built tree, still linked into `$HOME`, and still the winner at its own path. Only the tool
knows, and what it answers with is the overriding file's value rather than the fragment's:

```
$ git config --get user.email
me@example.com

$ shale which .config/git/conf.d/50-work.conf
winner    local  /home/you/.dotfiles/local/.config/git/conf.d/50-work.conf
```

A value you put in a fragment coming back as something else — or as nothing, exit 1 — is how you
notice. Then ask `which` about the *including* file rather than the fragment: a `shadowed` line
under `.gitconfig` says a higher layer replaced it, and that layer's copy has to carry the
`[include]` too. Where both layers have a claim on the file itself, that copy is the one to add it
to.

## Symlinks in a layer

A symlink is copied as a symlink, target text unchanged, and it is a leaf: shale never recurses into
one, so a layer's symlink to a directory collides with a lower layer's real directory rather than
merging into it.

The target is resolved from where the link ends up inside `current/`, not from `$HOME`. A link to a
sibling works — `.exrc -> .vimrc` resolves inside `current/` and reading `~/.exrc` gets the file.
A link written as though it started in `$HOME`, such as `.exrc -> .dotfiles/current/.vimrc`, dangles
after a successful apply that reports nothing. Never point a layer symlink into `current/`.

Keep links relative. Stow refuses an absolute one and abandons the whole apply, and stow 2.3.1
misses one below the top of the tree — an absolute link that appears to work there stops working
the day stow is upgraded:

```
WARNING! stowing current would cause conflicts:
  * source is an absolute symlink .dotfiles/current/.viminfo-link => /home/you/.dotfiles/current/.vimrc
All operations aborted.
```

`shale doctor` does not report this one, and deliberately: the two stow versions disagree about
which absolute links they refuse and where, so any finding shale wrote would be right on one of them
and wrong on the other. Nothing but the apply itself can tell you, which is the reason for the rule
rather than an exception to it.

## A layer, or a guard inside a file

Both are right in different cases, and the deciding question is whether the file should exist at all
on the other platform.

A file that has no business on a platform goes in an OS layer — a `.config/systemd/` tree in the
Linux layer, a `Library/LaunchAgents/` tree in the macOS one. Nothing on the other machine lists
that layer, so nothing on the other machine has the file.

A file that must exist everywhere but differs in one section ships once, in the base layer, with a
runtime guard inside it. A second copy in an OS layer would replace the whole file, and the two
copies would both need maintaining for the rest of the file's life.

```sh
case "$(uname -s)" in
  Darwin) alias ls='ls -G' ;;
  *) alias ls='ls --color=auto' ;;
esac
```

## Empty directories

Git cannot store an empty directory. Ship a `.gitkeep` in it, and expect it in `$HOME`: shale copies
it like any other file and stow links it. `~/.config/zsh/rc.d/.gitkeep` is the price of a fragment
directory that starts out empty.

## What a layer must not ship

Never a `.stow-local-ignore`. Shale writes the one in `current/` itself — it is the only file shale
creates rather than copies — and a layer providing one fails the build, naming the layer.

Never shale itself: applying a layer that ships it would replace the script with a link into
`current/`. `shale doctor` reports that, judged against the path the running shale was invoked by.
Run the installed copy — `~/.local/bin/shale doctor` — and the finding names the layer. Run the
layer's own copy directly and there is no such finding, because no layer shadows *that* path.

Watch for `~/.stowrc` and `~/.dotfiles/.stowrc`. Their `--ignore` patterns are *appended* to the
ones stow is already using rather than overriding them, so a stray `--ignore=zshrc` leaves
`~/.zshrc` silently unlinked with `shale apply` exiting 0. `shale doctor` notes a resource file if
one exists, and names the stow options it sets.

## Never edit `current/`

`~/.dotfiles/current` is generated output and is disposable — a bad build is fixed by fixing a layer
and building again. Editing it edits the only copy of that change, and `~/.zshrc` being a link into
it makes that easy to do by accident. A build that finds a file in the built tree newer than the
layer it came from warns and keeps the tree it displaced:

```
shale: /home/you/.dotfiles/current/.zshrc was newer than /home/you/.dotfiles/personal/base/.zshrc
shale:   either something wrote or moved a file into the built tree, or a different layer now provides this path and its copy is older
shale:   the layer copy wins; what was there is kept at /home/you/.dotfiles/current.old/.zshrc until the next build, to copy into a layer if it was your edit
```

Read the kept copy before you act on it, because the message covers two things and shale cannot tell
them apart. If it is your own file — an edit through the `~/.zshrc` link, or something `stow --adopt`
moved in — copy it into a layer, and do it before the next build, which reaps `current.old`. If it is
the copy the layer that used to win this path was providing, there is nothing to do: remove an
override, drop a layer from `shale.conf`, or reorder precedence, and the copy that wins the path in
its place is often the older file, which is the same comparison with nobody having touched
`current/`. Two clone roots pulled minutes apart is enough for that. Copying `current.old` back into
a layer there would put back the override you just stopped using.

Never version `current/`, and never point a layer symlink into it.

A symlink is never itself compared, in either direction. What a build copies for one is the link, not
the file at the end of it, so a timestamp comparison there would be about two files the build never
touches — and for a link whose target a higher layer shadows, the two sides resolve to two different
files and every build for ever reports a file nobody edited. So a hand-edited symlink in `current/`
goes unreported.

A file that arrives in the built tree from somewhere else keeps its own timestamp rather than
gaining a new one, so it can be *older* than the layer copy that replaces it — which is what
`stow --adopt` does, and it is the shape a lost file usually has. A build says nothing about that
direction, because an edited layer leaves every path it provides in exactly the same state and the
message would be on the loop you run all day. What the build does is keep `current.old`, and
`shale doctor` is where the files are named:

```
shale: 2 files in /home/you/.dotfiles/current do not match the layer copies that provide them
shale:   either something moved them into the built tree, which is what stow --adopt does, or a layer changed and there has been no build since
shale:   the next build replaces them with the layer copies, keeping what is there now at /home/you/.dotfiles/current.old
shale:   the build after that removes /home/you/.dotfiles/current.old, so a file moved into the built tree is recoverable for one build and no longer
shale:   /home/you/.dotfiles/current/.zshrc
shale:   /home/you/.dotfiles/current/.vimrc
```

Shale cannot tell an adopted file from a built tree its layers have moved past, and does not guess:
the note states both readings and it is a note, not a problem — `doctor` still exits 0 on it. Above
ten files it leads with the count and the directory they are under and names ten of them, so a
`git pull` that changed two hundred is a dozen lines rather than two hundred.

Where `doctor` has already found something that stops a build — a conflict, a layer it cannot read,
a `current` that is not a directory — the two lines about the next build are replaced by
`no build will run until the problems above are fixed, so nothing here is replaced yet`. The files
are still named; what changes is that nothing promises you a `current.old` that a refused build
never creates.

`doctor` names the files only in the window between one arriving and the next build. Afterwards the
built copy is the layer's again and the mismatch is gone; what `doctor` says then is that
`current.old` is there and that the next build removes it, so recovery is `current.old` and one
build deep. If you use `stow --adopt` against `current/`, run `shale doctor` before you build.

## The everyday loop: build, and when to apply

Edit a file a layer already has, and `shale build` is the whole loop. `~/.zshrc` is a link into
`current/`, so rebuilding the tree it points at makes the edit live at that instant — there is
nothing for stow to do, because the link is already right.

That build says nothing about the file you edited, beyond the `composed layer` line it prints for
each layer. It keeps `current.old` — the built copy your edit replaced was older than
the layer copy, which is the state a file moved into the tree also leaves, and shale keeps the
previous tree either way — but it prints no message about it, because a message there is a message
on every edit. `shale doctor` is where a built tree its layers have moved past is reported, and
after a build the mismatch is gone; what doctor names then is `current.old` itself — what it holds,
and that the next build removes it.

`shale apply` is what you need when a **path** appears or disappears, since making and unmaking links
is stow's job and nothing else does it:

- **A new file in a layer.** After a build it is in `current/` and no link in `$HOME` points at it.
  The apply creates one.
- **A new directory.** The same, and the apply creates the directory in `$HOME` as well.
- **A file deleted from every layer.** The build takes it out of `current/` and leaves the link in
  `$HOME` pointing at nothing. The apply prunes it.
- **A file that changes hands between layers.** No apply. The path is unchanged, so the link is
  unchanged, and the build alone changes what it resolves to — which is also how a `local` override
  takes effect the moment you build it.

Apply when you are unsure: it builds first, so it is never less than a build, and the cost is one
stow run. The one case it does not cover is a whole directory leaving every layer, whose links stow
never visits — `shale doctor` finds those, and `docs/migrating.md` says what to do about them.

Nothing about a build is incremental. Every one of them composes every layer from scratch, so a
one-character edit costs what a first build costs: on this machine 4.1 seconds for a 2000-file tree,
against 31 milliseconds for a seven-file one, and the edited build was 4.08 seconds — roughly two
milliseconds a file, on whatever your disk does. There is nothing to tune and no cache to warm; keep
the trees the size the dotfiles actually are.

Shale clones a layer root that is missing and never touches it again; updating a checkout is git's
job. Pull each clone root, then apply:

```sh
git -C ~/.dotfiles/personal pull
git -C ~/.dotfiles/work pull
shale apply
```

The first build after anything changes a layer — a pull, or your own edit a moment ago — keeps
`current.old` without saying so, and the build after it removes it. Every build is quiet; what a
`shale doctor` in between the pull and the build reports is the built tree not yet matching the
layers, which is what that pull just did. There is no `shale sync`.
