# shale

Composes directory trees (*layers*) into `~/.dotfiles/current` and hands the result to GNU Stow.
Issue #1 holds the settled decisions; `docs/` is for people authoring layers, not for people changing
shale.

## Build, run, test

`./tests/run [SUBSTRING]`. There is nothing to build.

A pass contains `N passed, 0 failed, 0 skipped`. **A skip is a failure**, and CI fails on any skip
rather than trusting the count. A case skips only for a missing or wrong-versioned `shellcheck`, a
missing `mkfifo`, or a run as root — and root is the one that costs you every case about mode bits,
silently, in the containers where it is the default. A missing `git`, `stow` or `chkstow` aborts the
run instead, before any case executes.

## The shape of the thing

`shale` is one file, and it must stay one file. Installing it is `cp shale ~/.local/bin/shale`, so a
`lib/` beside it keeps working from the repository and breaks the moment someone copies the script
somewhere else — the failure lands on the user, not on the contributor who split it.

Shale installs nothing, including itself: no distro detection, no package manager, no sudo, no
self-copy. It copies files and never interprets them — no templating, no merging, no `.append`
convention. A layer's file arrives in `$HOME` byte for byte or shale is broken.

This repository holds no dotfile content. Test fixtures are built inline by the cases that need them,
so there is no `fixtures/` tree to add one to.

## What the tests already enforce, so do not restate it

- `build`, `apply`, `doctor` and `which` dispatch, and eight other words exit 2 —
  `test_usage_command_surface_is_closed`. It rejects only the words it names, so a fifth command
  would pass; closure itself is held by review.
- Shale knows nothing of `profile.d`, `rc.d` or `helpers.sh` — `test_meta_no_fragment_conventions`.
  `docs/layers.md` teaches that convention at length to layer authors, so a contributor arrives
  believing the tool implements it. It must not, and the grep is what says so.
- The portability floor — `test_meta_no_banned_constructs` greps for the constructs off it by name.

## Where those greps do not reach

They enforce **constructs, not meanings**. Identical source behaves differently on the bash 3.2 floor
and on the container's bash 5.2, and no grep can see it — `"$dir"/.*` skips `.` and `..` under 5.2's
`globskipdots` and matches them under 3.2. The standing answer, stated once at `list_dir`, is that
every glob listing a directory goes through it and drops those two names by hand;
`test_meta_dot_globs_answer_the_same_under_both_glob_meanings` runs the script under the older
meaning on both jobs. Treating a clean meta run as portability evidence is the mistake; the macOS CI
job is what catches the rest of this class, after you push.

A new external command joins `cmd_doctor`'s tool list and the harness's `stub_path_without`, or its
absence goes unreported and nothing fails to say so. The enforcement runs the other way only: adding
to doctor's list alone reddens doctor cases, adding to neither reddens nothing.

The same trap runs outward. The container has GNU Stow 2.3.1 and the macOS runner 2.4.1; they differ
in conflict wording, in whether an unmanaged file is an unstow conflict, and in whether an absolute
layer symlink below the top level is refused. Assert version-stable fragments of stow's and chkstow's
output, never a whole line, with a comment saying why it is a fragment.

## Working on it

A branch per issue, one commit, merged when both CI jobs are green. Why the change was made goes in
the commit message — never into this file or the README.
