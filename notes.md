---
layout: page
title: notes
---
Git-Dot development notes
=========================
These are old notes made during development and only preserved for reference. They
may be incomplete or not make any sense. You probably want to read [this](/) instead.

On public hosting
-----------------

I don't host my dotfiles repo on cloud service such as GitHub, only on my private Gitolite server. I may use my [keybase] to host it privately on [kbfs].

Keybase uses [TripleSec], a simple, triple-paranoid, symmetric encryption library borne of [keybase].

On other projects
-----------------

Many people store dotfiles in version control systems such as `git` and there are several existing projects, such as:

* [dotfiles on Github], your unofficial guide to dotfiles on GitHub
* [Yet another dotfiles manager][YADM]
* [Yet another dotfiles repository][YADR]

[YADR]: https://skwp.github.io/dotfiles
[YADM]: https://thelocehiliosan.github.io/yadm/
[dotfiles on Github]: https://dotfiles.github.io
[todo]: https://en.wikipedia.org/wiki/Comment_(computer_programming)#Tags
[dotfiles]:  https://en.wikipedia.org/wiki/Dot-file
[keybase]: https://keybase.io
[triplesec]: https://keybase.io/triplesec
[kbfs]: https://keybase.io/docs/kbfs

I am uncomfortable allowing a some third-party tool to write into my home directory, so I decdided this is an occasion where wheel reinvention is called for...

Here is an exampe...

> YADR will take over your `~/.gitconfig`, so if you want to store your usernames, please put them into `~/.gitconfig.user`

Git-Dot uses Git-Crypt
----------------------

The cryptographic heavy-lifting has alread been implemented as [git-crypt], available on [GitHub][github-git-crypt]. An [Archlinux package][aur-git-crypt] for the latest version, currently 0.5.0 (released 20150531), is available on the AUR. There is also a [package for the master HEAD][aur-git-crypt-git].

[git-crypt]: https://www.agwa.name/projects/git-crypt
[github-git-crypt]: https://github.com/AGWA/git-crypt
[aur-git-crypt]: https://aur.archlinux.org/packages/git-crypt
[aur-git-crypt-git]: https://aur.archlinux.org/packages/git-crypt-git

Prototype
---------

The following prototype was sufficient to prove the concept:

    #!/bin/sh
    # /usr/bin/git-dot
    exec git --git-dir="$HOME/.git-dot" --work-tree="$HOME" "$@"

That isn't a lot of code for a functioning dotfiles repo. It allows, for example:

    $ git dot commit ...

The `--git-dir` option to `git` allows an alternative to `.git` to be specified, so we can use something like `~/.git-dot` (an alternative is to use the `GIT_DIR` environment variable).

A layer of helpers
------------------

The `git dot` script is little more than a layer of helpers wrapping `git` and
`git crypt` commands that could otherwise be given longhand. These helpers 
make the experience more transparent. Here is a command summary:

* `git dot init` performs a `git init`, a `git crypt init` and writes an exclude list (similar to `.gitignore`) to `$GIT_DIR/info/exclude`.
* `git dot clone` performs a `git clone` to clone a bare repository into `$GIT_DIR` and then configure it for use by the working copy in `$HOME`.
* `git dot encrypt` adds file specifiers to `.gitattributes` as required by [git-crypt] and commits it.
* `git dot lock` and `git dot unlock` do what they say. They do nothing if already in their target state.
* `git dot locked` returns `yes` if the repo is in a locked state, otherwise `no`.
* `git dot repo` returns `yes` if a repo is configured, otherwise `no`

Other `git` or `git crypt` commands may be given as `git dot ...` or `git dot crypt ...`.

Exclude File
------------

The `$GIT_DIR/info/exclude` file works similarly and in addition to `.gitignore'. The
`git dot init` command prepopulates this file:

    *
    !.*
    !.*/**
    *.*~
    $GIT_DIR

The Repo
--------

The repo is initilaised as a bare repo at `$HOME/.git-dot` and then reconfigured as
a non-bare repo with a working directory at `$HOME`.

The basic `.git/config` is

    [core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
    [remote "origin"]
        url = git@git:dotfiles.git
        fetch = +refs/heads/*:refs/remotes/origin/*
    [branch "master"]
        remote = origin
        merge = refs/heads/master

What `git dot` produces:

    [core]
       repositoryformatversion = 0
       filemode = true
       bare = false
       logallrefupdates = true
       worktree = /home/alice
    [filter "git-crypt"]
       smudge = \"git-crypt\" smudge
       clean = \"git-crypt\" clean
       required = true
    [diff "git-crypt"]
       textconv = \"git-crypt\" diff

Restoring backup files
----------------------

The below command can be used to move files from the backup into the working
directory if they are not encrypted:

    $ comm -12 \
        <(git dot crypt status | grep '^not' | awk '{print $3}' | sort -u) \
        <(cd git-dot-backup.S1az0cs1.tmp; find  -type f -printf '%P\n' | sort -u) | \
      ( cd git-dot-backup.S1az0cs1.tmp; xargs -I {} sh -c 'cp --parents -l -f {} "$HOME"; rm {}')

Usage
-----

It *just works*:

    $ git dot init
    $ git dot encrypt .netrc .ssh/id_rsa .gnupg/private-keys-v1.d/\*.key
    $ git dot add ssh/id_rsa .gnupg/private-keys-v1.d/*.key
    $ git dot commit -m "encrypted files"
    $ git dot crypt export-key repo-key

### Push

Push to another local filesystem location:

    $ git init --bare /path/to/dotfiles.git
    $ git dot remote add transfer /path/to/dotfiles.git
    $ git dot push -u transfer master

### Reset

To reset `$HOME` (for testing purposes) only requires removal of the `git dot` files:

    $ cd ~
    $ rm -rf .git-dot .gitattributes .git-crypt

## Example: what I did:

    $ rm -rf ~/.git-dot .git-crypt .gitattributes
    $ git dot init
    $ git dot crypt add-gpg-user 22D05A45
    $ git dot encrypt $(file .ssh/* | grep 'PEM RSA private key' | awk -F ':' '{print $1}' )
    $ git dot add $(file .ssh/* | grep 'PEM RSA private key' | awk -F ':' '{print $1}' )
    $ git dot encrypt .gnupg/{openpgp-revocs.d/\*.rev,private-keys-v1.d/\*.key}
    $ git dot add .gnupg/{openpgp-revocs,private-keys-v1}.d
    $ git dot encrypt .netrc .gist 
    $ git dot add .netrc .gist
    $ git dot commit -m 'Encrypted dotfiles'

Set up remote and push

    $ git dot remote add origin git@git:dotfiles
    $ git dot push -f -u origin master

<small>I used `-f` to force-push because I wanted to overwrite an old remote. future pushes can be done by `git push`</small>

ssh:

    $ git dot add $(file .ssh/* | grep 'OpenSSH RSA public key' | awk -F ':' '{print $1}' )
    $ git dot reset .ssh/authorized_keys # unstage - added by prior cmd
    $ git dot add .ssh/config
    $ echo .ssh/authorized_keys >> ~/.gitignore
    $ echo .ssh/known_hosts >> ~/.gitignore
    $ git dot add .gitignore
    $ git dot commit -m 'SSH non-sensitive files: config and public keys'

bash

    $ echo .bash_history >> ~/.gitignore
    $ git dot add .bash_logout .bash_profile .bashrc .gitignore
    $ git dot commit -m "Bash dotfiles; Ignore .bash_history"

gpg:

    $ echo .gnupg/random_seed >> .gitignore
    $ git dot add .gnupg/{gpg{,-agent}.conf,pubring.kbx,trustdb.gpg} .gitignore
    $ git dot commit -m .gnupg

Other host

    $ git dot clone git@git:dotfiles



Observations:

* `.git-crypt` does not exist until `add-gpg-user` is used. It contains the added GnuPG key and a copy of `.gitattributes`.

The copy file `.git-crypt/.gitattributes` contains this header:

    # Do not edit this file.  To specify the files to encrypt, create your own
    # .gitattributes file in the directory where your files are.
    
Debug output
------------

There's no verbose option to `git-dot` because I didn't want to interfere with
the arguments to `git` and `git-crypt`. However, to output the `git` commands that
`git dot` issues, Duplicate the `command` line in the `git` function and change the
`command` to `echo >&2`:

    echo >&2 git --git-dir="$GIT_DIR" --work-tree="$HOME" "$@"
    command git --git-dir="$GIT_DIR" --work-tree="$HOME" "$@"

Testing
-------

Use a test user

    sudo userdel -r alice
    sudo useradd -m alice
    sudo cp repo-key ~alice
    sudo cp -a .ssh ~alice/
    sudo chown -R alice: ~alice/.ssh
    sudo -i -u alice

Then

    $ git dot clone git@git:dotfiles.git
    Cloning into bare repository '/home/alice/.git-dot'...
    Enter passphrase for key '/home/alice/.ssh/johnlane_rsa': 
    remote: Counting objects: 144, done.
    remote: Compressing objects: 100% (132/132), done.
    Receiving objects: 100% (144/144), 498.81 KiB | 0 bytes/s, done.
    remote: Total 144 (delta 45), reused 0 (delta 0)
    Resolving deltas: 100% (45/45), done.

    The following files are now encrypted:
    .ssh/id_rsa
    The original files have been moved to './git-dot-backup.QGKRgVMu.tmp'.
    Unlock the encrypted files before moving them back.

    Your working directory is unclean. Please stash changes before unlocking.
      (use "git dot stash" before and "git dot stash pop" after unlocking)


The following example demonstrates a clone operation:


    #!/bin/sh
    git config --global user.email "alice@example.com"
    git config --global user.name "Alice"
    git="git --git-dir=/home/alice/.git-dot --work-tree=/home/alice"
    $git clone --bare git@git:dotfiles /home/alice/.git-dot
    $git config core.bare false
    $git config core.logallrefupdates true
    $git  reset
    
    backup=$(mktemp -p . -d)
    $git ls-tree -r master --name-only | xargs -I f cp -a --parents -l f "$backup" 2>/dev/null
    
    $git checkout .
    $git config remote.origin.fetch +refs/heads/*:refs/remotes/origin/*
    $git config branch.master.remote origin
    $git config branch.master.merge refs/heads/master
    $git config filter.git-crypt.smudge '"git-crypt" smudge'
    $git config filter.git-crypt.clean '"git-crypt" clean'
    $git config diff.git-crypt.textconv '"git-crypt" diff'
    exit
     cat <<-EOF > "/home/alice/.git-dot/info/exclude"
            *
            !.*
            !.*/**
            *.un~
            *.*~
            .git-dot
    EOF

Issues
------

There is an issue with cloning a repo where the first command outputs error
messages, one per encrypted file. The command works and subsequent commands
do not report any errors. This is rised as issue [github-git-crypt-108].

[github-git-crypt-108]: https://github.com/AGWA/git-crypt/issues/108

The issue is worked around in the `git-dot` script.

Refs
----

* [Versioning Personal Configuration Dotfiles](http://nullprogram.com/blog/2012/06/23/)
* [Publishing My Private Keys](http://nullprogram.com/blog/2012/06/24/)
* [skeeto/dotfiles](https://github.com/skeeto/dotfiles) on GitHub


<div class="message" style="font-size:75%">

Git-Dot is free software; you can redistribute it and/or modify it under the terms of the <a href="http://www.gnu.org/licenses/gpl.html">GNU General Public License</a> as published by the Free Software Foundation; either version 3 of the License, or (at your option) any later version.

</div>
