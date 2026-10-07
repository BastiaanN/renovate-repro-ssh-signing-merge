# SSH commit signing + persistRepoData: onboarding `mergeToLocal()` fails

Discussion: _TBD_

## Current behavior

With `gitPrivateKey` set to an **SSH** key and `persistRepoData=true`, the second Renovate run fails when:

- the onboarding PR already exists (created by the first run), and
- `main` received a new commit after the onboarding branch was created (so merging is not a fast-forward).

```
"commands":["merge","FETCH_HEAD"]
error: Couldn't load public key /tmp/git-private-ssh.key: No such file or directory?
fatal: failed to write commit object
"result":"unknown-error","exitCode":8
```

The first run writes `commit.gpgsign=true`, `gpg.format=ssh` and `user.signingkey=/tmp/git-private-ssh.key` into the persisted repo-local `.git/config` and deletes the key file on exit. The next run reuses that checkout, and `mergeToLocal()` runs `git merge FETCH_HEAD` without writing the key first (`handleCommitAuth()` is only called from `prepareCommit()`).

## Expected behavior

Onboarding continues normally (same as with `persistRepoData=false`, which succeeds on a clean base dir).

## Steps to reproduce

> [!IMPORTANT]
> **Fork this repository and run Renovate against your fork** (Renovate needs write access). Run 1 must create the onboarding PR itself: that commit is what writes the signing config into the persisted `.git/config`.

Tested with `renovate/renovate:44.141.0` (Docker). Each `docker run` is a fresh container with an empty `/tmp`, like a CI job or Kubernetes CronJob pod.

```sh
mkdir -p base && chmod 777 base
ssh-keygen -q -t ed25519 -N "" -f signing-key    # key does not need to be registered anywhere
export RENOVATE_GIT_PRIVATE_KEY="$(cat signing-key)"

cat > env.list <<EOF
RENOVATE_TOKEN=<classic PAT with repo scope>
RENOVATE_PLATFORM=github
RENOVATE_REPOSITORIES=<owner>/<fork of this repo>
RENOVATE_BASE_DIR=/base
RENOVATE_PERSIST_REPO_DATA=true
LOG_LEVEL=debug
EOF

run() { docker run --rm --env-file env.list -e RENOVATE_GIT_PRIVATE_KEY -v "$PWD/base:/base" renovate/renovate:44.141.0; }

run   # 1: creates the onboarding PR -> OK
# push any commit to main (e.g. edit this readme)
run   # 2: "Couldn't load public key /tmp/git-private-ssh.key" -> unknown-error
```

A PAT is used, so commits are made with local `git` ([`platformCommit`](https://docs.renovatebot.com/configuration-options/#platformcommit) only applies to GitHub Apps).

Do **not** merge the onboarding PR, otherwise the repository is onboarded and this code path is no longer reached.