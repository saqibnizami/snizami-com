---
date: 2019-09-02
title: Migrating Git Repositories with History
summary: How to mirror a git repository while preserving history.
tags: ["git", "code management"]
categories: ["how-to"]
---

## Summary
This is a very simple post detailing how to mirror a repo to a new account. I have repositories under other github accounts and on private git servers that I wanted to consolidate.

## Steps
```bash
git clone --mirror URL/REPO.git
```
1. Clone the repo with a `--mirror` tag. This will create a `REPO.git` directory.

2. Create a new repository on the server you would like to mirror to, lets call the address to this: `NEW-REPO-URL`. Note: the new repository doesn't have to have the same name.

```bash
cd REPO.git
```
3. Change to the newly mirrored repo

```bash
git remote set-url --push origin NEW-REPO-URL
```
4. Add the address to the new repo as a remote origin

```bash
git push --mirror
```
5. Push the repo to the new server/account.

## Notes
All the branches and commit history should be available in the new repository. However, I did notice that some repositories that I mirrored from GitHub Enterprise servers had my username as `GitHub Enterprise`. Unsure of how to fix that for now, but it isn't a deal breaker for my particular use case.