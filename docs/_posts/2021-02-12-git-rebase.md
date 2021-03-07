---
layout: post
title:  "A git branching strategy for concurrently contributing to a git project"
date:   2021-02-12 14:24 +0300
categories: git, git-rebase
---

# Rebase tutorial

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

## Notions to operate with

- `master` branch: the branch where the stable code lies. Usually nobody should be able to directly push to master.
- `feature` branch: the branch with the code of the functionality that should be added to the stable code.

## Problem description

Imagine a `master` branch with already existing commits:

```
A   B   C (master)
*-->*-->*
```

And the developer decides to create a new feature branch:
```bash
(master)$ git checkout -b feature1
```

Result:
```
A   B   C (remote:master, local:feature1)
*-->*-->*
```

Now the developer adds a commit into `feature1`:

```
A   B   C (remote:master)
*-->*-->*
         \       D (local:feature1)
          \----->*
```

What happens when someone merges his branch into `master` before you? 

```
A   B   C   E (remote:master)
*-->*-->*-->*
         \     D (local:feature1)
          \--->*
```

Your branch becomes outdated so if you will merge your branch now you will get:

```
A   B   C   E   M_1(remote:master)
*-->*-->*-->*--->*
         \      /    
          \-->*/ D (local:feature1)
```

where `M_1` is a merge commit. Up until now nothing scary and it looks pretty nice, but imagine having several commits merged in the meantime that start from different commits and everything looks entangled that you cannot see what code does your `feature1` branch wants to merge without seeing the diff from the merge request.

```
                 F   G (local:feature2)
                *-->*\
               /      \
A   B   C   E /  M_1   \ M_2 (remote:master)
*-->*-->*-->*--->*----->*
         \      /
          \-->*/ D (local:feature1)
```

In order to avoid this, before merging the merge request, rebase your `feature1` branch on top of master:

```bash
(feature1)$ git checkout master
(master)$ git pull origin master
(master)$ git checkout feature1
(feature1)$ git rebase master
(feature1)$ git push -f origin feature1
```

What do the above commands do:
1. Switch from branch `frature1` to `master` on your local repository
2. Update your local `master` branch by pulling all the new commits from origin master
3. Switch back to branch `feature1`
4. The most important part: rebase `feature1` branch on top of `master` branch
5. Update your remote branch by force pushing to `origin:feature1`

Before:

```
A   B   C   E (remote:master)
*-->*-->*-->*
         \     D (remote:feature1)
          \--->*
```

After step 4:

```
A   B   C   E (remote:master)
*-->*-->*-->*
         \   \    D' (local:feature1)
          \   \--->*
           \    D (remote:feature1)
            \--->*
```

Note that the hash of the commit has changed after rebasing at step 4, so on your remote `feature1` branch it is still the commit with hash `D`, while on your local branch, the hash of the commit changed to `D'` - the changes are the same if no conflicts arose during rebasing. That's why we should force push `feature1` to our remote repository.

After step 5:

```
A   B   C   E (remote:master)
*-->*-->*-->*
             \    D' (remote:feature1)
              \--->*

```

Those 5 steps from above can be substituted with a nicer suite of commands:

```bash
(feature1)$ git pull -r origin master
(feature1)$ git push -f origin feature1
```

Note the `git pull -r origin master` command - it is an alias for the first 4 commands.

In case that during rebasing you have conflicts:

```bash
(feature1)$ git pull -r origin master
(rebasing-in-progress)$ ... some conflicts detected, please solve them ...
```

Solve your conflicts and after that do:

```bash
(rebasing-in-progress)$ git add .
(rebasing-in-progress)$ git rebase --continue
(feature1)$ git push -f origin feature1 
```

## Summary

Remember the following commands:

1. `git pull -r origin master`
2. `git add .` - in case you had conflicts and you resolved them all
3. `git rebase --continue` - in case of conflicts and after the step above
4. `git push -f origin <your_feature_branch_name>` - force update your remote feature branch


More information about rebasing you can find [here](https://git-scm.com/book/en/v2/Git-Branching-Rebasing).
 