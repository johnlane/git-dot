---
layout: page
---
Git-Dot - Git for *dotfiles*
============================

**git-dot** lets you easily maintain your *dotfiles*<super>*</super> in a Git repository. It integrates
into your workflow as a new `git dot` command and works with [git-crypt] so that files
with sensitive contents can be encrypted.

<small><super>*</super>and *dot directories* too!

With **git-dot** you can safely store sensitive material in your repsitory such as
passwords, API tokens, SSH and OpenPGP private keys alongside other files that don't
require such protection.

<div class="masthead" style="border:0.2em solid red; text-align:center">
This is <a href="https://en.wikipedia.org/wiki/Software_release_life_cycle#Alpha">
alpha-grade</a> software that has undergone limited testing on Linux only!
<span style="font-size:75%">(feedback welcome; see end of page)</span>
</div>

There are [many dotfiles projects](https://dotfiles.github.io) but the objective of
this one is simplicity. In particular, `git dot`

* works directly off your dotfiles in `$HOME`;
* does not use symbolic links;
* uses [git-crypt] to offer a simple method of encyption;
* has no clever host-specific provisions;
* is a `bash` shell script that works on Linux.

Download and install
--------------------

Make sure you have [git-crypt] installed and working. Then download `git-dot`
to a location on your `$PATH` and make it executable, for example:

    $ curl -Lo git-dot https://git.io/vDJM9
    $ chmod +x git-dot
    $ sudo mv git-dot /usr/bin

<small>(or `git clone https://github.com/johnlane/git-dot`)

Usage
-----

Familiarise yourself with the available commands:

![git dot help](/assets/git-dot-help.png)

Any unrecognised command is passed to Git, allowing any `git` or `git crypt` command
to be used as `git dot` or `git dot crypt`.

### Create a repo

To create a new repo:

    $ git dot init
    Initialized empty Git repository in /home/alice/.git-dot/
    Generating key...

<small>
The Git directory used by `git dot` is `$HOME/.git-dot` so that it doesn't conflict with any other `.git` that you might have. It also creates `$HOME/.gitattributes` file where the files to be encrypted are specified (`git dot` manages this file for you).
</small>

#### Export the key

If you lock the repository then you will need its key to unlock it again. Export the
key and keep it somewhere safe (outside the repository):

    $ git dot crypt export-key repo-key

You can use a GnuPG key to unlock the repository once you have registered it:

    $ git dot crypt add-gpg-user alice@example.com

<small>
You will be unable to unlock the repository if the key that you require is inside it.
Read *I've locked my keys in my car* below and ensure you have a means of access to the
repository's key without needing to unlock the repository!

<small>
The [git-crypt] documentation contains further information about keys.

### Clone a repo

To clone an existing dotfiles repo into your `$HOME`:

    $ git dot clone http://example.com/dotfiles.git
    Cloning into bare repository '/home/alice/.git-dot'...

If your `$HOME` already contains dotfiles that the repo would encrypt then these are
moved into a backup directory and replaced with the encrypted versions from the repo.
![clone encrypted message](/assets/clone-encrypted.png)
<small>
You should first unlock the repository (but read below first!) and then compare these
files with those now in your `$HOME`, taking appropriate action as required to handle
any differences.

If your `$HOME` already contains dotfiles that the repo would not encrypt then these
are left in place. Git will see these as modified files and `git dot` will report that
your working directory (`$HOME`) is unclean.
![clone stash message](/assets/clone-stash.png)
<small>
You can only unlock a repo with a clean working directory so you will need to
temporarily stash these files as described next.

#### Stash and unlock

To unlock the repository when the working directory is unclean:

    $ git dot stash
    Saved working directory and index state WIP on master: ea9d3af 
    HEAD is now at ea9d3af
    $ git dot unlock repo-key
    $ git dot stash pop
    Dropped refs/stash@{0} (e71903c20c05f104065bacee43aa1c0a7dbe85f3)

<small>
The repository key is given in the file `repo-key`. You must have your repository's
key in order to unlock it. Read *I've locked my keys in my car*, below, if your
repository's key is locked inside it!


#### Comparing backup files

Afer unlocking the repository, you should compare any backups of files that were
replaced with checked out encrypted versions. You could try

    $ diff -qr git-dot-backup.VpjcrUMS.tmp . | grep -v '^Only in .'
    Files git-dot-backup.VpjcrUMS.tmp/.ssh/id_rsa and ./.ssh/id_rsa differ

or

    $ (cd git-dot-backup.VpjcrUMS.tmp; find . -type f -exec diff -q {} ~/{} \;)
    Files ./.ssh/id_rsa and /home/alice/./.ssh/id_rsa differ

#### Permissions

Apart from identiying executables, Git does not record file permissions and files
checked out from a repository may become world readable. You can use `git dot`
to secure permissions of the files in `$HOME` that are encrypted within the
repository:

    $ git dot protect

#### Missing remote branch

After using `git dot clone`, the output of `git dot status` may show:

    Your branch is based on 'origin/master', but the upstream is gone.

This means that the remote branch is configured but not fetched. To rectify:

    $ git dot fetch origin

which should result in `git dot status` reporting

    Your branch is up-to-date with 'origin/master'

### Encrypting files

To encrypt a file, first (*before committing the file*) tell `git dot` that
it should encrypt it:

    $ git dot encrypt .mysecrets
    [master 44e6c62] Specify 1 encrypted file   .mysecrets
     1 file changed, 1 insertion(+)

This adds and commits an entry to `.gitattributes`. You must then add
and commit the file:

    $ git dot add .mysecrets
    $ git dot commit -m "Hush, don't tell anyone!"
    [master 9efecd4] Hush, don't tell anyone!
     1 file changed, 0 insertions(+), 0 deletions(-)
     create mode 100644 .mysecrets

<small>
You'll need to edit and commit `.gitattributes` yourself if you want to stop encrypting a file.

The `git dot encrypt` command accepts one or more

* files, like `.mydotfile`
* paths, like `.mydotdir/myfile`
* wildcards, like `.mydotdir/\*.key`

like this:

    $ git dot encrypt .mydotfile .gnupg/private-keys-v1.d .mysecrets/\*.key

<small>Note the escaped wildcard (`\*`). Unescaped wildcards are expanded
by your shell and `git dot encrypt` would receive a list of paths.

<small>One entry is added to `.gitattributes` for each file or directory, after confirming existence in the filesystem. Warning messages are issued othewise. Any already represented in `.gitattributes` are ignored.

### Adding files

Use `git dot add` to add plaintext files as you would with any repository:

    $ git dot add .mydotfile
    $ git dot commit -m "my dot file"
    [master 6cece67] my dot file
     1 file changed, 1 insertion(+)

### Listing files

You can list the files in `$HOME` that `git dot` knows about

* `git dot tracked` lists the files being tracked by `git`.
* `git dot encrypted` lists the files that `git dot` will encrypt.
* `git dot plaintext` lists the files that `git dot` will not encrypt.

### Locking the repo

Use `git dot lock` to lock (encrypt) the files in your working copy (`$HOME`) but
*ensure you have the key to unlock it* before doing so!

Also consider the following:

* Locking encrypts files in your working `$HOME` directory and this may prevent
applications from operating properly if they rely on such files (think about your
private SSH and GnuPG keys).

* You do not need to lock the repository to secure the files that you have committed
  as encrypted because these are always stored encrypted within Git regardless of
  the lock state of the working directory.

* Git repo operations such as `clone`, `fetch`, `pull` and `push` always see encrypted
  content in its encrypted state.

**There is no operational reason to lock the working `$HOME` directory**. Encrypting
working dotfiles is not a design goal for `git dot` and is only possible because
[git-crypt] provides the encryption for `git dot`.

### Other commands

Use `git dot locked` to find out if the repository is locked. 

Use `git dot help` and `git dot license` for further information about `git dot`.
Also `git help` and `git crypt help` explain their respective commands, which
`git dot` can pass through (like the above examples demonstrate).


### Contributing

Contributions are welcome. Please use [Github][github-git-dot] to communicate
issues or pull requests.

### *I've locked my keys in my car!*

> I need my key to unlock the repo but my key is locked inside it!

There is no way around it - you need a key to unlock encrypted content.

**Make sure that you have a key before locking a new repository.**

Do this:

    $ git dot export-key repo-key

and keep `repo-key` safe but outside your repository. The name `repo-key` is not
a dotfile so, unless you force it, it won't get locked inside your repo.

If you're tempted to rely on your GnuPG key then think first because the secret
key you need may be locked inside your repo.

The intended use-case for `git dot` is secure storage of dotfiles in a Git repo
(i.e. separate from the working files in your home directory). This does not require
the working copy to be locked at all. Under normal use the only time a working copy
should be locked is if it has just been cloned.

**If you clone a repo then you will need its key.**

[git-dot]:   https://git-dot.johnlane.ie
[git-crypt]: https://www.agwa.name/projects/git-crypt
[github-git-dot]: https://github.com/johnlane/git-dot.git

<div class="message" style="font-size:75%">

Git-Dot is free software; you can redistribute it and/or modify it under the terms of the <a href="http://www.gnu.org/licenses/gpl.html">GNU General Public License</a> as published by the Free Software Foundation; either version 3 of the License, or (at your option) any later version.

</div>
