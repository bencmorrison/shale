# Migrating from a plain stow setup

You have `~/.dotfiles/common` and `~/.dotfiles/local` as stow packages, and you stow them by hand.
Shale composes the same trees into one package and stows that. The move is mechanical: unstow what
is there, turn the packages into layers, list them, apply.

Do it from a shell you can afford to lose. Unstowing removes `~/.zshrc`, `~/.profile` and everything
else the package owns, and every shell opened between then and the apply falls back to the system
defaults. Keep a second session already running, and do this over a connection you are not relying
on the dotfiles to make.

## Unstow first

```sh
stow -D -d ~/.dotfiles -t ~ common
stow -D -d ~/.dotfiles -t ~ local
```

Name the stow directory the packages actually live in — `-d ~/dotfiles-old`, or wherever it is — and
give `-t ~` explicitly. Both flags are worth passing even when the defaults would be right: stow
reads `./.stowrc` and `~/.stowrc`, and a `--dir=` or `--target=` in either silently moves the
operation somewhere else.

`stow -D` removes the links it made, and what is left depends on how the package went in. A folded
`~/.config` is itself a link, so it goes with the rest and nothing remains; a package stowed with
`--no-folding` leaves the directories stow created behind it, empty. Either one is expected and
harmless.

## Turn the packages into layers

A stow package and a layer are the same shape — a tree mirroring `$HOME` — so this is a rename. The
checkout becomes the *clone root*, and the layer becomes a subdirectory inside the repository:

```sh
git -C ~/.dotfiles/common remote -v          # if it was a checkout, note where from
mv ~/.dotfiles/common ~/.dotfiles/personal
cd ~/.dotfiles/personal
mkdir base
git mv .config .profile .zshrc base/         # every entry that belongs in $HOME
git commit -m "Move the layer under base/"
git push
```

`git mv` moves what git tracks. Move anything untracked you want to keep with plain `mv`.

Then write `~/.dotfiles/shale.conf`, lowest precedence first:

```
base    personal/base     git@github.com:you/dotfiles.git
local   local
```

The url says how to obtain `personal`, the clone root — not how to obtain the layer. That is why the
repository has to *hold* `base/` rather than *be* it. Rename the checkout straight to
`personal/base` and leave the url on the line, and this machine tells you nothing: shale clones only
where nothing exists, so the url is never used, and `doctor` sees two real directories and reports
clean. The next machine clones the repository to `~/.dotfiles/personal`, finds no `base` inside it,
and stops on the config line rather than on the repository:

```
shale: layer 'base' has no directory at /home/you/.dotfiles/personal/base, but its clone at /home/you/.dotfiles/personal exists
shale:   check the path on line 1
```

If `common` was never a checkout there is nothing to move inside and nothing to push: `mkdir -p
~/.dotfiles/personal`, `mv ~/.dotfiles/common ~/.dotfiles/personal/base`, and leave the third field
off the line. Add a url once a repository exists that holds the layer in a subdirectory.

Check what shale makes of the config before touching `$HOME`:

```sh
shale doctor
shale build
shale which .zshrc
```

`doctor` reports any package left in `~/.dotfiles` that no config line claims, which is how a
package you forgot to convert shows up:

```
shale: /home/you/.dotfiles/common is not a configured layer
shale:   it may be a leftover stow package, a stray clone, or a layer you removed from the config
```

Then `shale apply`.

Expect `$HOME` to look different afterwards. Where hand-stowing a single package often leaves
`~/.config` as one symlink into it, shale's apply leaves `~/.config` a real directory holding one
link per file. Files that other tools write into those directories then land in `$HOME` rather than
inside the built tree, where the next build would discard them.

## What a first apply refuses, and why

Stow plans the whole operation and abandons all of it if any part conflicts, so a failed apply
changes nothing in `$HOME`. The built tree is fine; clear what stow named and apply again. Apply
restows, so stow runs an unstow phase and then a stow phase, and a collision both phases see is
named once for each.

An old package still stowed over a path the built tree also provides:

```
WARNING! stowing current would cause conflicts:
  * existing target is stowed to a different package: .profile => .dotfiles/common/.profile
All operations aborted.
```

Unstow that package. Every path both packages provide collides this way, and stow has no notion of
one package outranking another.

A folded directory owned by a package outside `~/.dotfiles` — the old setup's `~/.config` being a
single link into it:

```
WARNING! unstowing current would cause conflicts:
  * existing target is not owned by stow: .config => dotfiles-old/common/.config
WARNING! stowing current would cause conflicts:
  * existing target is not owned by stow: .config
All operations aborted.
```

Stow can split a folded directory apart when the link points inside the stow directory it was given,
and it cannot when the link points anywhere else. Unstow the old package and the fold goes with it.

An ordinary file, not a link, already sitting at the path:

```
WARNING! unstowing current would cause conflicts:
  * existing target is neither a link nor a directory: .profile
WARNING! stowing current would cause conflicts:
  * existing target is neither a link nor a directory: .profile
All operations aborted.
```

That file is not in any layer and shale will not overwrite it. Move it aside, or copy its contents
into the layer that should own it and then delete it.

Do not reach for `stow --adopt` to clear any of these. It moves the file from `$HOME` into the
package — which here means into `~/.dotfiles/current`, generated output — and the next `shale build`
composes over it. Whether you get the content back depends on a timestamp: if the adopted file was
newer than the layer's copy the build warns and keeps it in `current.old`, and the build after that
reaps it; if it was older the build says nothing and there is nothing left to recover. Copy the file
into a layer instead.

## Links left behind after layers change

When a file leaves a layer, the next `shale apply` prunes its link. When an entire directory leaves
every layer, the links inside it are left dangling and the apply still exits 0: stow's unstow phase
only visits directories the package still has.

Find them, and delete the ones that point into `~/.dotfiles/current`:

```sh
chkstow -b -t ~
readlink ~/.config/foo/a.conf
rm ~/.config/foo/a.conf
```

`chkstow` ships with GNU stow and `-b` reports every symlink under the target whose target does not
exist, as `Bogus link: /home/you/.config/foo/a.conf`. Every one, not only stow's: it walks the whole
of `$HOME`, which includes `~/.dotfiles`, so a deliberately dangling link inside one of your own
layers and anything under `current.old` are on the list beside the links an apply orphaned. Check
each with `readlink` and delete only those whose target is under `~/.dotfiles/current`. The walk is
not quick. `stow -D` will not clear them either — it visits only the directories the package still
has.

## Uninstalling

```sh
stow -D -d ~/.dotfiles -t ~ current
```

This is also the escape hatch when an apply leaves `$HOME` in a state you would rather back out of.
It removes only the links into `current`; your layers, your config and the built tree are untouched,
and `shale apply` puts everything back. Directories stow created on the way in are left behind
empty.
