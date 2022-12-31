# Git rewrite guide


## License

This article is released under [Creative Commons Attribution-NonCommercial 4.0
International License](http://creativecommons.org/licenses/by-nc/4.0/).


## Motivation

Why rewrite Git history?

Some review workflows require that the branch being reviewed serves as a
template for the future commit series on main, so review revisions are
incremental rewrites to the branch to allow review of both commit diffs, commit
messages and commit partitioning.

> But I've heard that you should never...

In general that only holds for branches that multiple contributors commit to or
base branches on, e.g. the `main` branch.

This document does not take a stand on branch policies, so make sure everybody
agrees that rewriting history is OK before using any of the techniques described
here on a branch.


## Legend

I describe plain Git operations here because that way it's possible to search
the web for `how to do 'git xyz' in tool 'abc'`.

I use `topic` as a placeholder for any topic branch and `main` as a placeholder
for the mainline branch.

CLI examples have a prompt with the branch name in parentheses, to make it clear
which branch they should be executed from, e.g. `(topic)$`


## Basic operations

### Rebase on top of latest mainline

Typical reviewer feedback: "Please rebase on main"

The fastest and least intrusive way to do this on your current topic branch is
to fetch the mainline branch and then rebase on top of the remote branch:

```
(topic)$ git fetch origin main
(topic)$ git rebase origin/main
```

If there are conflicts, see **Resolving conflicts**.


### Reword commit message

Typical reviewer feedback: "Please reference issue number in commit message"

If the latest commit, use `git commit --amend` to modify it in place.

If an earlier commit, use **Interactive rebase** with the `reword` action.


### Reorder commits

Typical reviewer feedback: "Please order refactors before functional changes"

Use an **Interactive rebase** to move commits in the branch history.

If there are conflicts, see **Resolving conflicts**.


### Squash commits

Typical reviewer feedback: "Please commit tests together with code changes"

If necessary, first **Reorder commits** to move commits so they are adjacent.

Then use a second **Interactive rebase** to collapse adjacent commits into
one. There are two possible actions:

* `squash` squashes two or more commits into one while allowing you to edit
  commit message, starting from a sequence of all old messages
* `fixup` squashes two or more commits into one discarding the old messages

Usually, `squash` is preferable because the resulting commit diff changes and it
may be necessary to update the message accordingly.


### Split a commit

Typical reviewer feedback: "Please separate the functional changes from the
unrelated format change"

This is the least intuitive basic operation.

Use an **Interactive rebase** with the `edit` action to stop at the commit you
want to split. Note that it stops *after the commit has been applied*.

Then use a soft reset to get the committed content back into the working copy.

At this point, you can re-add and -commit whole files, or use **Interactive
staging** to commit parts of files in any order.

```
(topic)$ git rebase -i main
Stopped at f04ef13...  Implement one-two-three
You can amend the commit now, with

  git commit --amend

Once you are satisfied with your changes, run

  git rebase --continue

(topic)$ git reset --soft HEAD^

(topic)$ git add file1.c
(topic)$ git commit -m "Implement number one"

(topic)$ git add file2.c
(topic)$ git commit -m "Implement number two"

(topic)$ git add file3.c
(topic)$ git commit -c ORIG_HEAD

(topic)$ git rebase --continue
```

Note `-c ORIG_HEAD` on the last commit, which can be used to reuse the message
from the original commit.


## Techniques

### Resolving conflicts

Conflicts can happen in Git on most non-linear operations (merge, revert,
rebase, cherry-pick) because all of them move content changes around in the
commit sequence.

Conflict markers can be intimidating, but learning how to resolve them
methodically is a necessity for history rewriting.

Rather than butcher the subject here, I refer to the [Git book][gb-conflicts] or
the [GitHub docs][gh-conflicts] for excellent guidance and examples.

[gb-conflicts]: https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging
[gh-conflicts]: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/addressing-merge-conflicts/resolving-a-merge-conflict-using-the-command-line


### Interactive rebase

Learning how to use interactive rebase is a cornerstone of effective history
rewriting.

The `git commit` command has a battery of options to customize inter-commit
relationships (`--squash` and `--fixup`) and individual commit details such as
message, authorship, and timestamps. When used with `git commit --amend`, these
options can also modify the `HEAD` commit in place.

But typically when rewriting, we want to modify commits further back in the
branch history. An interactive rebase combines the power of `git rebase` with
the power of `git commit --amend`.

The rebase command finds the first common commit `C` between the current branch
and the provided base branch `B`, and replays all commits after `C` on top of
`B`. This helps to quickly catch up with a moving mainline branch without
merging back mainline changes and creating a non-linear history.

The interactive rebase allows you to script the rebase process and amend the
commits as they're being replayed.

Everything you can do in an interactive rebase, you could also do in a series of
Git commands, possibly involving temporary branches, cherry-picks, resets and
amends, but the interactive rebase provides a structure and safety net that
makes it much more convenient.

The most common inter-commit operations are:

* Reordering the commit sequence (change order in rebase script)
* Squashing commits together (`fixup`, `squash`)
* Removing commits entirely (`drop` or remove from rebase script)

All other actions can be achieved with `edit`, which stops after the commit and
lets you get creative. The `reword` command is just a shortcut for `edit`
immediately followed by `git commit --amend`.

The [Git book][gb-rewriting] has a good chapter on interactive rebase and
history rewriting in general and there are several online tutorials to help get
acquainted with it.

[gb-rewriting]: https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History


### Interactive staging

Sometimes a programming session leaves you with two independent but valuable
changes in the working directory. The vanilla `git add` command only allows
staging and committing individual files separately, so if there are multiple
unrelated changes to the same file, we need a more fine-grained tool.

Partial staging, `git add -p`, lets you interactively stage by hunk instead of
by file.

Interactive staging with `git add -i` brings up a whole little text-based user
interface for managing the index/stage with files or hunks.

I honestly find both are quite terrible, and I think it's fair to say this is
the poster child for graphical Git clients, which are better equipped to provide
a reasonable user experience.

But they provide an important building block for rewrites, so it's a good idea
to learn enough to be comfortable around them.


### Reflog

The Git reflog records information about how [refs][gb-refs] are created,
deleted or modified.

When you're rewriting history, this tends to happen a lot. The commit hash at
the `HEAD` of your branch changes with every rebase session and the reflog acts
as an audit log of your recent rewrites.

If you find yourself with a mangled branch after a series of rebases, walk
through the reflog and look for the last healthy branch head. You can simply
reset to the corresponding commit hash to recover your branch:

```
(topic)$ git reflog
...
# pick a commit hash you believe in, e.g. 1ba0ff3
(topic)$ git reset --hard 1ba0ff3
```

Should you do this by mistake, you can go always back to the reflog to re-reset
to where you were.

[gb-refs]: https://git-scm.com/book/en/v2/Git-Internals-Git-References


## Strategies

### Private/public

What happens before your branch is pushed is nobody's business but your own.

When you're doing development work, there will be a number of false starts,
small mistakes, accidental commits of broken code, etc. That's fine. Source
control is just backup at this point.

But once you invite other people to review, you want that branch to look
beautiful, logically sequenced, and internally consistent.

Pay attention to where you are in the process and where your changes are
going. You can safely put off getting too fancy with commit partitioning and
messages until just before you push your branch.


### Small, localized scratch commits

A key realization: it's _always_ easier to squash commits than to split them.

So when working on something that will likely need rewriting later, prefer small
commits with changes pertaining to a single file only. While this is terrible
advice in the general case, when preparing for a rewrite it makes life much
easier with regard to the order and composition of commits.

I even tend to commit small changes that break the build, just to make it easier
to shuffle everything around into a consistent series of commits later.


### Incremental rewrites

A single rebase session is all or nothing.

The `git rebase` command is a wonderful machine. At any point, if you get stuck
or make a mistake, you can run `git rebase --abort` to get back to a safe state.

But that also means you lose any valuable changes you've made since the rebase
session started. You may have reworded a couple of commit messages eloquently,
reordered and squashed two commits, resolving some conflicts on the way, before
you decided to give up and abort the rebase.

That's why it's a good idea to start every rebase session with a clear purpose
and stick to it. Run through the rebase cycle for every single improvement idea,
and abort as soon as you have any doubts.

If something goes wrong, you'll only lose the work put into trying that thing.

It can also be useful to do what-if rewrites to validate some ideas before doing
the actual rebase, e.g. "will these two commits move without conflict?" or "can
I drop this commit without breaking the branch?"


### Separate rewriting branch

Even if the history on your topic branch looks horrendous, the branch `HEAD` is
usually still working and self-consistent.

Sometimes rewrites alter the semantics of the changeset intentionally, sometimes
by accident. It can be a nice safety measure to create a new branch specifically
for history cleanup, and keep the original branch around, to make sure you have
a working starting point if all goes wrong:

```
(topic)$ git switch -c topic-rewrite
(topic-rewrite)$ git rebase -i main
...
(topic-rewrite)$ git rebase -i main
...
(topic-rewrite)$ git rebase -i main
...

# oh no! we're beyond repair
(topic-rewrite)$ git switch topic
```

And if all should go well, you can easily compare the rewritten branch to the
original branch:

```
(topic-rewrite)$ git diff topic
(topic-rewrite)$
```

If you've only modified the history, the content diff should be empty.

If the rewrite branch eventually turns out nicer than the original, you can
overwrite the original branch using:

```
(topic-rewrite)$ git branch -M topic
```

This discards the old `topic` history in favor of the new.
