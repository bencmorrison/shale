# Authoring a layer

A layer is a directory tree that mirrors your home directory. Everything in it is copied, path for
path, into `~/.dotfiles/current`, and everything in `current` is linked into `$HOME`. A file at
`personal/base/.config/nvim/init.lua` becomes `~/.config/nvim/init.lua`. There is nothing else to
learn about the mapping.

Shale copies files. It never merges them, never templates them and never reads what is inside one.
Fragment directories, numeric prefixes and shared helper files are conventions for *your* layers,
expressed in the config formats themselves and implemented by the files you write. Shale has no
knowledge of any of them and treats every one as an ordinary file.

## Where a layer lives

Give a layer a subdirectory of its repository rather than the repository root:

```
base    personal/base     git@github.com:you/dotfiles.git
```

Shale skips `.git` at every depth, and the built tree carries a `.stow-local-ignore` that keeps a
top-level `README*`, `LICENSE*` and `COPYING` out of `$HOME`. Nothing else is filtered — a
`.gitignore` at the top of the layer becomes `~/.gitignore`, and a `README.md` further down becomes
a real link too. Name the repository root as the layer and `~/.github/`, `~/Makefile` and
`~/CONTRIBUTING.md` join them. The subdirectory is what keeps the repository's own furniture out of
the image.

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

A layer cannot replace a directory with a file, or a file with a directory. `build` calls that a
conflict, names both layers and the two paths, and rebuilds nothing; `which` reports the same pair
as a note. Rename one of them.

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
They do not all behave alike.

Git's `include.path` takes one path and does not glob: `path = ~/.config/git/conf.d/*.conf` includes
nothing at all, silently. The base layer therefore names each file it expects, and a higher layer
supplies it; an include whose file does not exist is ignored without error, so naming a file no
layer ships costs nothing.

```
[include]
  path = ~/.config/git/conf.d/50-work.conf
```

SSH's `Include` does glob, and `Include ~/.ssh/config.d/*.conf` over a directory that does not exist
is not an error. Put it at the *top* of `~/.ssh/config`, above any `Host` block: ssh keeps the first
value it obtains for each keyword, so a `Host *` default written above the `Include` wins over
everything the included files say.

tmux's own is `source-file`, and the same shape applies: the base layer's `tmux.conf` names what it
sources, and the layers supply those files.

## Symlinks in a layer

A symlink is copied as a symlink, target text unchanged, and it is a leaf: shale never recurses into
one, so a layer's symlink to a directory collides with a lower layer's real directory rather than
merging into it.

The target is resolved from where the link ends up inside `current/`, not from `$HOME`. A link to a
sibling works — `.exrc -> .vimrc` resolves inside `current/` and reading `~/.exrc` gets the file.
A link written as though it started in `$HOME`, such as `.exrc -> .dotfiles/current/.vimrc`, dangles
after a successful apply that reports nothing. Never point a layer symlink into `current/`.

Keep links relative. Stow refuses an absolute one and abandons the whole apply:

```
WARNING! stowing current would cause conflicts:
  * source is an absolute symlink .dotfiles/current/.viminfo-link => /home/you/.dotfiles/current/.vimrc
All operations aborted.
```

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
layer's own copy directly and you get `no problems found`, because no layer shadows *that* path.

Watch for `~/.stowrc` and `~/.dotfiles/.stowrc`. Their `--ignore` patterns are *appended* to the
ones stow is already using rather than overriding them, so a stray `--ignore=zshrc` leaves
`~/.zshrc` silently unlinked with `shale apply` exiting 0. `shale doctor` notes a resource file if
one exists.

## Never edit `current/`

`~/.dotfiles/current` is generated output and is disposable — a bad build is fixed by fixing a layer
and building again. Editing it edits the only copy of that change, and `~/.zshrc` being a link into
it makes that easy to do by accident. A build that finds a file in the built tree newer than the
layer it came from warns and keeps the tree it displaced:

```
shale: /home/you/.dotfiles/current/.zshrc was newer than /home/you/.dotfiles/personal/base/.zshrc
shale:   you edited the built tree; the layer wins
shale:   your version is kept at /home/you/.dotfiles/current.old/.zshrc
```

Copy your version back into the layer from `current.old`. The next build reaps `current.old`, so do
it before then. Never version `current/`, and never point a layer symlink into it.

## Updating layers after the first run

Shale clones a layer root that is missing and never touches it again; updating a checkout is git's
job. Pull each clone root, then apply:

```sh
git -C ~/.dotfiles/personal pull
git -C ~/.dotfiles/work pull
shale apply
```

There is no `shale sync`.
