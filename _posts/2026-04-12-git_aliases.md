---
title: "Useful git aliases"
date: 2026-04-12
layout: single
categories: [git]
---

I wanted to share with you all some commands which I find useful to invoke via `git` when I 
am working on my own personal software projects.

Here is a quick summary of all of the aliases we are going to go through (this is the contents
of my `~/.gitconfig` file),

```
[alias]
    tree = log --all --decorate --oneline --graph
    prune = fetch --all --prune
    history = "for-each-ref --sort=-authordate --format=\"%(authordate) - %(refname) - %(authorname)\""
    branches = branch -va
```

As a reminder: to set aliases you can either manually edit the file mentioned above or you can do it 
via `git` directly like so,

```sh
git config --global alias.co "checkout"
```

## Visualise Branch History

Oftentimes, I find that I want to *inspect* my history in the form of a network graph.

Luckily `git` comes equipped with just the command for that,

```sh
# aliased to 'tree'
git log --all --decorate --oneline --graph
```

This produces a rather snazzy graphical output like so,

```
> git tree

* 1a0d232 (HEAD -> dev) Initialised post on git aliases
* db809eb (origin/main, main) Removed draft post as blog now setup
* 50c1098 Cleaned up blog post spacing
* 139ddd7 Final draft of initial post
* edd8d48 Testing Python code
* c12d9c6 Edited draft of first blog post
* 0022b5c Fixed bullet point error
* eaf7b5f Added initial draft
* c0af07b Added initial ramblings to blog post
* 5df023a Testing emoji in title
* 8acf0f7 Testing emojis
* e836c18 Added layout
* 79a1b5d Amended blog title
* f18934f Cleaned up the navigation
* ce1ca6e Added index file
* e76370a Added navigation;
* fe50cd0 Added first post
* 5fd7760 Added configuration for blog
```
Each line represents a different commit with the prepended hash, any relevant `head` pointers 
and a summarised commit message.

## Prune 'Deleted' Branches

You may (or may not have, if you are lucky) been in the situation before when the remote GitHub repo
is 'cleansed' by another developer (e.g. by removing stale branches).

The trouble is, you're sometimes left with the old references dangling in your local area which can make
your `git branch` output larger than it needs to be (and only adds noise).

To sync your local area with the remote - and delete old references - you can use this command,

```sh
# aliased to 'prune'
git fetch --all --prune
```

## Check Latest Branches

I had a problem the other day: I'd been tasked with cleaning up a messy repository
which had, of many other concerns, lots and lots of stale branches. But with 20+ branches,
it can be hard to deduce which ones are 'most relevant'.

That's why I like to have a handy `history` alias which allows you to view information
across all of your branches *sorted* by date,

```sh
# aliased to 'history'
for-each-ref --sort=-authordate --format="%(authordate) - %(refname) - %(authorname)"
```

Note: when you add this as an alias, you need to escape the quotations with backslash symbols (`\`).

When you run this command, you'll get a nice picture of your repository like so,

```
> git history

Mon Apr 13 09:14:40 2026 +0100 - refs/heads/main - nonstandarddev
Mon Apr 13 09:14:40 2026 +0100 - refs/remotes/origin/main - nonstandarddev
```

Which, for this repository... is ridiculously simple! But you can imagine this command being 
marginally more useful in a more complex repository with many moving parts!

Another related alias (included above) is `git branches` which is just short for `git branch -va`.
This shows you all of your branches (including the currently checked out branch) but does not sort
the output in any particular order.

