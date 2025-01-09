# Merging

Merging is the process of taking all Ubuntu changes made on top of one Debian
version of a package, and re-doing them on top of a new Debian version of the
package.

See [the Ubuntu wiki](https://wiki.ubuntu.com/UbuntuDevelopment/Merging/GitWorkflow)
for a more detailed workflow. This guide is intended to cover the majority of
use cases.

There is a list of packages that have been changed in Debian, but [not merged
into Ubuntu](http://reqorts.qa.ubuntu.com/reports/ubuntu-server/merges.html).

## Overview

We do merges using `git-ubuntu`. As such, the process in many ways follows that
of a git rebase, where commits from one point are replayed on top of another
point:

```text
--- something 1.2 ----------------------------- something 1.3
     \                                           \
      -- Ubuntu changes a, b, c -- 1.2ubuntu1     -- Ubuntu changes a, b, c -- 1.3ubuntu1
```

At a more detailed level, there are other subtasks to be done, such as:

* Splitting out large "omnibus" style commits into smaller logical units (one
  commit per logical unit).
* Harmonising `debian/changelog` commits into two commits: a changelog merge
  and a reconstruction.

With this process, we keep the Ubuntu version of a package cleanly applied to
the end of the latest Debian version, and make it easy to drop changes as they
become redundant.

## Process steps

- [Merging](#merging)
  - [Overview](#overview)
  - [Process steps](#process-steps)
  - [Preliminary steps](#preliminary-steps)
    - [Decide on a merge candidate](#decide-on-a-merge-candidate)
    - [Check existing bug entries](#check-existing-bug-entries)
    - [Make a bug report for the merge](#make-a-bug-report-for-the-merge)
    - [Clone the package repository](#clone-the-package-repository)
  - [The merge process](#the-merge-process)
    - [Start a Git Ubuntu merge](#start-a-git-ubuntu-merge)
    - [Make a merge branch](#make-a-merge-branch)
    - [Split commits](#split-commits)
    - [Check if there are commits to split](#check-if-there-are-commits-to-split)
      - [Identify logical changes](#identify-logical-changes)
      - [Split out logical commits](#split-out-logical-commits)
      - [Tag split](#tag-split)
      - [Purpose of logical tag](#purpose-of-logical-tag)
    - [Prepare the logical view](#prepare-the-logical-view)
      - [Check the result](#check-the-result)
      - [Create logical tag](#create-logical-tag)
    - [Rebase onto new Debian](#rebase-onto-new-debian)
      - [Conflicts](#conflicts)
      - [Corollaries](#corollaries)
      - [Empty commits](#empty-commits)
      - [Sync request](#sync-request)
      - [Check that the patches still apply cleanly](#check-that-the-patches-still-apply-cleanly)
      - [Unapply patches before continuing](#unapply-patches-before-continuing)
    - [Adding new changes](#adding-new-changes)
    - [Finish the merge](#finish-the-merge)
    - [Fix the changelog](#fix-the-changelog)
      - [Add dropped changes](#add-dropped-changes)
      - [Format any new added changes](#format-any-new-added-changes)
      - [Commit the changelog fix](#commit-the-changelog-fix)
      - [No changes to debian/changelog](#no-changes-to-debianchangelog)
      - [Tip](#tip)
  - [A brief summary of this phase (cheat sheet)](#a-brief-summary-of-this-phase-cheat-sheet)
  - [Upload a PPA](#upload-a-ppa)
    - [Get orig tarball](#get-orig-tarball)
    - [Build source package](#build-source-package)
    - [Push to your Launchpad repository](#push-to-your-launchpad-repository)
      - [Push your lp tags](#push-your-lp-tags)
    - [Create a PPA](#create-a-ppa)
      - [Create a PPA repository](#create-a-ppa-repository)
      - [Upload files](#upload-files)
      - [Wait for packages to be ready](#wait-for-packages-to-be-ready)
  - [Test the new build](#test-the-new-build)
    - [Test upgrading from the previous version](#test-upgrading-from-the-previous-version)
    - [Test installing the latest from scratch](#test-installing-the-latest-from-scratch)
    - [Other smoke tests](#other-smoke-tests)
  - [Submit Merge Proposal (MP)](#submit-merge-proposal-mp)
    - [Update the merge proposal](#update-the-merge-proposal)
    - [Open the review](#open-the-review)
  - [Follow the migration](#follow-the-migration)
    - [Package tests](#package-tests)
    - [Proposed migration](#proposed-migration)
  - [Manual steps](#manual-steps)
    - [Start a merge manually](#start-a-merge-manually)
      - [Generate the merge branch](#generate-the-merge-branch)
      - [Create tags](#create-tags)
      - [Start a rebase](#start-a-rebase)
      - [Clear any history, up to and including the last Debian version](#clear-any-history-up-to-and-including-the-last-debian-version)
      - [Create reconstruct tag](#create-reconstruct-tag)
    - [Create logical tag manually](#create-logical-tag-manually)
    - [Finish the merge manually](#finish-the-merge-manually)
    - [Get the orig tarball manually](#get-the-orig-tarball-manually)
      - [If git checkout also fails](#if-git-checkout-also-fails)
    - [Submit merge proposal manually](#submit-merge-proposal-manually)
  - [Known issues](#known-issues)
    - [Empty directories](#empty-directories)

## Preliminary steps

### Decide on a merge candidate

First, you need to check if a newer version is available from Debian. For this,
we can use the `rmadison` tool:

```bash
rmadison [package]
rmadison -u debian [package]
```

Example:

```bash
$ rmadison at
 at | 3.1.13-1ubuntu1   | precise | source, amd64, armel, armhf, i386, powerpc
 at | 3.1.14-1ubuntu1   | trusty  | source, amd64, arm64, armhf, i386, powerpc, ppc64el
 at | 3.1.18-2ubuntu1   | xenial  | source, amd64, arm64, armhf, i386, powerpc, ppc64el, s390x
 at | 3.1.20-3.1ubuntu2 | bionic  | source, amd64, arm64, armhf, i386, ppc64el, s390x
 at | 3.1.20-3.1ubuntu2 | cosmic  | source, amd64, arm64, armhf, i386, ppc64el, s390x
 at | 3.1.20-3.1ubuntu2 | disco   | source, amd64, arm64, armhf, i386, ppc64el, s390x
$ rmadison -u debian at
at         | 3.1.13-2+deb7u1 | oldoldstable       | source, amd64, armel, armhf, i386, ia64, kfreebsd-amd64, kfreebsd-i386, mips, mipsel, powerpc, s390, s390x, sparc
at         | 3.1.16-1        | oldstable          | source, amd64, arm64, armel, armhf, i386, mips, mipsel, powerpc, ppc64el, s390x
at         | 3.1.16-1        | oldstable-kfreebsd | source, kfreebsd-amd64, kfreebsd-i386
at         | 3.1.20-3        | stable             | source, amd64, arm64, armel, armhf, i386, mips, mips64el, mipsel, ppc64el, s390x
at         | 3.1.23-1        | testing            | source, amd64, arm64, armel, armhf, i386, mips, mips64el, mipsel, ppc64el, s390x
at         | 3.1.23-1        | unstable           | source, amd64, arm64, armel, armhf, hurd-i386, i386, kfreebsd-amd64, kfreebsd-i386, mips, mips64el, mipsel, ppc64el, s390x
at         | 3.1.23-1        | unstable-debug     | source
```

You'll be merging from Debian `unstable`, which in this example is `3.1.23-1`.

### Check existing bug entries

Check for any low hanging fruit in the Debian or Ubuntu bug list that can be
wrapped into this merge.

* Ubuntu bug tracker: https://bugs.launchpad.net/ubuntu/+source/[package]
* Debian bug tracker: https://tracker.debian.org/pkg/[package]

If there are bugs you'd like to fix, make a new SRU style commit at the end of
the merge process and put them together in the same merge proposal. This
process is described in the [adding new changes](#adding-new-changes) section
below.

### Make a bug report for the merge

Most of the regular server team merges are pre-planned and therefore likely
exist already at our [merges schedule](http://pinot.endarchy.org:4200/merges-schedule) and as bugs.

But we also sometimes pick up other merges from [merge-o-matic](https://merges.ubuntu.com/main.html),
our weekly [Merge Opportunities Report](https://lists.ubuntu.com/archives/ubuntu-server/2024-June/010077.html)
or just by getting aware due to any other reason.

In the case of not yet having an obvious pre-create bug search if there is
existing merge request bug entry in Launchpad.
If you don't find one, create one to avoid duplicate efforts and to allow
coordination.

To do so go to the package's Launchpad page:

https://bugs.launchpad.net/ubuntu/+source/[package]

From there, create a new bug report requesting a merge.

Example:

```text
URL: https://bugs.launchpad.net/ubuntu/+source/at/+filebug
Summary: "Please merge 3.1.23-1 into disco"
Description: "tracking bug"

result: https://bugs.launchpad.net/ubuntu/+source/at/+bug/1802914
```

Set the bug status to "in-progress" and assign it to yourself.

To let people only using m-o-m know go to the summary page
(for example https://merges.ubuntu.com/universe.html) and if the package is
listed there leave a comment linking to the bug.
This way, others not studying the LP bugs will know more easily that there is
already a bug filed for that merge. To do so go there and:
* Click in the Comment column on the invisible text entry field
* Leave a comment like "bug #123456" and press Enter
* The page will update and link to your bug
  
> **Save the bug report number, because you'll be using it throughout the merge
> process.**

### Get the package repository

If not yet present, cloning the repository is the start of all further
interaction. If instead the repository is already present, update it to
ensure to have the newest content before any further action.

#### Clone the package repository

```bash
git ubuntu clone <package> [<package>-gu]
```

Example:

```bash
$ git ubuntu clone at at-gu
```

It's a good idea to append some `git-ubuntu` specific label (like `-gu`) to
distinguish it from clones of Debian or upstream git repositories (which tend
to want to clone as the same name).

#### Update the package repository

Since this is just git, the best way to update the git ubuntu based content
as well as any other potentially added further remotes is to just update
them all before going into the merge process.

```bash
$ git fetch --all
```

## The merge process

### Start a Git Ubuntu merge

From within the git source tree:

```bash
git ubuntu merge start pkg/ubuntu/devel
```

This will generate the following tags for you:

| Tag          | Source                                                               |
| ------------ | -------------------------------------------------------------------- |
| `old/ubuntu` | ubuntu/devel                                                         |
| `old/debian` | last import tag prior to old/ubuntu without ubuntu suffix in version |
| `new/debian` | debian/sid                                                           |

If `git ubuntu merge start` fails, [do it manually](#start-a-merge-manually).

### Make a merge branch

Use the merge tracking bug and the current Ubuntu devel version it's going into
(in the example of doing a merge below, the current Ubuntu devel was `disco`
and the `merge` bug for the case was `1802914`).

```bash
$ git checkout -b merge-lp1802914-disco
```

If there's no merge bug, the Debian package version you're merging onto can be
used (for example `merge-3.1.23-1-disco`).

Sometimes you may notice a message like the following one when making the
merge branch:

```bash
$ git checkout -b merge-augeas-mirespace-testing
Switched to a new branch 'merge-augeas-mirespace-testing'

WARNING: empty directories exist but are not tracked by git:

tests/root/etc/postfix
tests/root/etc/xinetd.d/arch

These will silently disappear on commit, causing extraneous
unintended changes. See: LP: #1687057.
```

These empty directories can cause the rich history to become lost when
uploading them to the archive. Fortunately,
[a workaround exists](#empty-directories).

### Split commits

In this phase, you split out old-style commits that lumped multiple changes
together.

### Check if there are commits to split

```bash
$ git log --oneline

2af0cb7 (HEAD -> merge-3.1.20-6-disco, tag: reconstruct/3.1.20-3.1ubuntu2, tag: split/3.1.20-3.1ubuntu2) import patches-unapplied version 3.1.20-3.1ubuntu2 to ubuntu/disco-proposed
2a71755 (tag: pkg/import/3.1.20-5) Import patches-unapplied version 3.1.20-5 to debian/sid
9c3cf29 (tag: pkg/import/3.1.20-3.1) Import patches-unapplied version 3.1.20-3.1 to debian/sid
...
```

Get all commit hashes since old/debian and check the summary for what they
changed, using:

```bash
git log --stat old/debian..
```

Example: (comes from merging `heimdal` package)

```bash
git log --stat old/debian..
```

```text
commit 9fc91638b0a50392eb9f79d45d68bc5ac6cd6944 (HEAD ->
merge-7.8.git20221117.28daf24+dfsg-1-lunar)
Author: Michal Maloszewski <michal.maloszewski@canonical.com>
Date:   Tue Jan 17 16:16:01 2023 +0100

    Changelog for 7.8.git20221117.28daf24+dfsg-1

 debian/changelog | 1 -
 1 file changed, 1 deletion(-)



commit e217fae2dc54a0a13e4ac5397ec7d3be527fa243
Author: Michal Maloszewski <michal.maloszewski@canonical.com>
Date:   Tue Jan 17 16:13:49 2023 +0100

    update-maintainer

 debian/control | 3 ++-
 1 file changed, 2 insertions(+), 1 deletion(-)



commit 3c66d873330dd594d593d21870f4700b5e7fd153
Author: Michal Maloszewski <michal.maloszewski@canonical.com>
Date:   Tue Jan 17 16:13:49 2023 +0100

    reconstruct-changelog

 debian/changelog | 10 ++++++++++
 1 file changed, 10 insertions(+)



commit 58b895f5ff6333b1a0956dd83e478542dc7a10d3
Author: Michal Maloszewski <michal.maloszewski@canonical.com>
Date:   Tue Jan 17 16:13:46 2023 +0100

    merge-changelogs

 debian/changelog | 68
 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 1 file changed, 68 insertions(+)
```

You can see that this command shows us the specific commit, as well as what
was changed within the commit (i.e., how many files were changed and how many
insertions and deletions are there).

If you see `changelog` with any other file(s) changing in a single commit, it's
guaranteed that you'll need to split it. `debian/changelog` should only ever be
changed in commits on its own, without touching any other file. You should
still look over all commits just to make sure.

Another giveaway would be a commit named:
`Import patches-unapplied version 1.2.3ubuntu4 to ubuntu/cosmic-proposed`,
where it's applying from an Ubuntu source rather than a Debian one (in this
case `ubuntu4`).

If there are no commits to split, simply [add the `split` tag](#tag-split) and
[move onto preparing the logical view](#prepare-the-logical-view).

#### Identify logical changes

The next step is to separate the changes into logical units. For the `at`
package, this is trivial: just put the changelog change in one commit, and
the control change in the other.

The second example, for `nspr`, is more instructive. Here we have 5 files
changed, that need to be split out:

* All changelog changes go to one commit called `changelog`.
* Update maintainer (in debian/control) goes to one commit called
  `update maintainers`.
* All other logically separable commits go into individual commits.

Look in `debian/changelog`:

```text
nspr (2:4.18-1ubuntu1) bionic; urgency=medium

  * Resynchronize with Debian, remaining changes
    - rules: Enable Thumb2 build on armel, armhf.
    - d/p/fix_test_errcodes_for_runpath.patch: Fix testcases to handle
      zesty linker default changing to --enable-new-dtags for -rpath.
```

There are two logical changes, which we'll need to separate. Look at the
changes in individual files to see which file changes should be logically
grouped together.

Example:

```bash
$ git show d7ebe661 -- debian/rules
```

In this case, we have the following file changes to separate into logical units:

| File(s)            | Logical unit                                 |
| ------------------ | -------------------------------------------- |
| `debian/rules`     | Enable Thumb2 build on armel, armhf.         |
| `debian/patches/*` | Fix testcases to handle zesty linker default changing to --enable-new-dtags for -rpath.   |
| `debian/control`   | Change maintainer                            |
| `debian/changelog` | Changelog                                    |

#### Split out logical commits

Start a rebase at `old/debian`, and then reset to `HEAD^` to bring back the
changes as uncommitted changes.

1. Start a rebase: `git rebase -i old/debian`
1. Change the commit(s) you're going to split from `pick` to `edit`.
1. Do a `git reset` to get your changes back: `git reset HEAD^`

Next, add the commits:

------------------------------------------------------------------------------

Logical unit:

```bash
$ git add debian/patches/*
$ git commit
```

Commit Message:

```text
  * d/p/fix_test_errcodes_for_runpath.patch: Fix testcases to handle
    zesty linker default changing to --enable-new-dtags for -rpath.
```

------------------------------------------------------------------------------

Logical unit:

```bash
$ git add debian/rules
$ git commit
```

Commit Message:

```text
  * d/rules: Enable Thumb2 build on armel, armhf.
```

------------------------------------------------------------------------------

Maintainers:

```bash
$ git commit -m "update maintainers" debian/control
```

------------------------------------------------------------------------------

Changelog:

```bash
$ git commit -m changelog debian/changelog
```

------------------------------------------------------------------------------

Finally, complete the rebase:

```bash
$ git rebase --continue
```

The result of this rebase should be a sequence of smaller commits, one per
`debian/changelog` entry (with potentially additional commits for previously
undocumented changes).

It should represent a broken-out history (viewable with `git-log`) for the
latest Ubuntu version and no content differences to that Ubuntu version. This
can be verified with `git diff -p old/ubuntu`.

#### Tag split

Note: Do this even if there were no commits to split.

```bash
$ git ubuntu tag --split
```

#### Purpose of logical tag

- If we do this step, then we will have a distinct boundary between looking at
  the past (analysis of what is there already) and the actual work we want to
  perform (bringing the old Ubuntu delta forward).
- By having a well-defined point, it is easier to do the future work, and also
  for a reviewer to start from the same point. The reviewer can very easily and
  almost completely automatically determine if the logical tag is correct.
- The logical tag is the cleanest possible representation of a previous Ubuntu
  delta. By determining this representation, we make it as easy as possible to
  bring the delta forward.

### Prepare the logical view

In this phase, we make a clean, "logical" view of the history. This history is
cleaned up (but has the same delta), and only contains the actual changes that
affect the package's behaviour.

We first start with a rebase from `old/debian`:

```bash
$ git rebase -i old/debian
```

Now we do some cleaning:

* Delete imports, etc.
* Delete any commit that only changes metadata like changelog, maintainer.
* Possibly rearrange commits if it makes logical sense.

You should also squash these kinds of commits together:

 * Changes and reversions of those changes, since they resolve to a No-Op.
 * Multiple changes to the same patch file, since they should be a logical unit.

To squash a commit, move its line underneath the one you want it to become part
of, and then change it from `pick` to `fixup`.

#### Check the result

At the end of the "squash and clean" phase, the only delta you should see from
the split tag is:

```bash
$ git diff --stat split/6.8-0ubuntu2 
 debian/changelog | 31 -------------------------------
 debian/control   |  3 +--
 2 files changed, 1 insertion(+), 33 deletions(-)
```

Only changelog and control were changed, which is what we want.

#### Create logical tag

What is the logical tag? It is a representation of the Ubuntu delta present
against a specific historical package version in Ubuntu.

```bash
$ git ubuntu tag --logical
```

This may fail with an error like: `ERROR:HEAD is not a defined object in this
git repository.`, in which case [do it manually](#create-logical-tag-manually).

### Rebase onto new Debian

```bash
$ git rebase -i --onto new/debian old/debian
```

#### Conflicts

If a conflict occurs, you must resolve it. We do so by modifying the
conflicting commit during the rebase.

An example, merging `logwatch 7.5.0-1`:

```bash
$ git rebase -i --onto new/debian old/debian
...
CONFLICT (content): Merge conflict in debian/control
error: could not apply c0efd06... - Drop libsys-cpu-perl and libsys-meminfo-perl from Recommends to
...
```

Take a look at the conflict in `debian/control`:

```text
    <<<<<<< HEAD
    Recommends: libdate-manip-perl, libsys-cpu-perl, libsys-meminfo-perl
    =======
    Recommends: libdate-manip-perl
    Suggests: fortune-mod, libsys-cpu-perl, libsys-meminfo-perl
    >>>>>>> c0efd06... - Drop libsys-cpu-perl and libsys-meminfo-perl from Recommends to
```

Upstream removed `fortune-mod`, and deleted the entire line since it was no
longer needed. Resolve it to:

```text
Recommends: libdate-manip-perl
Suggests: libsys-cpu-perl, libsys-meminfo-perl
```

Continue with the rebase:

```bash
$ git add debian/control
$ git rebase --continue
```

#### Corollaries

Mistake corrections are squashed.

Changes that fix mistakes made previous in the same delta are squashed against
them. For example:

* `2.3-4ubuntu1` was the previous merge.
* `2.3-4ubuntu2` adjusted `debian/rules` to add a configure flag
  `--build-beter`
* `2.3-4ubuntu3` fixed the typo in `debian/rules` to say `--build-better`
  instead.
* When the logical tag is created, there will be only one commit relating to 
  `--build-better`, which omits any mention of the typo.

> **Note**:
> 
> If a mistake exists in the delta itself, then it is retained.
> For example, if `2.3-4ubuntu3` was never uploaded and the typo is still
> present in `2.3-4ubuntu2`, then `logical/2.3.-4ubuntu2` should contain a
> commit adding the configure flag with the typo still present.

#### Empty commits

If a commit becomes empty, it's because the change has already been applied
upstream:

```text
The previous cherry-pick is now empty, possibly due to conflict resolution.
```

In such a case, the commit can be dropped.

```bash
$ git rebase --abort
$ git rebase -i old/debian
```

Keep a copy of the redundant commit's commit message, then delete it in the
rebase.

#### Sync request

If all the commits are empty, or you realised there are no logical changes,
you're facing a **sync request**, not a merge. Refer to the
[sync guidelines](Syncs.md) to continue.

#### Check that the patches still apply cleanly

```bash
$ quilt push -a --fuzz=0
```

**If quilt fails**:

Quilt can fail at this point if the file being patched has changed
significantly upstream. The most common reason is that the issue the patch
addresses has since been fixed upstream.

For example:

```bash
$ quilt push -a --fuzz=0
...
Applying patch ssh-ignore-disconnected.patch
patching file scripts/services/sshd
Hunk #1 FAILED at 297.
1 out of 1 hunk FAILED -- rejects in file scripts/services/sshd
Patch ssh-ignore-disconnected.patch does not apply (enforce with -f)
```

If this patch fails because the changes in `ssh-ignore-disconnected.patch` are
already applied upstream, you must remove this patch.

```bash
$ git log --oneline

1aed93f (HEAD -> ubuntu/devel)   * d/p/ssh-ignore-disconnected.patch: [sshd] ignore disconnected from user     USER (LP: 1644057)
7d9d752 - Drop libsys-cpu-perl and libsys-meminfo-perl from Recommends to   Suggests as they are in universe.
```

Removing `1aed93f` will remove the patch.

* Save the commit message from `1aed93f` for later, including in the `Drop
  Changes` section of the new changelog entry.
* `git rebase -i 7d9d752` and delete commit `1aed93f`.

#### Unapply patches before continuing

```bash
$ quilt pop -a
```

### Adding new changes

Add any new changes you want to include with the merge. For instance, the new
package version may fail to build from source (FTBFS) in Ubuntu due to new
versions of specific libraries or runtimes.

Each logical change should be in its own commit to match the work done up to
this point on splitting the logical changes.

Moreover, there is no need to add changelog entries for these changes manually.
They will be generated from the commit messages with the merge finish process
described below.

### Finish the merge

```bash
$ git ubuntu merge finish pkg/ubuntu/devel
```

If this fails, [do it manually](#finish-the-merge-manually).

### Fix the changelog

Git Ubuntu attempts to put together a changelog entry, but it will likely have
problems. Fix it up to make sure it follows the standards. See
[committing your changes](CommittingChanges.md) for information about what it
should look like.

#### Add dropped changes

If you dropped any changes (due to upstream fixes), you must note them in the
changelog entry:

```text
  * Drop Changes:
    - Foo: change to bar
      [Fixed in 1.2.3-4]
```

#### Format any new added changes

If you added any new changes, they should be in their own section in the
changelog:

```text
  * New Changes:
    - Bar: change to foo
    - Baz: adjust for Foo changes
```

#### Commit the changelog fix

```bash
$ git commit debian/changelog -m changelog
```

#### No changes to debian/changelog

The range `old/ubuntu..logical/<version>` should contain no changes to
`debian/changelog` at all. We do not consider this part of the logical delta.
So, any commits that contain only changes to `debian/changelog` should be
dropped.

#### Tip

If you "diff" your final logical tag against the Ubuntu package it analyses,
then the diff should be empty, except:

1. All changes to `debian/changelog`:

   We deliberately exclude these from the logical tag, relying on commit
   messages instead.

1. The change that `update-maintainer` introduced, and (rarely) similar
   changes like a change to `Vcs-Git` headers to point to an Ubuntu VCS
   instead.
   
   For the purposes of this workflow, these are not considered part of our
   “logical delta”, and instead are re-added at the end.

#### Tip

You can use `execsnoop-bpfcc` from the package `bpfcc` to find what debhelper scripts were called for a certain package. This is helpful for debugging what scripts were called, and what parameters were passed to them.

For example,

    $ sudo apt install bpfcc-tools
    $ sudo execsnoop-bpfcc -n multipath

Now in another shell run

    $ sudo apt install --reinstall multipath-tools

In the original shell you should see something like

    PCOMM            PID     PPID    RET ARGS
    multipath-tools  13939   13931     0 /var/lib/dpkg/info/multipath-tools.prerm upgrade 0.9.4-5ubuntu3
    multipath-tools  13951   13931     0 /var/lib/dpkg/info/multipath-tools.postrm upgrade 0.9.4-5ubuntu3
    multipath-tools  13959   13956     0 /var/lib/dpkg/info/multipath-tools.postinst configure 0.9.4-5ubuntu3
    multipathd       14009   1         0 /sbin/multipathd -d -s


## A brief summary of this phase (cheat sheet)

1. `rmadison <package_name>`
1. `rmadison -u debian <package_name>`
1. `git ubuntu clone <package_name> <package_name>-gu`
1. `cd <package_name>-gu`
1. `git ubuntu merge start pkg/ubuntu/devel`
1. `git checkout -b
merge-<version_of_debian_unstable>-<current_ubuntu_devel_name>`
1. `git log --stat old/debian..`
1. `git ubuntu tag --split` -> if nothing to split, type that command
straight away
1. `git rebase -i old/debian`
1. Drop metadata changes and reorder/merge/split commits.
1. `git diff split/`
1. `git ubuntu tag --logical`
1. `git show logical/<version>` -> check if the new tag exists
1. `git rebase -i --onto new/debian old/debian`
1. `quilt push -a --fuzz=0`
1. `quilt pop -a`
1. `git ubuntu merge finish pkg/ubuntu/devel`

## Upload a PPA

### Get orig tarball

Ubuntu doesn't know about the new tarball yet, so we must create it.

```bash
$ git ubuntu export-orig
```

If the upstream version does not yet exist in Ubuntu, that is, the new package from Debian also includes a new upstream version, you should add the `--for-merge` option:

```bash
$ git ubuntu export-orig --for-merge
```

If this fails, [do it manually](#get-the-orig-tarball-manually).

### Build source package

```bash
$ dpkg-buildpackage \
 --build=source \
 --no-pre-clean \
 --no-check-builddeps \
 -sa \
 -v3.1.20-3.1ubuntu2
```

The switches are:

* `-sa` = include orig tarball (required on a merge)
* `-vXYZ` = include changelog since XYZ

Since our merge upload represents all changes that happened in Debian since
our last merge plus anything added as part of the merge itself `-v` should
usually point to the last published Ubuntu version. Here's an example:

* Ubuntu merged as `1.3-1ubuntu1`
* Then Ubuntu had a fix in `1.3-1ubuntu2`
* But in the meantime Debian merged upstream as `1.4-1`
* And then added a fix in `1.4-2`
* New Ubuntu will be `1.4-2ubuntu1`
* `-v` should here be set to `1.3-1ubuntu2`
  * Thereby the .changes file will include `1.4-1`, `1.4-2`, and `1.4-2ubuntu1`
  * That represents all the changes that happened from the perspective of an Ubuntu user upgrading from `1.3-1ubuntu2 to `1.4-2ubuntu1`

If sponsoring a merge or any other upload for someone else, please remember the
need to sign their upload with your key. See [Sponsor a package](Sponsorship.md#sponsor-a-package)
for more information about that.
Furthermore just like you the sponsor needs to know about e.g. setting `-v`
right and using `-sa` when needed. If in doubt, coordinating with them will
be helpful.

### Push to your Launchpad repository

Now that the package is tested and builds successfully, it's time to push it
to your Launchpad repository.

The easiest way is to run it like this:

```bash
git push your-lp-username
```

You'll get an error message and a suggestion for how to set upstream. For
example:

```bash
$ git push kstenerud
fatal: The current branch merge-lp1802914-disco has no upstream branch.
To push the current branch and set the remote as upstream, use

    git push --set-upstream kstenerud merge-lp1802914-disco
```

Run the suggested command to push to your repository.

#### Push your lp tags

```bash
$ git push <your-git-remote> old/ubuntu old/debian new/debian reconstruct/<version> logical/<version> split/<version>

To ssh://git.launchpad.net/~kstenerud/ubuntu/+source/at
 * [new tag]         split/3.1.20-3.1ubuntu2 -> split/3.1.20-3.1ubuntu2
 * [new tag]         logical/3.1.20-3.1ubuntu2 -> logical/3.1.20-3.1ubuntu2
 * [new tag]         new/debian -> new/debian
 * [new tag]         old/debian -> old/debian
 * [new tag]         old/ubuntu -> old/ubuntu
 * [new tag]         reconstruct/3.1.20-3.1ubuntu2 -> reconstruct/3.1.20-3.1ubuntu2
```

### Create a PPA

You'll need to have a PPA for reviewers to test.

#### Create a PPA repository

https://launchpad.net/~your-username/+activate-ppa

Give it a name that identifies the Ubuntu version, package name, and bug
number, such as `at-merge-lp1802914`.

> **IMPORTANT:**
> 
> Be sure to enable all architectures to check that it builds (click on
> `Change details` in the top right corner of the newly created PPA page).

#### Upload files

```bash
$ dput ppa:kstenerud/at-merge-lp1802914 ../at_3.1.23-1ubuntu1_source.changes
```

#### Wait for packages to be ready

Check the PPA page to see when packages are finished building:
https://launchpad.net/~kstenerud/+archive/ubuntu/disco-at-merge-1802914

Also, look at the package contents to make sure they have actually been
published:
https://launchpad.net/~kstenerud/+archive/ubuntu/disco-at-merge-1802914/+packages

## Test the new build

Test the following:

1. [Run package tests (if any)](PackageTests.md)
1. [Upgrading from the previous version](#test-upgrading-from-the-previous-version)
1. [Installing the latest where nothing was installed before](#test-installing-the-latest-from-scratch)
1. [Other smoke tests](#other-smoke-tests)

### Test upgrading from the previous version

Example:

```bash
$ lxc launch ubuntu-daily:ubuntu/cosmic tester && lxc exec tester bash
$ apt update && apt dist-upgrade -y && apt install -y at
```

> **Note**: Disco is not yet available at the time of writing, so we use Cosmic.

The test:

```bash
echo "echo xyz >test.txt" |at now + 1 minute && sleep 1m && cat test.txt && rm test.txt
```

Upgrade:

```bash
$ add-apt-repository -y ppa:kstenerud/at-merge-lp1802914
```

> **Note**: Disco is not yet available at the time of writing, so we must
> modify the source list entry:

```bash
$ vi /etc/apt/sources.list.d/kstenerud-ubuntu-at-merge-lp1802914-cosmic.list
* change cosmic to disco

$ apt update && apt dist-upgrade -y
```

Test the upgraded version:

```bash
$ echo "echo abc >test.txt" | at now + 1 minute && sleep 1m && cat test.txt && rm test.txt
```

### Test installing the latest from scratch

```bash
$ lxc launch ubuntu-daily:ubuntu/cosmic tester && lxc exec tester bash
$ add-apt-repository -y ppa:kstenerud/at-merge-lp1802914
$ apt update && apt dist-upgrade -y && apt install at
$ echo "echo abc >test.txt" | at now + 1 minute && sleep 1m && cat test.txt && rm test.txt
```

### Other smoke tests

* Try running various basic commands.
* Try running regression tests: https://git.launchpad.net/qa-regression-testing
* For packages that Build-Depend on itself (openjdk, jruby, kotlin, etc.) try to build it using the new version.

## Submit Merge Proposal (MP)

> **NOTE**:
> 
> Git branches with `%` in name don't work. Use something like `_`.

```bash
$ git ubuntu submit --reviewer $REVIEWER --target-branch debian/sid
Your merge proposal is now available at: https://code.launchpad.net/~kstenerud/ubuntu/+source/at/+git/at/+merge/358655
If it looks OK, please move it to the 'Needs Review' state.
```

Where `$REVIEWER` is the team (or user) on Launchpad that you need to look at
your change, for instance `canonical-foundations`, `canonical-public-cloud`, or
similar.

**If you do not have upload rights for this package, use `ubuntu-sponsors`
here.** This will add your Merge Proposal to the [Ubuntu sponsoring
queue](http://sponsoring-reports.ubuntu.com/general.html) so people with upload
rights for that package may eventually review it for you.

Omitting the `--reviewer` field will request a review from the Ubuntu Server
team. If you are unsure if your package should be reviewed by the Server team,
please send it to the sponsoring queue instead (as described above).

* Using a target branch of `debian/sid` may seem wrong, but is a workaround
  for LP: #1976112

If this fails, [do it manually](#submit-merge-proposal-manually).

### Update the merge proposal

 * Link the PPA
 * Add any other info (as a comment) that will help the reviewer to MP.

Example:

> PPA: https://launchpad.net/~kstenerud/+archive/ubuntu/disco-at-merge-1802914

Basic test:

> `$ echo "echo abc >test.txt" | at now + 1 minute && sleep 1m && cat test.txt && rm test.txt`

Package tests:

> This package contains no tests.


### Open the review

Change the MP status from "work in progress" to "needs review".

## Follow the migration

Once the merge proposal goes through, you must follow the package to make sure
it gets to its destination.

### Package tests

The results from the latest package tests will be published for each Ubuntu
release.

For example: http://autopkgtest.ubuntu.com/packages/o/openssh/focal/amd64

### Proposed migration

The status of all packages will be available from the
[Ubuntu archive](http://people.canonical.com/~ubuntu-archive/proposed-migration)
or one of its subdirectories. The top level directory is for the current dev
release. Previous releases are in subdirectories.

-----------------------------------------------------------------------------

## Manual steps

### Start a merge manually

#### Generate the merge branch

Create a branch to do the merge work in:

```bash
$ git checkout -b merge-lp1802914-disco
```

#### Create tags

| Tag          | Source             |
| ------------ | ------------------ |
| `old/ubuntu` | ubuntu/disco-devel |
| `old/debian` | last import tag prior to old/ubuntu without ubuntu suffix in version    |
| `new/debian` | debian/sid         |

As per [Debian releases](https://www.debian.org/releases/), `debian/sid` always
matches to Debian "unstable".

You can find the last import tag using
`git log | grep "tag: pkg/import" | grep -v ubuntu | head -1`:

```text
...
commit 9c3cf29c05c3fddd7359e71c978ff9a9a76e4404 (tag: pkg/import/3.1.20-3.1)
```

So, we create the following tags:

```bash
$ git tag old/ubuntu pkg/ubuntu/disco-devel
$ git tag old/debian 9c3cf29c05c3fddd7359e71c978ff9a9a76e4404
$ git tag new/debian pkg/debian/sid
```

#### Start a rebase

```bash
$ git rebase -i old/debian
```

#### Clear any history, up to and including the last Debian version

If the package hasn't been updated since the git repository structure changed,
it will grab all changes throughout time rather than since the last Debian
version. Just delete the older lines from the interactive rebase.

In this case, up to, and including import of `3.1.20-3.1`.

#### Create reconstruct tag

```bash
$ git ubuntu tag --reconstruct
```

Next step: [Split commits](#split-commits)

### Create logical tag manually

Use the version number of the last ubuntu change. So if there are
`3.1.20-3.1ubuntu1` and `3.1.20-3.1ubuntu2`, use `3.1.20-3.1ubuntu2`.

```bash
$ git tag -a -m "Logical delta of 3.1.20-3.1ubuntu2" logical/3.1.20-3.1ubuntu2
```

> **Note**:
> 
> Certain characters aren't allowed in git. For example, `:` should be
> replaced with `%`.

Next step: [Rebase onto new Debian](#rebase-onto-new-debian)

### Finish the merge manually

Merge the changelogs of old Ubuntu and new Debian:

```bash
$ git show new/debian:debian/changelog >/tmp/debnew.txt
$ git show old/ubuntu:debian/changelog >/tmp/ubuntuold.txt
$ merge-changelog /tmp/debnew.txt /tmp/ubuntuold.txt >debian/changelog
$ git commit -m "Merge changelogs" debian/changelog
```

Create a new changelog entry for the merge:

```bash
$ dch -i
```

Which creates, for example:

```text
at (3.1.23-1ubuntu1) disco; urgency=medium

  * Merge with Debian unstable (LP: #1802914). Remaining changes:
    - Suggest an MTA rather than Recommending one.

 -- Karl Stenerud <karl.stenerud@canonical.com>  Mon, 12 Nov 2018 18:11:53 +0100
```

Commit the changelog:

```bash
$ git commit -m "changelog: Merge of 3.1.23-1" debian/changelog
```

Update maintainer:

```bash
$ update-maintainer
$ git commit -m "Update maintainer" debian/control
```

Next step: [Fix the changelog](#fix-the-changelog)

### Get the orig tarball manually

```bash
$ git checkout -b pkg/importer/debian/pristine-tar
$ pristine-tar checkout at_3.1.23.orig.tar.gz
$ git checkout merge-3.1.23-1-disco
```

#### If git checkout also fails

```bash
$ git checkout merge-lp1802914-disco
$ cd /tmp
$ pull-debian-source at
$ mv at_3.1.23.orig.tar.gz{,.asc} ~/work/packages/ubuntu/
$ cd -
```

Next step: [Check the source for errors](#check-the-source-for-errors)

### Submit merge proposal manually

```bash
$ git push kstenerud merge-lp1802914-disco
```

Then, create a MP manually in Launchpad, and save the URL.

Next step: [Update the merge proposal](#update-the-merge-proposal)

## Known issues

### Empty directories

We need to use a
[python script](https://git.launchpad.net/~racb/usd-importer/plain/wip/emptydirfixup.py?h=emptydirfixup)
written by Robie Basak (@racb). Why is it a problem that we get empty
directories?

Git's frontend doesn't let you add an empty directory. Usually the workaround
is to create any necessary empty directory at build time, or failing that, to
create a placeholder file like `.empty` and check that in.

Neither of these approaches work for git-ubuntu's importer in the general case.
A source package can ship an empty directory by nature of the source package
format. But the build system (i.e. `debian/rules`) in the source package
expects the source exactly as packed. Just as some builds break if empty
directories are missing, other builds might break if empty directories are not
actually empty.

Internally, git supports empty directories just fine. Directories map to git
tree objects. An empty tree object is the obvious way of representing an empty
directory, and git seems to accept them if they are represented this way. It's
just the git index and frontend that do not support them.

In `git-ubuntu`, we therefore import empty directories "correctly" and
losslessly by using empty tree objects as necessary. However, when such a tree
is checked out at the client end, the empty directories disappear as they pass
through the index, and get lost. A subsequent commit made by a developer then
gets created from the index, so does not include the empty directories even if
they haven't been touched.

This becomes an issue if such a commit is subsequently presented back to
`git-ubuntu` as rich history to be adopted against an upload. `git-ubuntu`
finds that the upload (with empty directories) doesn't match the rich history
commit (with missing empty directories).

This script restores the empty directories locally as a workaround. It takes a
non-merge commit and examines its parent to discover which empty directories
have been lost. It provides an equivalent replacement commit.

Run it with `fix-head` to replace HEAD with a commit that has empty directories
restored.

Run it with `fix-many` and a parameter pointing to a base commit to run
`git-rebase` to fix a set of commits.

Note that in both cases, the parent must have the empty directories in order
for them to be copied down through the fixed-up commits. In the common case
where this script is needed, you'll be starting from an "official" `git-ubuntu`
import tag or branch, so this will be true in these cases. However, this does
mean that you need to use `fix-many` all the way back to the first commit after
such an "official" commit. If you have intermediary un-fixed commits and then
try to apply `fix-head` to the end, then it won't work as the empty directories
won't get copied forward.

Example of use:

```bash
git ubuntu clone apache2
cd apache2
git tag -f base
<add commits>
python3 emptydirfixup.py fix-many base
git ubuntu tag --upload
```

For an entire real case you can follow this workflow:

```bash
# get emptydirfxup script
wget -O ../emptydirfixup.py "https://git.launchpad.net/~racb/usd-importer/plain/wip/emptydirfixup.py?h=emptydirfixup"

# clone as usual
git ubuntu clone "${source_package}" "${source_package}-gu"
cd "${source_package}-gu/"

# make the merge branch (here you see the warning message)
git checkout "${last_remote}" -b "${branch_name}"

# tag the base and rebase on ubuntu/devel
git tag -f base
git checkout ubuntu/devel
git rebase base

# start the merge
git ubuntu merge -f start

#... Merge work as usual ...

# Workaround LP: #1939747
rm .git/hooks/pre-commit

# finish the merge
git ubuntu merge finish pkg/ubuntu/devel

#... Create MP as usual, get reviewed/approved, etc. ...

# Fix the empty dir set of commits
python3 ../emptydirfixup.py fix-many base

# Build the package
debuild -S $(git ubuntu push-for-upload)

# Uploading
git push pkg "upload/${version}"
dput ubuntu "${changes_file}"

#... Done! ...
```
