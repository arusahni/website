.. title: How Git Checkout's Previous Branch Shortcut Works Under the Hood
.. slug: git-revisions
.. date: 2021-04-26 18:02:51 UTC-04:00
.. tags: tech
.. link: 
.. description: Understanding how `git checkout -` works under the hood.

One piece of Git shorthand I use all the time is `git checkout -`. Much like
`cd -`, it references the previous item in your history. In the case of `cd`,
it will change your current directory to the previous one you were in. So,

```
$ echo $PWD
/home/aru
$ cd code/git-req
$ echo $PWD
/home/aru/code/git-req
$ cd -
~
$ echo $PWD
/home/aru
```

Git has its own version of this:

```
$ git branch --show-current
master
$ git checkout my-new-feature
Switched to branch 'my-new-feature'
Your branch is up to date with 'origin/my-new-feature'.
$ git checkout -
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
```

Handy! But how does it work? I poked through the files in `.git` (and even
watched the filesystem for changes, but nothing jumped out).

The good news is that Git is open source. So, let's go to the source.

The Git repository is big. Git does a lot, it makes sense. How do I learn where
to look?  Well, I have  bit of prior knowledge. I'm the author of a Git
extension named `git-req`. The hyphen in that name? It is a nifty way to inject
commands into the `git` namespace. So, if I wanted to invoke it, I run `git req`
(no hyphen)! Git's codebase is similarly laid out, with each entry point
represented by its own file.

```
builtin
├── add.c
├── ...
├── branch.c
├── ...
├── checkout.c
├── ...
├── clone.c
├── ...
├── commit.c
└── ...
```

You can see some greatest hits there. What's interesting to us is `checkout.c`
- that's the command we're looking for!

Now, where to look in that file? By convention, most C programs use `main` as
an entrypoint. Let's search for that.

[`checkout_main()`](https://github.com/git/git/blob/311531c9de557d25ac087c1637818bd2aad6eb3a/builtin/checkout.c#L1562)? Check.

Now, how does that method receive arguments? Visually scanning the function
surfaces this command:

```c
/*
 * Extract branch name from command line arguments, so
 * all that is left is pathspecs.
 */
```

I think we're getting warmer. A few lines down, there's a reference to
`parse_branchname_arg()`. Since the hyphen that we're looking for is a
functional stand-in for a branch name, that sounds like a logical place to
look next.  Sure enough, at the top of the function, there's
[the following expression](https://github.com/git/git/blob/311531c9de557d25ac087c1637818bd2aad6eb3a/builtin/checkout.c#L1267-L1268):

```c
if (!strcmp(arg, "-"))
    arg = "@{-1}";
```

Bam. If the argument is a hyphen, then _replace it_ with `@{-1}`.... wat. Let's
[check the docs](https://git-scm.com/docs/git-checkout#Documentation/git-checkout.txt-ltbranchgt):

> Branch to checkout; if it refers to a branch (i.e., a name that, when
> prepended with "refs/heads/", is a valid ref), then that branch is checked
> out. Otherwise, if it refers to a valid commit, your `HEAD` becomes
> "detached" and you are no longer on any branch (see below for details).  
> You can use the `@{-N}` syntax to refer to the N-th last branch/commit checked
> out using "git checkout" operation. You may also specify `-` which is
> synonymous to `@{-1}`.

... I guess I should've started there. But, this raises another question. How
does Git store this history? What is the terminology for this new syntax?

_a few minutes of grepping later..._

This is a syntax for something called a _reflog_. What's that? Sorry to
disappoint you, but I'm not diving into the code without checking the docs
first. [git-reflog](https://git-scm.com/docs/git-reflog) is what we're looking
for.

> Reference logs, or "reflogs", record when the tips of branches and other
> references were updated in the local repository. Reflogs are useful in
> various Git commands, to specify the old value of a reference. For example,
> `HEAD@{2}` means "where `HEAD` used to be two moves ago",
> `master@{one.week.ago}` means "where `master` used to point to one week ago
> in this local repository", and so on. See gitrevisions for more details.

So, this data is stored in the reflog, and the syntax is a
[`gitrevision`](https://git-scm.com/docs/gitrevisions).

> The construct `@{-<n>}` means the <n>th branch/commit checked out before the
> current one.

There we have it. To recap, `git checkout -` is the same as `git checkout
@{-1}` This more verbose gitrevision syntax unlocks the ability to jump further
back in your history.
