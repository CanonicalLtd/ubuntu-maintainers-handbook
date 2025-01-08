# Merge proposals

When you want to push changes to a package, the first step is to open a merge
proposal (MP). The merge proposal is where you discuss the reasoning for the
proposed change, and it provides reviewers the opportunity to comment on it.

After the MP is approved, if you have upload rights you can upload the package
directly via 'dput'. If you do not have upload rights, you'll work with a
"sponsor" who will do the upload for you.

## Opening a merge proposal

### Prepare a Description

The description is free-form, but should contain everything you did. Let the
reviewer know if you'll need sponsorship. You should also append the DEP-8
results.

Example SRU merge proposal:

```text
Cherry-picked existing cosmic fix from 8581dd80e48e4e9793236b178b5c9aceeb133966 in pkg/ubuntu/cosmic-devel:

      * debian/patches/fix-postconf-segfault.diff: Fix a postconf segfault
        when map file cannot be read. Thanks to Viktor Dukhovni <postfix-
        users@dukhovni.org>. (LP: #1753470)

Please tag & sponsor.

PPA: ppa:kstenerud/postfix-fix-lp1753470-postconf-segfault

Steps to test:

# lxc launch ubuntu-daily:ubuntu/bionic builder
# lxc exec builder bash

# apt dist-upgrade
# apt install -y postfix
# touch /etc/postfix/valiases.cf
# chmod 0600 /etc/postfix/valiases.cf
# echo "virtual_alias_maps = pgsql:/etc/postfix/valiases.cf" >> /etc/postfix/main.cf
# su - ubuntu
$ /usr/sbin/postconf virtual_alias_map

* This should crash.

# sudo add-apt-repository -ys ppa:kstenerud/postfix-fix-lp1753470-postconf-segfault
# sudo apt upgrade
/usr/sbin/postconf virtual_alias_map

* This should not crash.

Package Test Results:

autopkgtest [11:15:08]: test postfix: - - - - - - - - - - results - - - - - - - - - -
postfix PASS
autopkgtest [11:15:09]: @@@@@@@@@@@@@@@@@@@@ summary
postfix PASS
```

### Open the merge proposal

You'll need to have a branch set up for your package. The following steps will
create the merge proposal:

* Go to your git repositories (https://code.launchpad.net/~your-username/+git),
  which shows you a list of all your git repositories, ordered by "last updated"
* Navigate down the list to the postfix repository and click on it.
* Under "Branches", click on your branch.
* Click "Propose for merging".

  * **Target repository:** Should be correct already, e.g.
    `lp:ubuntu/+source/somepackage`.
  * **Target branch:** The release you are changing the package for, e.g.
     `ubuntu/bionic-devel`.
  * **Commit message:** (leave empty).
  * **Description:** Your merge proposal description (as per section above).
  * **Reviewer:** `canonical-<your-team>` (e.g. `canonical-server-reporter`
    for server team members).

 * Click "Propose Merge"

You'll get a merge proposal page like this:
https://code.launchpad.net/~kstenerud/ubuntu/+source/postfix/+git/postfix/+merge/353267


### The second reviewer

There are three options for the second reviewer, depending on what type of
package it is:

1. `canonical-server-motu-reviewers` for universe packages.
2. `canonical-server-packageset-reviewers` for server packages.
3. `canonical-server-core-reviewers` for core/main packages.

You can see what kind of package it is with `apt-cache policy`. For example:

```bash
$ apt-cache policy postfix
postfix:
  Installed: 3.3.0-1ubuntu0.1~ppa1
  Candidate: 3.3.0-1ubuntu0.1~ppa1
  Version table:
 *** 3.3.0-1ubuntu0.1~ppa1 500
        500 http://ppa.launchpad.net/kstenerud/postfix-postconf-segfault-1753470/ubuntu bionic/main amd64 Packages
        100 /var/lib/dpkg/status
     3.3.0-1 500
        500 http://archive.ubuntu.com/ubuntu bionic/main amd64 Packages
```

It's in main, so we won't use `canonical-server-motu-reviewers`. We can use
`ubuntu-upload-permission` to determine which of the others it belongs to:

```bash
$ ubuntu-upload-permission -a postfix
Please enter password for encrypted keyring:
All upload permissions for postfix:

Component (main)
================
* Ubuntu Core Development Team (ubuntu-core-dev) [team]

Packagesets
===========

core:

You can not upload postfix to cosmic, yourself.
But you can still contribute to it via the sponsorship process: https://wiki.ubuntu.com/SponsorshipProcess
```

It only lists core, so the second reviewer is
`canonical-server-core-reviewers`. To add the second reviewer:

* Click "Request another review" in the reviewer section.
* Type in `canonical-server-core-reviewers`.


### Get sponsorship

Before [asking your sponsor to upload](Sponsorship.md), it is wise to verify
that your proposed upload does build, does fix the issue and does not regress
anything.

If you have reason to believe that issues may manifest differently on
different architectures, then it is wise to test these as well. The reason to
test before sponsorship and upload is that the sponsorship process takes time
(and for an SRU, the additional SRU process), so if a problem is detected
later it will take longer to go through the sponsorship and SRU review
processes *again* to resolve it.

For SRUs, note that the final testing is performed during SRU verification
later, so the testing recommended at the stage of preparing an SRU is just
some very basic [smoke testing](https://en.wikipedia.org/wiki/Smoke_testing_(software))
to find likely problems. Thorough testing can be performed later to avoid
duplicating effort.

Once your MP has been reviewed, you can request sponsorship by pointing to
the git commit at the head:

```text
Please sponsor this MP. Git commit: 566d8c9eff6a13c25c2ef5f5d9e176f49c52a3b4
```

The sponsor will tag the upload and `dput` it to where it belongs.


## Retire a merge proposal

If a merge proposal should no longer land as-is, you have four options:

1. *Mark it Rejected*

   You can do this by changing the "Status" of the merge proposal from near
   the top left of the web UI. Doing this will remove it from the
   [Active Reviews page](https://code.launchpad.net/~canonical-server-reporter/+activereviews).

1. *Force push a replacement*

   If the essential topic of the change should remain, but the proposed
   changes need to be completely replaced, then you can force push to the
   source branch. This will cause all previous merge proposal comments and the
   source branch name to be retained, but to progress the merge proposal by
   proposing a replacement set of commits.

1. *Supersede the merge proposal*

   You can do this by using the "Resubmit proposal" link near the top right of
   the web UI. This allows you to create a replacement merge proposal that
   links to the one being superseded. The supserseded merge proposal will be
   marked as such. This allows you to supply a new branch name and start with
   a fresh set of comments, but without losing the previous history.

1. *Delete the merge proposal*

   You can do this by using the "Delete proposal to merge" link near the top
   right of the web UI. This should generally only be done as a last resort
   since it will lose all history, such as comments - as if the merge proposal
   had never existed.

   **Note**: This might be necessary for appropriateness or legal reasons, but
   normally we prefer to use one of the other options since retaining the
   history of what happened may be useful in the future.
