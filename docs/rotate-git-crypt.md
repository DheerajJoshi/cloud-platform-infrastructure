# Rotating the git-crypt key

We use `git-crypt` for securing secrets committed to `git`. It uses a symmetric encryption key which is then encrypted for users using their GPG keys.

When rotating the `git-crypt` symmetric key, you should follow the steps below:

1. Ensure you have unlocked the repository: `git-crypt unlock`

2. Make a list of the current users:
  ```
  # for every file found, extract its filename minus the suffix,
  # which corresponds to the user's key id
  ls .git-crypt/keys/default/0/*.gpg | xargs -I{} -- basename {} .gpg > git-crypt-users
  ```

2. Make a list of the encrypted files:
  ```
  # for every currently encrypted file, extract its path, relative
  # to the root of the repository
  git-crypt status -e | awk '{ print $2; }' > git-crypt-files
  ```

3. Remove `git-crypt` configuration:
  ```
  rm -rf .git-crypt .git/git-crypt
  ```

4. Re-initialise `git-crypt`,
  ```
  git-crypt init
  ```
  make sure you have all the users' keys,
  ```
  cat git-crypt-users | xargs -I{} gpg --recv-key {}
  ```
  and add all the users back,
  ```
  cat git-crypt-users | xargs -I{} git-crypt add-gpg-user -n --trusted {}
  ```
  At this point, your git index will have a number of pending changes, do not commit them yet.

5. Re-encrypt all the secrets: the list of secrets was extracted in `git-crypt-files` (see step 2). Since the master encryption key is being rotated, all of the secrets that have been encrypted with it *must* be rotated. Once finished, you can add the files to the index

  ```
  cat git-crypt-files | xargs git add
  ```

6. Commit your changes. One way to check that the files in the git index are properly encrypted before you commit your changes is like so:
  ```
  git show :<path-to-file>
  ```
  or after committing (and before you've pushed):
  ```
  git show HEAD:<path-to-file>
  ```
  where `<path-to-file>` is either absolute from the base of the git repo or relative (eg.: `git show:./my-secret-file.yaml`)

  You should see binary output which begins with `^@GITCRYPT^@`.

Note: If you need to `checkout` an older commit, branch, tag etc., make sure to `git-crypt lock` your repository before in order to avoid a broken local working directory. Once you've locked and checked out the desired revision, you can `git-crypt unlock`.
