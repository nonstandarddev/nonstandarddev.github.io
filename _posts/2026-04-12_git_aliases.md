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
```

As a reminder: to set aliases you can either manually edit the file mentioned above or you can do it 
via `git` directly like so,

```sh
git config --global alias.co "checkout"
```

## Network Graph

Often you may find you'd like to inspect your git history in a graphical manner.

`git` comes equipped with just the command for that,

```sh
# aliased to 'tree'
git log --all --decorate --oneline --graph
```

This produces a rather snazzy graphical output like so,

```
nonstandarddev.github.io/_posts  dev [!?]                                                                                                                                                             
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

