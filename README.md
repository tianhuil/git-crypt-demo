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

2. Initialize the repository (for development key but you can use whichever you wish)

   ```bash
   git-crypt init -k development
   ```

3. Add every engineer's public key to git crypt to enable them to decrypt the secret.  On your computer, you can just do this:

   ```bash
   git-crypt add-gpg-user -k development me@example.com
   ```

   To add a key from another computer, do this:

   ```bash
   gpg --export -a another@example.com > public.key
   # send the key through a public channel
   gpg --import public.key
   gpg --sign-key another@example.com
   git-crypt add-gpg-user -k development another@example.com
   ```

4. Export a symmetric key for github

   ```bash
    git-crypt export-key -k development - | base64 | pbcopy
   ```

   Paste that into the secrets of your repository (the one for this repository is at <https://github.com/tianhuil/git-crypt-demo/settings/secrets/>  actions) as the value for the key `GIT_CRYPT_KEY`.

5. On github workflow, use `sliteteam/github-action-git-crypt-unlock@1.0.2` and supply `GIT_CRYPT_KEY` from github secrets

6. If you need to unlock secrets in an environment that does not support Github Actions (e.g. Vercel), follow step (4) above to add `GIT_CRYPT_KEY` as a secret to that environment.  Then run the following in a bash script to unlock git crypt:

   ```bash
   echo "$GIT_CRYPT_KEY" | base64 -d > ./git-crypt-key
   git-crypt unlock ./git-crypt-key
   rm ./git-crypt-key
   ```

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
git-crypt status -e
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

## Rotating secrets

The protocol for rotating secrets is to keep the secrets file unchanged but
rotate the key.  Specifically, it is to run the following steps that generates 2
commits.  The below steps rotate the file `development.secret` to the key
`development-v4`.  For more details, see the pull request
<https://github.com/tianhuil/git-crypt-demo/pull/3>.

1. Generate new symmetric key and grant permission to your private asymmetric
   GPG key (the `add-gpg-user` command automatically commits the GPG key):

   ```bash
   git-crypt init -k development-v4
   git-crypt add-gpg-user -k development-v4 alice@example.com # commit generated
   ```

2. Update the line in `.gitattributes` to use the latest key.

   ```text
   # .gitattributes

   development.secret filter=git-crypt-development-v4 diff=git-crypt-development-v4
   ```

3. Re-encrypt the file

   ```bash
   mv development.secret .tmp
   mv .tmp development.secret
   ```

4. Before committing the changes from step (3), verify that the secrets file is
   encrypted:

   ```bash
   git status # make sure development.secret has binary changes
   git crypt status -e # make sure development.secret is encrypted
   # commit the changes
   ```

5. Before pushing the changes to Github, lock the files and diff to ensure that
   the keys has changed

   ```bash
   git crypt lock -k development-v4
   git diff main -- development.secret
   git crypt unlock
   ```

   While the repository is unlocked, you should also be able to checkout older
   branches that use the old symmetric key, thus transparently preserving
   backward compatibility.

   ```bash
   git diff main -- development.secret
   
   more .gitattributes  # new key
   more development.secret  # should be binary
   git checkout main
   more .gitattributes  # old key
   more development.secret  # should be binary
   ```

6. To add the secret to an external service, copy the command to the clipboard

   ```bash
   git-crypt export-key -k development-v4 - | base64 | pbcopy
   ```

## Resources

-[GPG Cheatsheet](http://irtfweb.ifa.hawaii.edu/~lockhart/gpg/)
