---
title: "Sh*t with git: worktrees"
categories:
  - programming
tags:
  - git
  - worktrees
---

# Sh*t with `git`: worktrees

Since about the first year of my studies I've been using `git`. I manage and
version my personal documents, letters, system configuration, notes, tax
declaration (oioi, I should really finish last year's taxes soon..), basically
my digital life with `git`, using a little server with `gitolite` as
home-local server and synchronization point where needed.

Additionally, of course, I've been working with `git`. Basically every project I
worked with in the last many years was versioned with `git`, with the exeption
of `perforce` being used by my first employer (🤮, not the employer, great
people. I had a lot of fun there and learned a lot. But perforce...)

Anyway, what I am going for is, I have used `git` for a long time. I have
understood `git` for a bit less time, I have been confused by `git` many times
and I spent a lot of time trying to resolve merge conflicts. I think I have a
moderately solid understanding of the basic concepts of `git`, I know
implementation details and I can explain the difference between a `git merge`
and a `git rebase`. For the last few years, I have just simply used `git`
without thinking about it and adapting my basic workflow.

Until about 3 weeks ago.

I have heard of [worktree](https://git-scm.com/docs/git-worktree) before, but
never passed the stage of "Huh, interesting, I have to look into that one day."

Whenever I wanted to quickly fix something on main, wanted to review a specific
branch or for whatever other reason do something outside of the context of the
branch I am currently working on, I would stash / commit my current changes,
switch branches, do the thing, switch back and either pop the stash or reset my
branch / amend to the temporary commit.

Whenever I needed parallel states of a repository, e.g. because binary files
have to be examined or for a bit of crude copy paste action or parallel work or
whatever, I would create physical copies of the repository in the same top
level directory, à `my_repo` and `my_repo_copy`. Which is a bit sh*t, for
several reasons. Nobody needs several identical index directories, and it's
just confusing as well as a lot of manual action.

## The Problem

I have to or want to quickly work on a different branch than my current working
state, or I want to work with several branches at the same time.

I don't want to create intermediate commits, stash and pop, manually copy paste
and create error prone workflows that need manual cleanup.

## Solution

[![asciicast](https://asciinema.org/a/1261378.svg)](https://asciinema.org/a/1261378)

`git worktree`!

Manage multiple working trees attached to the same repository.

A `git` repository can support multiple working trees, allowing you to check
out more than one branch at a time. With `git worktree add` a new working tree
is associated with the repository, along with additional metadata that
differentiates that working tree from others in the same repository. The
working tree, along with this metadata, is called a "worktree".

What does that mean?

I have a repository, e.g. the one with this content from
[github](https://github.com/gierdo/gierdo.github.io).

It's cloned locally in `gierdo.github.io` and checked out to `master`. Now, I
want to work on the new article about worktrees in the branch `worktrees`. I
could run a `git checkout -b worktrees` or `git checkout worktrees` and switch
branches on my "main worktree", which is, well, the main worktree you always
get when you clone or init a repository.

Or I could checkout the branch to a "linked worktree" under a specified
directory, reusing my entire rest of the setup and the already present `.git/`,
with `git worktree add -b worktrees place_for_my_worktrees/worktrees` or `git
worktree add worktrees place_for_my_worktrees/worktrees`

This creates a directory `place_for_my_worktrees/worktrees`, which contains the
(you guessed it) linked worktree.

Pretty handy!

To make it even more handy for me and support shell completion including fuzzy
searching, I configured a bit of sugar around it.

In my global gitconfig, there are two _alias_ configurations, allowing me to
make sure that all worktrees are added under the same repository-local
directory (`worktrees/`):

```toml
[alias]
  worktree-add = "!f() { git worktree add -b \"$1\" \"worktrees/$1\"; }; f"
  worktree-checkout = "!f() { git worktree add \"worktrees/$1\" \"$1\"; }; f"
```

My global gitignore contains the `woktrees` directory, as my worktrees are
in-tree of the parent "main tree". Yes, this is potentially a bit dangerous,
but also very convenient. I'm a bit undecided about this, but in all 3 weeks of
experience with it I haven't encountered any issues, and it's easy enough to
change if needs be..

```text
...
tags
.tool-versions
.vim
worktrees
...
```

My custom zsh completion configuration contains the completion definition to
complete branches for the git worktree checkout, which makes it behave like a
normal git checkout, including fuzzy completion if configured correctly.

```zsh
#compdef git-worktree-checkout

_git-worktree-checkout() {
    _values 'branches' ${${(f)"$(git for-each-ref --format='%(refname:short)' refs/heads refs/remotes 2>/dev/null)"}:-""}
}
```

If you are interested in the full configuration, it's all [here](https://github.com/gierdo/dotfiles).

## tl;dr

- sometimes you need several branches or you have to quickly switch branches
- workarounds are possible
- workarounds are workarounds
- `git worktree` might be better than the alternatives
