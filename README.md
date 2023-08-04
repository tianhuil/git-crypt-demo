# git-crypt-demo

Demo for git crypt

## Introduction

This demo shows how to make [git-crypt](https://github.com/AGWA/git-crypt) work with github workflows using [https://github.com/sliteteam/github-action-git-crypt-unlock](this github workflow).  This is a seamless solution with (almost) reproducible solution for secrets management:

1. Symmetric keys are stored plaintext but not commited to git in `.git/git-crypt/keys/`.
   - Servers (e.g. CI) will use the plaintext symmetric keys in (1) to unlock the corresponding secrets file.
2. Those same symmetric keys are encrypted using the user's PGP public keys and commited to git in `.git-crypt/keys/`.
   - Git filters are specified in `.gitattributes` that automatically encrypt and decrypt secrets transparently to users with PGP private keys mentioned in (2).

## How to Use

1. Install gpg on your os (`brew install gpg` or `sudo apt-get gpg`) and create a user (if you haven't done so already).

   ```bash
   gpg --passphrase '' --quick-gen-key 'user@example.com'
   ```

2. Initialize the repository

   ```bash
   git crypt init
   ```

3. Add every engineer's public key `git crypt` to enable them to decrypt the secret.  On your computer, you can just do this:

   ```bash
   git crypt add-gpg-user me@example.com
   ```

   To add a key from another computer, do this:

   ```bash
   gpg --export -a another@example.com > public.key
   # send the key through a public channel
   gpg --import public.key
   gpg --sign-key another@example.com
   git crypt add-gpg-user another@example.com
   ```

4. Export a symmetric key for github

   ```bash
    git crypt export-key -k development - | base64 | pbcopy
   ```

   Paste that into the secrets of your repository (the one for this repository is at <https://github.com/tianhuil/git-crypt-demo/settings/secrets/>  actions) as the value for the key `GIT_CRYPT_KEY`.

5. On github workflow, use `sliteteam/github-action-git-crypt-unlock@1.0.2` and supply `GIT_CRYPT_KEY` from github secrets
6. After using your secrets on the workflow, remove the secrets file (for extra security).

This repo (intentionally) exposes the secret in the [github workflow](https://github.com/tianhuil/git-crypt-demo/runs/1545130895?check_suite_focus=true) but this [remains encrypted in git](https://github.com/tianhuil/git-crypt-demo/blob/main/file.secret).

## View gpg users

To easily check which users have been added to git crypt, add the alias

```bash
git config [--global] alias.crypt-users "! git log  .git-crypt/keys/*/*/*.gpg | egrep '\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,6}\\b'"
```

to add the config either locally or globally.

Based on [https://github.com/AGWA/git-crypt/issues/39](this Github Issue).

## View key fingerprint

To view the key fingerprint (e.g. when exchanging public keys), you can check the fingerprint of the original and the copy you received:

```bash
gpg --fingerprint another@example.com
```

## Staging secrets

The rules for what are set as a secret in `.gitattributes` are subtle and behave unexpectedly.  When making changes to secrets, always run

```bash
git crypt status -e
```

to view which files are secrets.

## Signing and trusting keys

When you get a key, you need to both sign and trust it to use for git crypt (presumably after checking the fingerprint).

```bash
gpg --edit-key another@example.com
>>> trust
>>> quit
gpg --sign-key another@example.com
```

And follow the instructions [for trusting here](https://www.gnupg.org/gph/en/manual/x334.html).

## How adding gpg keys works under the hood

From [these](https://github.com/AGWA/git-crypt/issues/47#issuecomment-103765784) [comments](https://github.com/AGWA/git-crypt/issues/47#issuecomment-103778947):
> git-crypt supports two forms of encryption -- symmetric key and GPG [...] Technically, a symmetric key is used for encryption of files in every case - it is just that in the second case the symmetric key itself is encrypted with one or more GPG keys and those copies of the encrypted key are committed to the repository, allowing a user whose GPG key was "added" to decrypt the encrypted contents in the repository using nothing more than their private key and the repository itself.

## Resources

-[GPG Cheatsheet](http://irtfweb.ifa.hawaii.edu/~lockhart/gpg/)
