# Authoring a layer

A layer is a directory tree that mirrors your home directory. Everything in it is copied, path for
path, into `~/.dotfiles/current` — or into `current` under whatever `SHALE_DIR` names, which the
README's *Configuration* section covers — and almost everything in `current` is linked into `$HOME`.
A file at `personal/base/.config/nvim/init.lua` becomes `~/.config/nvim/init.lua`. The exception is
the handful of names stow never links — version-control furniture, editor backups and emacs
autosaves — which *Where a layer lives* below sets out; apart from those there is nothing else to
learn about the mapping.

Shale copies files. It never merges them, never templates them and never reads what is inside one.
Fragment directories, numeric prefixes and shared helper files are conventions for *your* layers,
expressed in the config formats themselves and implemented by the files you write. Shale has no
knowledge of any of them and treats every one as an ordinary file.

The one thing a layer cannot carry in the tree itself is a mode. Git records no permission bits for
a directory and only the executable bit for a file, so an `.ssh`, a credentials file or anything
else private arrives on the next machine at that clone's umask — typically `755` and `644` — and no
command says so. Declare those modes in a `.shale-modes` at the top of the clone root before the
layer's first apply; [Permissions](#permissions) below is the whole of it.

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

That list needs no configuring and applies to every setup, so a file it covers reaches `current/`
and stops there with every command exiting 0. When something you shipped does not turn up in
`$HOME`, `shale which` on it names the list that is blocking it — see [When a pattern arrives
late](#when-a-pattern-arrives-late).

The three fields are separated by whitespace and there is no quoting, so a name, a path and a url
cannot contain a space or a tab. A directory called `my layer` cannot be a layer; rename it.
`base    my layer` is not a syntax error, because it is the same three strings as a layer at `my`
cloned from a repository called `layer` — so shale refuses it only where a directory called
`my layer` is there to say which was meant. Where it is not, the line is taken at its word and
`doctor` cannot see it, because no url is fetched until the build — and the build then fails on the
clone, unless `layer` really is a repository.

Names are ASCII. A layer name, and each component of a layer path, starts with an ASCII letter, a
digit or `_`, and may go on with those, `.` and `-`; a url is not restricted. A directory called
`léyer` therefore cannot be a layer any more than `my layer` can — rename it. Shale is the same tool
on macOS and on Linux, and an allow-list is what stops one `shale.conf` from meaning two different
things on them.

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

Stow does follow one, which is where it parts company with the composer. Where `$HOME` already holds
a real directory at that path, an apply links the contents of the link's directory into it and
leaves your own files there untouched — measured on stow 2.3.1 and 2.4.1. `doctor` reports that path
all the same, as one holding something no apply put there: what a build will put at the far end of a
link is not something it can read off the link, and one file of yours at a name that directory
provides is a refusal stow abandons the whole apply over. Move your directory aside if you want the
link, or leave it and read what the apply says.

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

## Permissions

**Git records none of this.** A tree entry has no permission bits at all, and a blob's are one
executable bit, so a freshly cloned layer has whatever the clone's umask gave it — `755` on most
machines, `775` on Debian and Ubuntu, and `644` for a `config` you committed at `600`. Nothing you
`chmod` in a working tree survives the trip to the next machine.

So a mode you want is a mode you declare. `.shale-modes`, at the top of the clone root and committed
with the layers, is plain text — which is the one thing git carries perfectly:

```
~/.dotfiles/
  shale.conf          this machine's
  personal/           the clone root, the git repository
    .shale-ignore     committed
    .shale-modes      committed
    base/  wsl/
```

```
# ~/.dotfiles/personal/.shale-modes
# Blank lines and lines starting with '#' are ignored.
700  .ssh
600  .ssh/config
700  .gnupg
600  .netrc
```

The mode first, in `chmod`'s own order, then the path — relative to the **layer root**, which is the
top of the tree a layer mirrors, so `.ssh/config` is the same spelling you would use with `shale
which`. One or more spaces or tabs between them; only the first run separates the two, so a path with
a space in it needs no quoting.

A declaration covers a file as readily as a directory. A `600` on a config is as lost to a clone as a
`700` on a directory, and shale does not have to guess which of your files are secret if you tell it.

### What a declaration does

`build` sets the declared mode on the path in `~/.dotfiles/current`. `apply` sets it again on the
directory stow makes in `$HOME` — stow carries no mode of its own, creating each directory with
`mkdir` and the umask of whoever ran it. A declared *file* needs nothing in `$HOME`: what is there is
a symlink into `current/`, and reading through it reads the file the build already set the mode on.

The result is the same on every machine at every umask, which is the whole point:

```
committed:                     .ssh=700  .ssh/config=600
clone at umask 022, apply:     ~/.ssh=700  ~/.ssh/config=600
clone at umask 002, apply:     ~/.ssh=700  ~/.ssh/config=600
```

One digit is shale's rather than yours: **the owner's triad is always `7` for a directory.** Only
that digit, and the other two are taken exactly as written:

```
declared:  050  500  000  070  007
composed:  750  700  700  770  707
```

Shale has to write into a directory it has just created — the next layer composes into it, and an
interrupted build has to be able to delete the tree it left behind — and a directory in `$HOME` you
cannot enter is not an arrangement any dotfile wants, nor one stow can descend to link what is
inside. The group and world bits, which are the ones that decide whether `~/.ssh` is safe, are
untouched by it. A declared *file* gets all three digits unchanged.

**A directory nobody declares is `755`** in `current/`, at any umask, and shale sets nothing at all
for it in `$HOME` — there it is stow's `mkdir` against your umask, as it was before any of this.
Shale acts on the paths it was told about and no others.

### What stops a build

- **A path no layer provides.** A declaration for a file you have since renamed is otherwise a silent
  no-op, and the mode goes missing on the next machine with every command exiting 0. Shale names the
  file and the line: `.shale-modes:2: no layer provides '.ssh/id_rsa'`. Correct the path or remove
  the line.
- **A path that is a symlink.** `chmod` follows one, so honouring the line would set the mode of
  whatever it points at, somewhere else entirely, and there is no portable way to do the other thing.
- **Four octal digits.** Setuid, setgid and the sticky bit are refused rather than set on a path in
  your home directory: everything created inside a `setgid` directory afterwards takes its group,
  including files no layer ever named. `0700` is refused by the same rule, and the message names the
  three digits to write instead.
- **A glob.** `600 .ssh/*` is refused: one exact path per line. Exact paths are also what makes
  precedence between two clone roots something you can read.
- **`..`, a leading `/`, a trailing `/`, an empty component.** Each refused by name.

`shale doctor` finds every one of these before a build, and each is a problem rather than a note: no
build runs until it is fixed.

### Two roots declaring one path

Config order, like everything else — the last one read wins. `shale which` names the line that
decides a path, and the lines it shadows:

```
$ shale which .ssh
merged    work  /home/you/.dotfiles/work/base/.ssh/
merged    base  /home/you/.dotfiles/personal/base/.ssh/
shale: note: '.ssh' is declared mode 755 at /home/you/.dotfiles/work/.shale-modes:1
shale:   shadowed: /home/you/.dotfiles/personal/.shale-modes:1 declares 700
shale:   shale sets that mode on /home/you/.dotfiles/current/.ssh
shale:   and on /home/you/.ssh, where it never widens a directory that was already there
```

`merged` rather than `winner` because both providers are directories and directories merge; the
trailing slash is how `which` writes one. For a declared file the last line is different, because
what is in `$HOME` is a link and the mode lives on the copy the link points at:

```
$ shale which .netrc
winner    base  /home/you/.dotfiles/personal/base/.netrc
shale: note: '.netrc' is declared mode 600 at /home/you/.dotfiles/personal/.shale-modes:2
shale:   shale sets that mode on /home/you/.dotfiles/current/.netrc
shale:   /home/you/.netrc is a link into that tree, so that is the mode read through it
```

Neither last line is printed for a path an ignore list covers: no apply links one, so no apply sets a
mode on one, and the note below the table says so.

### Never-widen, in `$HOME`

**A directory that was already in `$HOME` before the apply is only ever narrowed.** Keep `~/.ssh` at
`700` and add a layer declaring `755`, and the apply leaves your `700` alone. The same declaration
against a `~/.ssh` that does not exist yet, or one at `775`, gives you `755` exactly.

The asymmetry is the rule: a declaration is what a layer's author asked for, the mode on the machine
is yours, and a mode fix that quietly opens `~/.ssh` is worse than the mode being wrong.

`apply` says so in one line, with the count and no more, because this is a standing disagreement
rather than an event and it lasts until you change one side:

```
shale: 1 directory in /home/you was left as it is: shale never widens one that was already there; 'shale doctor' names it
```

`shale doctor` is where the detail is, as a note rather than a problem — the directory that survives
is the *safer* of the two, so nothing is broken and `doctor` still exits 0:

```
shale: an apply left 1 directory in /home/you as it is
shale:   /home/you/.ssh is 700, and shale is asked for 755
shale:   shale sets the mode of a directory it creates in /home/you, and never widens one that was already there
shale:   to take the mode the declaration asks for, chmod that directory yourself
shale:   to keep the mode it has, change the declaration: 'shale which' names the line that decides a path
```

`apply` exits 0 either way — nothing it was asked to do was left undone. A mode it cannot set at all
is a different thing: that is named on stderr and `apply` exits 1 with the links in place.

### A declaration that never reached `$HOME`

The other way a line can go unhonoured is the path not being linked at all — stow declined it, or
something removed the link. The mode is on the copy in `current/` and on nothing anybody uses, and
`doctor` says so as a note:

```
shale: 1 declared mode is not on anything in /home/you
shale:   /home/you/.dotfiles/personal/.shale-modes:4 declares 600, and nothing in /home/you is at /home/you/.netrc
shale:   the built tree holds each of these paths with the declared mode on it, and an apply is what puts one in /home/you
```

Nothing is set on a path an ignore list blocks, in `$HOME` or below it, declaration or no
declaration. No apply links anything there, so the `~/.cache` a layer names in `.shale-ignore` is
yours and shale does not touch its mode — the same paths `shale which` reports as ignored. `doctor`
says nothing about one either.

### Where the file is read

One `.shale-modes` per clone root — the top of the first component of each layer path — and every one
of them concatenated into a single list, exactly as `.shale-ignore` is. Shale reads these files and
composes none of them, at any depth, the way it never composes a `.git`, so one never reaches `$HOME`
even when a clone root is named as a layer itself. A `.shale-modes` beside the config, or anywhere
inside a layer, is read by nothing; `shale doctor` names those, because silence there is a file of
declarations doing nothing.

## Permissions on the built tree itself

`~/.dotfiles/current` is `700`, and so are `current.new` and any `current.old`. No layer provides
those and no line declares them: shale picks one mode, and picks the narrowest, on every machine at
every umask.

It matters more than it looks. Every symlink shale makes in `$HOME` resolves *through* that
directory, so its mode is what decides who may read the file at the end of the link and who may
replace it — and what is inside it is a copy of every file of every layer, at whatever mode a clone
left it. Read the first paragraph of the section above again: a clone gives you `644` on a `config`
you committed at `600`, and it stays `644` until a line says otherwise. `700` on the tree is what
contains that mistake in the meantime.

The cost is that nothing but you can walk the built tree. Nothing shale runs needs to — stow reads it
as you, and so does every other command here — but if your `$HOME` is group-readable by design and
something else reads a file shale linked, it resolves the link into this directory and stops. There
is no option to relax it, and a `chmod` will not hold: a build does not chmod `current`, it renames a
fresh `700` tree over the old one. `shale doctor` says so where it finds one that is not `700`, as a
note:

```
shale: /home/you/.dotfiles/current is mode 0755, and shale builds its permission bits at 0700
shale:   every symlink shale makes in /home/you resolves through that directory, so its mode decides who can read what they point at and who can replace it
shale:   the next build renames a fresh tree over it, made with 0700 and with whatever set-id or sticky bit /home/you/.dotfiles gives a new directory, so a chmod here does not hold
```

Only the permission bits are shale's. A `setgid`, `setuid` or sticky bit on the built tree came from
`~/.dotfiles` and shale neither sets nor clears it: no umask masks one, a new directory takes the
set-group-ID bit from the directory it is made in, and the rename that installs a build carries it
across. So a `~/.dotfiles` on a shared volume gives you `2700` here rather than `0700`, on every
build, and `doctor` says nothing about it — that bit is yours. It is the mode of a genuinely wide
tree, `2755` say, that `doctor` reports, and it reports the whole of it so you can see both halves.

The modes *inside* the tree are untouched by this: `current/.ssh` is whatever the section above says
it is — the declared mode, or `755` for a directory nobody declared.

## What a layer must not ship

Never a `.stow-local-ignore` at the top of a layer. Shale writes the one in `current/` itself — it is
the only file shale creates rather than copies — and a layer whose own top level holds one fails the
build, naming the layer, whether it is a file or a directory. Further down it is an ordinary file:
`~/.config/.stow-local-ignore` composes and gets linked like anything else.

That refusal tests the exact name, and stow reserves the name at the top of the tree a shade more
widely than the build does: stow adds it to whatever ignore list it reads, and matches it with one
trailing newline on the end as well. So a layer whose top level holds `.stow-local-ignore` followed
by a newline composes without a word and is linked by no apply, and a *directory* so named takes
everything under it. `shale which` says so for those paths. Renaming that top-level entry is the
only fix; there is no line to edit anywhere. None of it reaches below the top of the layer, where
that name — with the trailing newline or without — is an ordinary file that gets linked, and where
`which` rightly says nothing about it.

Never shale itself: applying a layer that ships it would replace the script with a link into
`current/`. `shale doctor` reports that, judged against the path the running shale was invoked by.
Run the installed copy — `~/.local/bin/shale doctor` — and the finding names the layer. Run the
layer's own copy directly and there is no such finding, because no layer shadows *that* path.

Watch for `~/.stowrc` and `~/.dotfiles/.stowrc`. Their `--ignore` patterns are *appended* to the
ones stow is already using rather than overriding them, so a stray `--ignore=zshrc` leaves
`~/.zshrc` silently unlinked with `shale apply` exiting 0. `shale doctor` notes a resource file if
one exists; what is in it is stow's business and shale never reads it.

## Blocking a file without removing it

`.DS_Store`, vim swap files like `.init.lua.swp`, `Thumbs.db`: junk that lands in a layer through
ordinary use, that the repository's `.gitignore` hides from `git status` but not from shale — shale
composes from the filesystem, not from git's index — and that stow's default list does not cover.
Block it with a `.shale-ignore` file at the top of the clone root, committed with the layers:

```
~/.dotfiles/
  shale.conf          this machine's
  personal/           the clone root, the git repository
    .shale-ignore     committed
    base/  wsl/
```

```
# ~/.dotfiles/personal/.shale-ignore
# Blank lines and lines starting with '#' are ignored.
.DS_Store
*.swp
Thumbs.db
```

Every build reads one such file per clone root — the top of the first component of each layer path —
and **concatenates all of them into one list**. There is no per-layer or per-repository scope, and
there cannot be: stow's ignore list applies to the built tree as a whole, so a file that looked
scoped and was not would be worse than none. Shale reads these files and composes none of them, at
any depth, the way it never composes a `.git` — so one never reaches `$HOME` even when a clone root
is named as a layer itself. A `.shale-ignore` beside the config, or anywhere inside a layer, is read
by nothing; `shale doctor` names those, because silence there is a file of patterns doing nothing.

### The pattern syntax

Globs, in the familiar subset of a `.gitignore`'s:

| | |
|---|---|
| `*` | any run of characters inside one path component |
| `?` | exactly one character |
| `[abc]`, `[a-z]` | one character from the set |
| `[!abc]`, `[^abc]` | one character not in the set |
| anything else | itself, including a space, a `#` and a `$` |

A pattern with no `/` matches a path component **at any depth**: `.DS_Store` covers every one in the
tree. A pattern with a `/` anywhere is anchored at the **top of the built tree**, which is where a
leading `/` would put it — `/notes` and `.config/nvim/swap` name one place each, and the leading
slash is optional. A directory that matches takes everything under it, so `secrets` removes
`secrets/key` too, and `/keep/*` removes `keep/deeper/kept` as well as `keep/gone`: stow skips a
directory whose name matches and never looks inside it.

A `#` starts a comment only at the beginning of a line. Anywhere else it is part of the pattern, and
a pattern that has to *begin* with one is written as a set: `[#]notes[#]`.

That subset is the whole of it, whichever shell you run `shale` from. Setting an option does not on
its own reach the scripts a shell starts, but exporting one does: `export SHELLOPTS` hands every
`set -o` option on to every child, `export BASHOPTS` does the same for the `shopt` options on bash
4.1 and later — there is no such variable on the bash 3.2 macOS ships — and a `BASH_ENV` file runs
before any script bash starts and can set anything, on every version. In a shell that passes on an
`extglob`, `@(a|b).swp` would be an alternation rather than a filename, and with a `nocasematch`,
`*.swp` would cover a file called `A.SWP`. Shale turns those off for itself before it reads a
pattern, so a committed `.shale-ignore` says the same thing from your login shell, from cron and from
CI.

Nothing you write is a regular expression. Shale translates each glob into the regexp stow's ignore
file wants and escapes every other character on the way, so a `+`, a `(` or a `.` in a filename is
that character and nothing else. What it will not translate it refuses by name, with the file and
the line, and the build stops there:

| Refused | Why |
|---|---|
| a backslash | there is no escape character; every character but `*`, `?` and `[...]` is already literal |
| `**` | a pattern with no `/` matches at any depth already, so `**/x` is written `x` |
| a trailing `/` | stow's list cannot say "directories only"; without the slash it matches the directory and takes its contents |
| a leading `!` | there is no negation, and stow has none to give it |
| `[` with no `]`, an empty `[]`, `a//b` | not a pattern shale can read |
| a set starting `[.`, `[:` or `[=` | perl reads those as syntax it does not implement and complains on stderr on every apply |
| `[a-Z]`, `[z-a]`, `[[:alpha:]]`, a `/`, a space or a non-ASCII character inside `[...]` | a set holds letters, digits, `_ . , : = @ % ~ + - #` and ranges within one of those three alphabets |

That refusal is the point: stow joins every pattern in the file into one alternation and compiles it
with perl, in an order perl gives it rather than the file's. A pattern that is not self-contained
does not fail on its own — an unclosed `(` swallows the `|` after it and takes the next pattern with
it, a `[a-Z]` makes the whole list fail to compile so that no apply runs at all, and a lone `\` can
silently disable one of stow's own default entries, differently on each run. All of it was measured
before this syntax existed.

### When a pattern arrives late

The file is still composed into `current/`; the pattern changes only what stow links. Two commands
report what the list is doing. `doctor` names the list where the built tree holds a file it covers,
and says nothing about a list that is covering nothing — a pattern written before the junk it is for
is withholding no file from `$HOME` and has nothing to report:

```
$ shale doctor
shale: 3 ignore patterns are in force, and the built tree holds 1 file they cover
shale:   shale writes them into /home/you/.dotfiles/current/.stow-local-ignore, below stow's own list, so no apply links a path they match
shale:   a link an earlier apply made at such a path stays until it is removed by hand, and any this report found are named above
shale:   /home/you/.dotfiles/personal/.shale-ignore
shale:   line 3: .DS_Store
shale:   line 4: *.swp
shale:   line 5: Thumbs.db

$ shale which .config/nvim/.init.lua.swp
winner    base  /home/you/.dotfiles/personal/base/.config/nvim/.init.lua.swp
shale: note: '.config/nvim/.init.lua.swp' is on the ignore list — no apply links it
shale:   /home/you/.dotfiles/personal/.shale-ignore:4: *.swp
shale:   shale writes that pattern into /home/you/.dotfiles/current/.stow-local-ignore, which is the list stow reads
```

`which` answers before the first build, because it matches the globs itself rather than reading the
file a build writes.

It reads stow's own default list the same way, and says which list it is, because the remedy is not
the same — there is no line to edit, and every build rewrites the file:

```
$ shale which 'notes~'
winner    base  /home/you/.dotfiles/personal/base/notes~
shale: note: 'notes~' is on stow's own ignore list — no apply links it
shale:   stow's default ignore list: .+~
shale:   shale writes that list into /home/you/.dotfiles/current/.stow-local-ignore on every build
shale:   that list has no line to edit: nothing but a different name gets this path past it
```

`doctor` says nothing about that list. A layer holding a `.gitignore` is not a problem, and the
answer is only wanted about the one path you went looking for.

**Adding a pattern does not unlink what is already linked.** Stow skips an ignored path rather than
unstowing it, so the link an earlier apply made stays exactly where it was, and the apply that comes
after the pattern exits 0 without mentioning it. `shale doctor` names every such link:

```
shale: /home/you/.DS_Store is a link into the built tree at a path the ignore list covers
shale: remove the link named above
shale:   it was made by an apply that ran before that path was on the list stow reads
shale:   stow skips an ignored path rather than unstowing it, so no build or apply removes it
shale:   the built tree is correct and needs no rebuilding
```

Deleting them is the whole of the fix — nothing needs rebuilding or reapplying — and `shale which`
says the same thing about the one path you asked about, naming the link it found.

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
moved in — copy it into a layer, and do it before the next build, which reaps `current.old` and says
so on stderr while that tree still holds anything the layers cannot produce again. If it is
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
built copy is the layer's again and the mismatch is gone, so nothing names your file any more — this
direction of the comparison, the one an adopted older file lands in, is the one where the window
closes quietly. The other direction it does not: where `current.old` holds a file *newer* than the
layer copy that replaced it, one whose *kind* the layers have changed since — a path that is a
directory or a symlink in a layer now, where the kept tree has a file, or a whole kept directory at
a path a layer now provides as a symlink — or one at a path no layer provides at all, `doctor` says so in a note giving the directory's path and saying the next build
removes it. That is a note about a directory
and not about your file — the walk that decides it stops at the first path it cannot account for and
reports no path at all, so which file it was is not in the note. So the window before the build is
still the one worth having: if you use `stow --adopt` against `current/`, run `shale doctor` first.

## The everyday loop: build, and when to apply

Edit a file a layer already has, and `shale build` is the whole loop. `~/.zshrc` is a link into
`current/`, so rebuilding the tree it points at makes the edit live at that instant — there is
nothing for stow to do, because the link is already right.

That build says nothing about the file you edited, beyond the `composed layer` line it prints for
each layer. It keeps `current.old` — the built copy your edit replaced was older than
the layer copy, which is the state a file moved into the tree also leaves, and shale keeps the
previous tree either way — but it prints no message about it, because a message there is a message
on every edit. `shale doctor` is where a built tree its layers have moved past is reported, and
after a build the mismatch is gone. It says nothing about the `current.old` that build left either,
and the reason is what is in it: every copy in that tree is a layer's own file from the build before,
older than the layer copy that replaced it, so the layers still have all of it. A tree holding
something they do not — a file newer than its layer copy, one the layers have since made a directory
or a symlink, or one at a path no layer provides — is the case doctor does report, below.

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
`current.old` without saying so, and the build after it removes it without saying so either: what
goes is the layers' own files at the layers' own age. Builds are quiet through the whole of that
loop; what a `shale doctor` in between the pull and the build reports is the built tree not yet
matching the layers, which is what that pull just did. There is no `shale sync`.
