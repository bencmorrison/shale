# Configuration

What goes in `shale.conf`, where shale keeps what it builds, and the two files a clone root can
carry — `.shale-ignore` and `.shale-modes`. [layers.md](layers.md) covers authoring the layers
themselves.

## shale.conf

`~/.dotfiles/shale.conf` lists one layer per line, lowest precedence first:

```
name    path              url
```

`path` is relative to `~/.dotfiles`. `url` says how to clone the *first component* of `path` — the
clone root — because several layers commonly come from one repository. Give the url once per clone
root and leave it off the other lines that share it. Blank lines and `#` comments are ignored. The
three fields are separated by whitespace and there is no quoting, so none of them can contain a
space; [layers.md](layers.md) says what shale can and cannot tell you when one does. A
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

## SHALE_DIR

`~/.dotfiles` is the default and not the only choice: set `SHALE_DIR` to an absolute path and that
directory holds `shale.conf`, the layers and the built tree instead. Every path `shale` prints names
the directory in force, the usage text in the README included. Trailing slashes are ignored; an unset or
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
[layers.md](layers.md#blocking-a-file-without-removing-it) has the detail.

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
Nothing that exits 0 tells you a mode was never declared: an `.ssh` or a credentials file is
composed and linked at the cloning machine's umask with every command exiting 0, `doctor` included,
because a mode nobody declared is not a fault shale can see. So declare before the first apply. A
url-less layer's clone root is the layer directory, which you create, so its `.shale-modes` can be
there first. A repository layer carries its `.shale-modes` committed with it, and its clone root
does not exist until the first apply creates it, so a mode not already committed with the layer
cannot be declared ahead of it: apply once, put the `.shale-modes` in the clone now on disk, commit
it, and apply again. The second apply narrows what you declared and widens nothing, so the first
apply's open mode does not stay open.

[layers.md](layers.md#permissions) has the whole of it.
