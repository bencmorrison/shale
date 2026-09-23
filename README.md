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
system already has. Shale installs nothing, including itself. `shale doctor` names every tool it
cannot find, so it is the fastest answer to a machine where something will not run.

Shale is written to a floor of bash 3.2 and POSIX utility flags, which is what macOS ships. The test
suite runs in CI on Linux and on macOS, the macOS job under `/bin/bash` so that the floor is
exercised rather than assumed. WSL is Linux for shale's purposes. No other platform is exercised.

## Install

Shale is one file, so installing it is downloading that file and putting it on your `PATH`. The
quick way is the released script, from any directory — `-O` writes it into the working one:

```sh
mkdir -p ~/.local/bin
curl -fsSLO https://github.com/bencmorrison/shale/releases/latest/download/shale
mv shale ~/.local/bin/shale
chmod +x ~/.local/bin/shale
command -v shale
```

`latest/download` tracks whatever release is newest; `download/v2.0.0` in its place pins the version
that URL gives you.

Cloning gets the same script plus `examples/` and `docs/`, which the bootstrap below and
[docs/migrating.md](docs/migrating.md) both draw on, and the agent skill in `skills/shale/`, which
[Letting a coding agent drive it](#letting-a-coding-agent-drive-it) covers. These commands, and
every one in the bootstrap, run from the clone's root:

```sh
git clone https://github.com/bencmorrison/shale.git
cd shale
mkdir -p ~/.local/bin
cp shale ~/.local/bin/shale
command -v shale
```

`~/.local/bin` is the usual choice, and it is often not on `PATH` on the machine you are
bootstrapping, because putting it there is a thing your dotfiles do and they are not applied yet.
That is what the `command -v` line is for: nothing printed means this shell cannot find shale, and
`export PATH="$HOME/.local/bin:$PATH"` is enough to get through the bootstrap below. Applying your
layers is what makes it permanent.

Shale never needs to know where it lives, and there is nothing else to install. There is no flag
that prints the version: the header of the usage text names it, and that is the copy you have.

## Configuration

`~/.dotfiles/shale.conf` lists one layer per line, lowest precedence first — `name`, a `path`
relative to `~/.dotfiles`, and the `url` to clone that path's top directory from, given once per
clone:

```
base    personal/base     git@github.com:you/dotfiles.git
wsl     personal/wsl
work    work/base         git@github-work:employer/dotfiles.git
local   local
```

Reordering the lines changes precedence; nothing else does. The config belongs to the machine, not
to any layer repo. [docs/configuration.md](docs/configuration.md) has the full format, `SHALE_DIR`
for keeping all of this somewhere other than `~/.dotfiles`, and the two files a clone root can
carry: `.shale-ignore` to keep a file out of every apply, and `.shale-modes` to declare modes.

## Bootstrap on a new machine

```sh
mkdir -p ~/.dotfiles && chmod 0700 ~/.dotfiles
cp examples/minimal.conf ~/.dotfiles/shale.conf    # from a clone; or write it by hand
$EDITOR ~/.dotfiles/shale.conf                     # list this machine's layers
mkdir -p ~/.dotfiles/local                         # a layer with no url is yours to create
```

`examples/minimal.conf` is one repository layer, cloned over https, and a `local` directory. Without
a clone, write those two lines yourself in the format above. Shale clones a layer that has a url and
creates no layer that has not, so every url-less layer has to exist before the first apply —
`local`, in both shipped examples. `examples/wsl-work.conf` is the larger example, with several
layers from one clone and a second repository reached through an SSH `Host` alias, which resolves
only once an `.ssh/config` defining it is in place.

Declare any mode that matters before that first apply. Git records no permission bits for a
directory and only the executable bit for a file, so an `.ssh` or a credentials file is linked at
the cloning machine's umask — typically `755` and `644` — with every command exiting 0. Declarations
go in a `.shale-modes` at the top of the clone root, which for a url-less layer is the layer
directory itself:

```
# ~/.dotfiles/local/.shale-modes
700  .ssh
600  .ssh/config
```

```sh
shale apply                        # clones what is missing, builds, links
```

[docs/configuration.md](docs/configuration.md#declaring-the-mode-of-a-path) has the whole format,
and what to do for a repository layer whose modes are not committed yet.

Dotfiles already in `$HOME`? Whether they are ordinary files or stow packages you stow by hand, they
have to stop being files at those paths before the first apply, and anything of yours worth keeping
has to be copied into a layer by hand — shale merges nothing.
[docs/migrating.md](docs/migrating.md) takes both starting points step by step.

An account nobody has touched has some already: `useradd` copies `.bashrc`, `.profile` and
`.bash_logout` into it from `/etc/skel`, and one `git config --global user.email you@example.com`
writes `.gitconfig`. A layer providing any of those is refused on the very first apply, and
`apply --replace` is the one command that clears it — it moves the paths `doctor` reports as
*holding something no apply put there* into `~/.dotfiles/replaced/<timestamp>/`, keeping the tree
each one had, and then applies. It moves and never deletes;
[docs/commands.md](docs/commands.md#moving-aside-what-stow-would-refuse) says exactly what it
covers.

## Everyday use

After editing a file a layer already has, `build` is the whole loop: `~/.zshrc` is a link into
`current`, so rebuilding the tree it points at makes the edit live at that instant. `apply` is for
when a *path* appears or disappears — a file or directory added to a layer has nothing in `$HOME`
pointing at it until stow makes the link, and one deleted from every layer leaves a link behind
until stow prunes it. Apply when unsure: it builds first, so it is never less than a build.
[docs/layers.md](docs/layers.md) has the boundary case by case.

To update your layers, pull each clone root with git, then apply:

```sh
git -C ~/.dotfiles/personal pull
shale apply
```

To find out where a file in `$HOME` comes from, ask `shale which`; it names every layer providing the
path, winner first, or marks each `merged` for a directory:

```
$ shale which .zshrc
winner    work   /home/you/.dotfiles/work/base/.zshrc
shadowed  base   /home/you/.dotfiles/personal/base/.zshrc
```

Edit the layer, never `~/.dotfiles/current` — that is generated output, and the next build replaces
it. Shale only ever replaces whole files; to have several layers add to one config, the config format
has to include a fragment directory, which [docs/layers.md](docs/layers.md#fragment-directories)
writes out.

## Commands

```
$ shale
shale 2.0.0 - layered dotfiles builder

usage:
  shale build          compose the configured layers into ~/.dotfiles/current
  shale apply          build, then stow current into your home directory
    --replace          move aside what 'doctor' says no apply put there
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
above. The four commands in it are the whole surface: there are no others, and `apply --replace` is
the only option.

| Command | What it does |
|---|---|
| `shale build` | Clones any missing clone root that has a url, then composes the layers into `~/.dotfiles/current` |
| `shale apply` | Builds, then stows `current` over `$HOME`, replacing the last apply's links |
| `shale doctor` | Checks the prerequisites, the config, the layers, the built tree and `$HOME`; it will not exit 0 on a setup `build` or `apply` is certain to refuse |
| `shale which PATH` | Names every layer providing `PATH`, winner first or `merged` for a directory, and when `build` would refuse the set |

[docs/commands.md](docs/commands.md) has each command in depth: exit codes, what goes to stdout and
stderr, what a build reports, `which`, conflicts between layers, and everything `doctor` checks.

## Letting a coding agent drive it

Ask a coding agent to change a dotfile on a shale machine and it will edit `~/.zshrc` — a link into
`current/`, so the next build throws the edit away. `skills/shale/SKILL.md` is the fix: a skill that
teaches an agent to find the owning layer with `shale which`, edit that, and build or apply, along
with `--replace`, `.shale-modes`, `.shale-ignore` and what to ask before touching. It is written to
the open [Agent Skills](https://agentskills.io/specification) format and names no agent's own tools,
so any agent that reads that format can use it, and one that does not can be pointed at the file as
ordinary instructions.

Shale does not install it, any more than it installs itself. Copy the `shale` directory — the
directory, not only the file, because the format names a skill after the directory holding it — into
wherever your agent reads skills from; each agent's own documentation says where, and
[agentskills.io/clients](https://agentskills.io/clients) links to it for most of them. From a clone:

```sh
mkdir -p <your agent's skills directory>
cp -R skills/shale <your agent's skills directory>/
```

Without a clone, fetch the one file into a directory named `shale`:

```sh
mkdir -p <your agent's skills directory>/shale
curl -fsSL -o <your agent's skills directory>/shale/SKILL.md \
  https://raw.githubusercontent.com/bencmorrison/shale/main/skills/shale/SKILL.md
```

That URL takes the skill from `main`, which can be ahead of the released script; put the tag your
script came from in place of `main` to match them, for any release that includes the skill.

That skills directory is usually under `$HOME`, so the tidier home for it is a layer: put the `shale`
directory at the same path inside one of your layers and `shale apply` links it into place on every
machine that layer reaches. Either way it is a copy of one version; take a new one when you take a
new script.

An agent that reads `AGENTS.md` but not skills can be given one line in the `AGENTS.md` it reads,
naming the path of the copy you made and saying to read it before touching a dotfile.

## Uninstalling

`stow -D -d ~/.dotfiles -t ~ current` removes every link shale made and leaves your layers, config
and built tree alone; `shale apply` puts them back. To leave shale for good and keep your dotfiles
as ordinary files, follow
[docs/migrating.md](docs/migrating.md#leaving-shale-with-your-dotfiles) — the order of its steps
matters, and the last one deletes `~/.dotfiles`.

## Documentation

- [docs/configuration.md](docs/configuration.md) — `shale.conf`, `SHALE_DIR`, `.shale-ignore` and
  `.shale-modes`.
- [docs/commands.md](docs/commands.md) — every command in depth, and the built tree.
- [docs/layers.md](docs/layers.md) — authoring a layer: what goes where, fragments, symlinks,
  permissions, and the build-or-apply boundary case by case.
- [docs/migrating.md](docs/migrating.md) — moving to shale from plain files or stow packages, what a
  first apply refuses, and leaving shale again.
- [docs/known-limits.md](docs/known-limits.md) — what shale does not do, and what it costs.
- [skills/shale/SKILL.md](skills/shale/SKILL.md) — the agent skill that
  [Letting a coding agent drive it](#letting-a-coding-agent-drive-it) installs.

## Licence

MIT. See `LICENSE`.
