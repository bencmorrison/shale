---
name: shale
description: Operate a home directory whose dotfiles are managed by shale, which composes layered directory trees into ~/.dotfiles/current and links them into $HOME with GNU Stow. Use when asked to change, add, remove or debug a dotfile or shell config (.zshrc, .profile, .gitconfig, .ssh/config, ~/.config/...) on a machine where ~/.dotfiles/shale.conf exists, `shale` is on PATH, or files in $HOME are symlinks into ~/.dotfiles/current. Also use when setting up shale on a new machine or when a `shale build`, `shale apply` or `shale doctor` run reports a problem.
license: MIT
---

# Operating shale

Shale stacks directories called *layers*, each mirroring `$HOME`, into one built tree at
`~/.dotfiles/current`, and stows that tree into `$HOME`. The layer listed later in
`~/.dotfiles/shale.conf` wins a path, whole file. Shale copies files byte for byte: it never merges,
templates or reads them.

Where `SHALE_DIR` is set, it replaces `~/.dotfiles` everywhere below. Your shell may not have read
the user's shell startup files. If `SHALE_DIR` is unset in your environment, search those files
(`~/.zshenv`, `~/.zshrc`, `~/.profile`, `~/.bashrc` and anything they source) for it before
assuming the default, and export it for every `shale` command you run.

## The rule that matters most

**Edit the layer, never the result.** `~/.dotfiles/current` is generated output, and every dotfile
shale manages in `$HOME` is a symlink into it. An edit to `~/.zshrc`, or to anything under
`current/`, lands in the built tree. The next build overwrites it with the layer copy and keeps the
edit only in `~/.dotfiles/current.old`, and the build after that deletes it. Opening `~/.zshrc` in an
editor or writing to it with a shell redirect follows the link and makes exactly that mistake.

## Before changing anything

1. Run `shale doctor`. It exits 0 with a last line of `no problems found` (sometimes followed by
   `, but read the note above`), or exits 1 with `N problem(s) found`. Notes and problems look alike
   line by line; only that closing line tells you whether any were problems. Every state doctor can
   see that stops a build or apply is a problem, so fix problems first. The exceptions: if doctor
   says there is no dotfiles directory or no `shale.conf`, stop and ask the user. That usually means
   `SHALE_DIR` is set somewhere you have not read. Never create `~/.dotfiles` or a `shale.conf`
   unless the user has asked you to set shale up.
2. Run `shale which PATH` for the file you are about to touch. It accepts `.zshrc`, `~/.zshrc`, an
   absolute path, or the built-tree path. The `winner` line is the layer file to edit. `shadowed`
   lines are lower layers whose copy is currently ignored, so editing one of those changes nothing.
   `merged` lines mean the path is a directory. Every listed layer contributes to it and nothing is
   shadowed, even when only one layer is listed.
3. If `which` exits 1 with `no layer provides`, shale does not manage that path. Edit or remove an
   unmanaged file in place as you would on any machine, including files in project directories such
   as `~/src/proj/.gitignore`. Bring a new file into shale only when the user wants it managed as a
   dotfile. Then choose a layer as described below.

## Making a change

**Changing a file a layer already provides:** edit the `winner` path from `shale which`, then run
`shale build`. The link in `$HOME` already points at the built tree, so the rebuild makes the edit
live. If `~/PATH` is not a symlink into `current/`, no apply has linked it yet, so run `shale apply`
instead.

**Adding a new file or directory:** create it inside a layer at the path it should have relative to
`$HOME`. A layer at `~/.dotfiles/personal/base` that holds `.config/foo/config` produces
`~/.config/foo/config`. Then run `shale apply`. A build alone puts the file in `current/` but nothing
links it into `$HOME`, and every command still exits 0.

**Removing a file:** delete it from every layer that provides it (`shale which` lists them), then run
`shale apply` so the leftover link is pruned. When a whole directory leaves every layer, stow leaves
the links inside it dangling. `shale doctor` names them, and removing those named links with `rm`
is the fix.

**Decide from the facts, do not guess.** A path you added or removed means `apply`; a change to the
contents of a file a layer already provides means `build`. If you cannot tell which you did, run
`shale doctor` and check whether `~/PATH` is a symlink into `current/`. If it is still unclear, ask
the user. Do not settle the question by applying: apply writes to `$HOME`, and the point of knowing
which case you are in is that you then check the right thing afterwards.

**A layer file that does not show up in `$HOME`:** if `which` shows a plain `winner` with no note and
nothing is at `~/PATH`, no apply has run since the path was added, so run `shale apply`. If `which`
prints a note, it says why: an ignore pattern, stow's own ignore list, or a conflict a build would
refuse. If neither applies, check
`~/.stowrc` and `~/.dotfiles/.stowrc` for an `--ignore`. Stow reads those files and shale does not,
and `shale doctor` notes that one exists.

**Choosing a layer for a new file:** read `shale.conf`. Its lines are `NAME PATH [URL]`, lowest
precedence first, and PATH is relative to `~/.dotfiles`. The first component of PATH is the *clone
root*. A URL appears on only one line per clone root, so find which clone root a layer belongs to
before deciding whether it is shared. A clone root with a URL on any line is a git clone that other
machines share. One with no URL exists only on this machine. Ask the user which layer they want when
the choice is not obvious from the file's purpose. Replacing a file by putting a copy in a higher
layer discards every line of the lower copy, so run `diff` between the two paths first and tell the
user what would be lost.

Layer repositories belong to the user. Do not commit, push or pull in them unless asked. To update
from upstream, the user pulls each clone root with git and then runs `shale apply`. There is no
`shale sync`.

## Adding to a config rather than replacing it

Shale has no append, include or fragment feature. Additive config is written into the config files
themselves:

- **Shell:** a common layout is an entry point in the base layer (`~/.profile`, `~/.zshrc`) that
  sources every file in a directory such as `~/.config/profile.d/*.sh`. Any layer can then add a
  numbered fragment. This is the user's convention, implemented by their own files, so check the
  base layer's entry point for the directory it actually sources.
- **git, ssh and tmux:** use their native includes. Git's `[include]` goes at the bottom of the file
  and does not glob. ssh's `Include` goes at the top, above any `Host` block. tmux's `source-file`
  goes at the bottom. At the wrong end, overrides are lost with no error. Confirm the effective value
  with `git config --get KEY`, `ssh -G HOST` or `tmux show-options -g`.

## Permissions

Git does not record directory modes, so a private path such as `.ssh` or `.gnupg` arrives at the
umask's default mode, and no command reports it. Declare the mode in a `.shale-modes` file at the top
of the **clone root**, for example `~/.dotfiles/personal/.shale-modes`, not inside `personal/base`.
Each line is a three-digit octal mode followed by an exact path relative to the layer root:

```
700  .ssh
600  .ssh/config
```

There are no globs, and setuid, setgid and sticky bits are refused. Shale never widens a directory
that already exists in `$HOME`.

## Keeping a file out of $HOME

List glob patterns in `.shale-ignore` at the top of the clone root, one per line. A pattern without
`/` matches at any depth, and a pattern containing `/` is anchored at the top of the layer. `**`, a
trailing `/` and backslashes are refused. The file is still built into `current/` but is not linked.

## When apply refuses

Stow usually refuses the whole apply and changes nothing. It can also stop part-way through, and
shale then says `stow failed part-way` and that `$HOME` may be partly relinked. After either outcome,
run `shale doctor` again before doing anything else, and act on what it names:

- **Something no apply put there: a file, a directory, or a link of the user's own.** This is the
  usual first-apply refusal, typically `.bashrc`, `.profile` or `.gitconfig`. It is also what a real
  directory where a layer ships a file produces. If the user wants to keep anything in it, copy that
  content into the right layer first. Then run `shale apply --replace`. It moves each such path to
  `~/.dotfiles/replaced/<timestamp>/`, deleting nothing, and applies.
- **A link owned by an older stow package.** `--replace` would move the link aside, leaving the old
  package half-stowed. Ask the user, then unstow the package first:
  `stow -D -d <directory holding the package> -t ~ <package>`. This is the one direct use of `stow`.
- **A symlink that already points at the built tree's own copy of its path, spelled as an absolute
  target.** It looks applied but is not, stow refuses it, and `--replace` does not move it.
  `shale doctor` names it. Remove the link with `rm` and apply again.

**Never use `stow --adopt`.** It moves the user's file into `current/`, and the next build overwrites
it. Never `rm` a file or directory that holds content. Removing a symlink that doctor names is fine.

## Reading results

Exit 0 means the command did what was asked. Exit 1 means shale diagnosed a problem. Exit 2 means the
invocation was wrong. Command output (`doctor`'s report, `which`'s answer, one `composed layer` line
per layer from a build) goes to stdout, and diagnostics go to stderr.

The commands are `build`, `apply`, `doctor`, `which PATH` and `help` (also `-h`/`--help`, or `shale`
with no arguments), which prints the usage text with the version. `--replace`, on `apply`, is the only
flag. There are no others.

## Setting up a new machine

Follow "Bootstrap on a new machine" in the README linked below, step by step. Ask the user for the
layers and repository URLs to put in `shale.conf`, because the config belongs to the machine and no
layer repository carries it. Every layer without a URL must exist as a directory before the first
apply, and modes for private paths must be declared before it.

## Stop and ask the user before

- deleting `~/.dotfiles/current.old` or anything under `~/.dotfiles/replaced/`, because both can hold
  the only copy of the user's file;
- reordering, adding or removing lines in `shale.conf`, which changes which layer wins every path;
- editing a layer that comes from a repository the user shares with other machines, when the change
  looks machine-specific;
- running `stow` yourself, including the `stow -D` above, or starting a second `shale` while one is
  running. Shale takes no lock.

## Further reading

- Install and bootstrap:
  https://github.com/bencmorrison/shale/blob/main/README.md
- `shale.conf`, `SHALE_DIR`, `.shale-ignore` and `.shale-modes`:
  https://github.com/bencmorrison/shale/blob/main/docs/configuration.md
- Every command in depth, exit codes, and what `doctor` checks:
  https://github.com/bencmorrison/shale/blob/main/docs/commands.md
- Layer authoring, precedence, modes and ignores:
  https://github.com/bencmorrison/shale/blob/main/docs/layers.md
- First apply, migrating from plain files or stow, and leaving shale:
  https://github.com/bencmorrison/shale/blob/main/docs/migrating.md
