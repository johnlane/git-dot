Git-Dot - Git for *dotfiles*
============================

**git-dot** lets you easily maintain your *dotfiles* in a Git repository. It integrates
into your workflow as a new `git dot` command and works with [git-crypt] so that files
with sensitive contents can be encrypted.

With **git-dot** you can safely store sensitive material in your repsitory such as
passwords, API tokens, SSH and OpenPGP private keys alongside other files that don't
require such protection.

It

* works directly off your dotfiles in `$HOME`;
* does not use symbolic links;
* uses [git-crypt] to offer a simple method of encyption;
* has no clever host-specific provisions;
* is a `bash` shell script that works on Linux.

More information [here][git-dot]. License GPLv3.

[git-dot]:   https://git-dot.johnlane.ie
[git-crypt]: https://www.agwa.name/projects/git-crypt
