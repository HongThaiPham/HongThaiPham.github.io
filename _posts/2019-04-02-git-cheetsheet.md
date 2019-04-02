---
layout: post
title: 'GIT: The basic'
date: 2019-04-02 14:44:00 +0700
categories: [uncategory]
tags: [git]
image: place_holder.jpeg
---

## Three states

1. **Committed** means that the data is safely stored in your local database.
2. **Modified** means that you have changed the file but have not committed it to your database yet.
3. **Staged** means that you have marked a modified file in its current version to go into your next commit snapshot.

## Config

- Global

```
git config --global user.name "John Doe"
git config --global user.email "John@gmail.com"
```

- Local

```
git config --local user.name "John Doe"
git config --local user.email "John@gmail.com"
```

Check git config with:

```
git config --list
```

## Operator

`git init` for create and git folder

`git status` for status files (untrack, modified, staged)

`git add <file>` for add files to staged

`git reset HEAD -- <file>` which effectively reverts git add and prevents the changes to this file from participating in the next commit.

```
git log
git log -p
git log --stat
git log --pretty
```

`git commit --amend` edit the last commit with the new one, (just `git add <file>` and `git commit --amend`)

`git checkout -- <file>` to checkout (revert file want)

`git remote`

`git remote` To see which remote servers you have configured

`git remote -v` To shows you the URLs

`git remote add [shortname] [url]` to add new remote git repository

`git fetch [remote-name]` to get data from remote project

`git push [remote-name] [branch-name]` to push

`git remote rename [remote-name] [remote-newname]`

`git remote rm [remote-name]`

Make `git tag`: Git uses two main types of tags: lightweight and annotated. A lightweight tag is very much like a branch that doesn’t change — it’s just a pointer to a specific commit. Annotated tags, however, are stored as full objects in the Git database.

`git tag` to show tag

`git tag -a [tagname] -m [message]` to create an annotated

After that we can use `git show [tagname]`

`git branch` to list all branch

`git checkout <name-branch>` to switch to that branch

`git branch -d <name-branch>` to deleted branch

`git branch --merged` to figure out branchs merged,

`git branch --no-merged` list branchs didn't merge
