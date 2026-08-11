# Migrating to shale

There are two starting points. If your dotfiles are ordinary files sitting in `$HOME`, which is the
commoner one, go straight to *Coming from plain files*. If you already have stow packages such as
`~/.dotfiles/common` and stow them by hand, start at *Unstow first*: shale composes the same trees
into one package and stows that, so the move is mechanical — unstow what is there, turn the packages
into layers, list them, apply. Both paths meet at *What a first apply refuses, and why*, and the way
back out is the same for both.

Do it from a shell you can afford to lose. Either path takes `~/.zshrc`, `~/.profile` and everything
beside them out of `$HOME` for a while — unstowed, or moved aside — and every shell opened between
then and the apply falls back to the system defaults. Keep a second session already running, and do
this over a connection you are not relying on the dotfiles to make.

One thing to know before you start, rather than after: do not use `stow --adopt`. Your first apply
will refuse a pile of conflicts, `--adopt` is what a search turns up for clearing them, and here it
moves your file out of `$HOME` and into `~/.dotfiles/current`, which is generated output that the
next `shale build` overwrites. That build says so and keeps the whole previous tree at `current.old`,
so an adopted file survives one build and no more — the build after it reaps `current.old`. Copy the
file into a layer instead. The refusals themselves are covered below, each with what to do about it.

## Coming from plain files

`~/.zshrc`, `~/.profile`, `~/.gitconfig` and a `~/.config/nvim/init.lua`, ordinary files that nothing
manages. There is no package to unstow, so this section is the whole of the move.

Two things to know before the first command. Shale never merges: where a layer provides a path, its
file replaces yours whole, and nothing reads what is inside either copy. And shale will not overwrite
your file in order to do it — stow refuses at any path holding a real file, which is the wall further
down. Every line of reconciliation between your version and a layer's is yours to do by hand.

If **your own files are to be the layer** — you are starting shale from your dotfiles rather than
adopting anyone else's — the move is a `mv` and an apply, and there is nothing to reconcile:

```
$ mkdir -p ~/.dotfiles/local/.config
$ printf "local  local\n" > ~/.dotfiles/shale.conf
$ mv ~/.zshrc ~/.profile ~/.gitconfig ~/.dotfiles/local/
$ mv ~/.config/nvim ~/.dotfiles/local/.config/
$ shale apply
shale: composed layer 'local' from /home/you/.dotfiles/local
```

`mv` rather than `cp`: each file has to be gone from `$HOME` before the apply, or it collides with
the link stow wants to put in its place. Make `~/.dotfiles/local` a git repository whenever you want
one — to shale it is a directory either way.

If **a layer already provides those paths**, from a repository you are adopting, then your file and
the layer's are two versions of one path and exactly one of them can win. Write `shale.conf` and
create the url-less layers as the README's bootstrap has it, then apply with your files still in
place, and stow refuses the lot. On stow 2.3.1 that means every path named twice, once for the
unstow phase and once for the stow phase:

```
$ shale apply
shale: composed layer 'base' from /home/you/.dotfiles/personal/base
shale: composed layer 'local' from /home/you/.dotfiles/local
WARNING! unstowing current would cause conflicts:
  * existing target is neither a link nor a directory: .config/nvim/init.lua
  * existing target is neither a link nor a directory: .gitconfig
  * existing target is neither a link nor a directory: .profile
  * existing target is neither a link nor a directory: .zshrc
WARNING! stowing current would cause conflicts:
  * existing target is neither a link nor a directory: .config/nvim/init.lua
  * existing target is neither a link nor a directory: .gitconfig
  * existing target is neither a link nor a directory: .profile
  * existing target is neither a link nor a directory: .zshrc
All operations aborted.
shale: stow failed; nothing in /home/you was relinked
shale:   stow plans the whole operation and abandons all of it rather than half-applying one
shale:   /home/you/.dotfiles/current is built and correct: fix what stow reported above, in /home/you or in a layer, then apply again
```

Stow 2.4.1 refuses exactly the same set in half the lines: one `stowing current would cause
conflicts` block rather than two, each path named once, and each line ending `since neither a link
nor a directory and --adopt not specified`. Read the intent rather than the shape, because the shape
is what moved between the versions — and read that last clause as a suggestion from a program that
knows nothing about shale. `--adopt` moves your file into `~/.dotfiles/current`, which the next build
overwrites, for the reason at the top of this page. The five steps below are the version of that
suggestion which keeps your file.

Nothing in `$HOME` changed either way, and no file of yours was read or altered. The list is your
work queue. Take it in this order.

1. **Build, which touches nothing in `$HOME`.** `shale build` writes only `~/.dotfiles/current`, so
   it is safe to run with your files where they are, and it gives you the layers' version of each
   path to compare against.

2. **Compare, path by path.** `shale which` names the layer a version came from, and `diff` says what
   adopting it would cost you:

   ```
   $ shale which .zshrc
   winner    base   /home/you/.dotfiles/personal/base/.zshrc

   $ diff -u ~/.zshrc ~/.dotfiles/current/.zshrc
   --- /home/you/.zshrc
   +++ /home/you/.dotfiles/current/.zshrc
   @@ -1,2 +1 @@
   -export EDITOR=vim
   -alias gs="git status"
   +export EDITOR=nvim
   ```

   A path whose diff is empty needs nothing — `~/.config/nvim/init.lua` here — and goes straight in
   the pile at step 4. Everywhere else the diff is the reconciliation, and it is manual: five years
   of accumulated aliases do not come across on their own, and nothing but that diff will tell you
   they went.

3. **Move what you are keeping into a layer, and build again.** Your line goes in the layer that
   should own it — usually `local`, above the repository you are adopting — and the diff is how you
   know when you are finished:

   ```
   $ diff -u ~/.zshrc ~/.dotfiles/current/.zshrc
   --- /home/you/.zshrc
   +++ /home/you/.dotfiles/current/.zshrc
   @@ -1,2 +1,2 @@
   -export EDITOR=vim
   +export EDITOR=nvim
    alias gs="git status"
   ```

   The alias survived because it was copied into `~/.dotfiles/local/.zshrc` by hand; `EDITOR` differs
   because that difference was the point of adopting the layer.

   The build you run here reports one file in `current` older than the layer copy that replaced it,
   and its first clause names `--adopt` again. It is describing the file you just wrote, which is
   newer than the built tree it is about to replace, and it says so of every layer edit — see *The
   everyday loop* in [layers.md](layers.md). Nothing has adopted anything.

4. **Move the originals out of `$HOME`.** They are what stow named, and they only have to stop being
   files at those paths. Keep them until you have lived with the result for a week:

   ```
   $ mkdir -p ~/pre-shale/.config/nvim
   $ mv ~/.zshrc ~/.profile ~/.gitconfig ~/pre-shale/
   $ mv ~/.config/nvim/init.lua ~/pre-shale/.config/nvim/
   ```

5. **Apply.**

   ```
   $ shale apply
   shale: composed layer 'base' from /home/you/.dotfiles/personal/base
   shale: composed layer 'local' from /home/you/.dotfiles/local
   ```

   Nothing more is printed and nothing more is needed: `~/.zshrc` is now a link into the built tree.
   `~/pre-shale` stays an ordinary directory that shale neither reads nor reports.

## Unstow first

Only where the old setup is stow packages; from plain files there is nothing to unstow.

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
clean. `~/.dotfiles/local` has to exist on the next machine before any of this: a layer with no url
is one shale cannot obtain, and the build stops on it before it clones anything. With it there, that
machine clones the repository to `~/.dotfiles/personal`, finds no `base` inside it, and stops on the
config line rather than on the repository:

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
named once for each. Every transcript below is stow 2.3.1's. Match the intent rather than the shape,
because the shape is what moves between stow versions: on 2.4.1 two of these three arrive as one
block rather than two, one of them loses the `=> package/path` half of the line, and the third is
reworded into advice to use `--adopt` — see *Coming from plain files*.

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

An ordinary file, not a link, already sitting at the path — `existing target is neither a link nor a
directory`, quoted in full under *Coming from plain files*. That file is not in any layer and shale
will not overwrite it. Move it aside, or copy what you want from it into the layer that should own it
and then move it aside anyway.

Do not reach for `stow --adopt` to clear any of these, for the reason given at the top. It moves the
file from `$HOME` into the package — which here means into `~/.dotfiles/current`, generated output —
and the next `shale build` composes over it. That build reports it:

```
shale: 1 file in /home/you/.dotfiles/current was older than the layer copy that replaced it
shale:   either something moved it into the built tree, which is what stow --adopt does, or a layer changed and this is the first build since
shale:   what was in /home/you/.dotfiles/current before this build is kept at /home/you/.dotfiles/current.old
```

Both readings, because shale cannot tell them apart: an adopted file keeps its own timestamp, and so
does a `current` the layers have moved past since it was built. You will see the same message the
first time you build after pulling a layer, and it is nothing to act on there. Either way the
previous tree is at `current.old` and your file is in it, until the next clean build reaps it. Copy
the file into a layer instead and there is nothing to recover.

## Links left behind after layers change

When a file leaves a layer, the next `shale apply` prunes its link. When an entire directory leaves
every layer, the links inside it are left dangling and the apply still exits 0: stow's unstow phase
only visits directories the package still has.

`shale doctor` finds them:

```
shale: /home/you/.config/foo/a.conf points at /home/you/.dotfiles/current/.config/foo/a.conf, which the built tree no longer provides
shale: stow prunes a link only from a directory the built tree still holds, so a directory that left every layer keeps its links
shale:   remove the links named above; the built tree is correct, and nothing needs rebuilding or reapplying
shale:   stow 2.4 or later can sweep the whole target tree instead, which unstows everything and needs the apply after it:
shale:     stow -D -p --no-folding -d '/home/you/.dotfiles' -t '/home/you' current && shale apply
shale:   earlier stow, 2.3.1 included, abandons that sweep at the first file in the target that is neither a link nor a directory
```

Delete the links it names. Each one points at a path that is not there, `current` is already correct,
and nothing has to be rebuilt or reapplied afterwards.

The sweep is the alternative, and only on stow 2.4 or later. `-p` is what would make it work: it
walks the whole target tree on unstow rather than only the directories the package still has, and
plain `stow -D` leaves every orphan in place. On 2.3.1 that same walk treats every ordinary file in
the target as a conflict — one `~/notes.txt` is enough — and abandons the run before removing
anything:

```
WARNING! unstowing current would cause conflicts:
  * existing target is neither a link nor a directory: notes.txt
All operations aborted.
```

It also unstows everything else, hence the `shale apply` after it, and the walk is `$HOME`, which
`info stow` warns "can be prohibitive if your target tree is very large" — so it is a remedy rather
than what every apply does.

The audit needs `chkstow`, which ships with GNU stow; `doctor` says so and skips it if it is missing.
Running `chkstow -b -t ~` yourself lists more than `doctor` does — every broken symlink under `$HOME`
as `Bogus link: /home/you/.config/foo/a.conf`, which includes your own layers and anything under
`current.old`, and a symlink a layer ships that points at nothing is not a fault. Doctor reports only
the links outside `~/.dotfiles` that point into `~/.dotfiles/current` at a path the built tree no
longer holds. It also says when `chkstow` could not walk all of `$HOME` — a `.stow` or `.notstowed`
marker makes it skip a directory — because a walk that stopped early has nothing to say about what it
did not reach. Either way the walk is not quick.

## Backing out one apply

```sh
stow -D -d ~/.dotfiles -t ~ current
```

The escape hatch when an apply leaves `$HOME` in a state you would rather back out of. It removes
only the links into `current`; your layers, your config and the built tree are untouched, and
`shale apply` puts everything back. Directories stow created on the way in are left behind empty.

On its own it leaves you with no dotfiles at all, which is the right state only if you are about to
apply again.

## Leaving shale, with your dotfiles

Four commands, in this order. The first two are the whole of it; the last two are cleanup you can put
off:

```sh
stow -D -d ~/.dotfiles -t ~ current    # every link into current/ goes; $HOME keeps the directories
cp -RL ~/.dotfiles/current/. ~/        # the built tree lands as ordinary files, filling them again
rm ~/.stow-local-ignore                # shale's own file, copied back with everything else
rm -rf ~/.dotfiles                     # layers, config and built tree, once the repos are pushed
```

`.` in `~/.dotfiles/current/.` is what makes the copy the *contents* of the tree rather than a
`~/current` directory, and it takes the dotfiles with it. `-L` reads through a symlink a layer ships
and writes a real file, so nothing you keep points into a tree you are about to delete. The price of
`-L` is that a layer symlink pointing at nothing has nothing to read: `cp` copies everything else,
reports that one and exits 1 —

```
cp: cannot stat '/home/you/.dotfiles/current/./.danglink': No such file or directory
```

— and the fourth command would then delete the only copy of it. Deal with the named entry before you
delete anything.

What lands is exactly what the built tree holds, and shale generates one file of its own in there:
`~/.stow-local-ignore`, the third command. Run `ls -a ~/.dotfiles/current` before the copy and that
listing is what you are about to get — worth doing, because this copy pays no attention to
`.stow-local-ignore` and so brings out whatever it was suppressing. Give a layer a subdirectory of
its repository, as `docs/layers.md` says to, and that is nothing at all. Name a repository root as a
layer instead and its `README.md` and `LICENSE` have been in the built tree all along, kept out of
`$HOME` only by that ignore file; the copy puts them in `$HOME` beside your dotfiles.

Modes come across for the bit that matters: an executable stays executable, and a `~/.netrc` at 600
arrives at 600. Two smaller things `cp` does here without `-p`, which are why `-p` is deliberately
absent: a new file is created subject to your `umask`, so a 666 file in the tree arrives 644 under
the usual 022 — and a directory that already exists in `$HOME`, such as a `~/.ssh` you keep at 700,
keeps its own mode rather than being widened to the tree's. Add `-p` only if you want the tree's
modes and timestamps exactly, and know that it will chmod those existing directories.

Order matters. Copy before the unstow and every destination is still a link back into the source, so
GNU coreutils 9.4 refuses each of those and exits 1:

```
cp: '/home/you/.dotfiles/current/./.zshrc' and '/home/you/./.zshrc' are the same file
```

Nothing of yours is overwritten, but the run is not a no-op: `.stow-local-ignore` is the one entry in
the built tree stow never linked, so it has no link to collide with and it lands in `$HOME` anyway —
the file the third command exists to delete. That refusal is `cp`'s doing rather than stow's or
shale's, so treat it as a mess to avoid and not as a guard. Unstow first. Delete `~/.dotfiles` before
the copy and there is nothing left to copy from; take that last line seriously, because
`~/.dotfiles/local` is the layer that was never anywhere else.

That is the whole of shale's footprint: it writes nothing outside `~/.dotfiles`, and the script is
the copy you made yourself, at `~/.local/bin/shale` or wherever you put it. Delete that too and
shale is gone.
