# Known limits

- Rebuilding replaces `~/.dotfiles/current` in place, so there is a brief window in which the
  symlinks in `$HOME` point at nothing. Apply relinks immediately afterwards.
- No build is incremental: every one composes every layer from scratch, so a one-character edit
  costs what a first build costs. Roughly two and a half milliseconds a file — 5.0 seconds for a
  2000-file tree spread over 400 directories, 35 milliseconds for a seven-file one, on the container
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
  [migrating.md](migrating.md) covers the case in full.
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
