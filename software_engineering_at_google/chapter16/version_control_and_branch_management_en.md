## PART IV

# Tools

#### CHAPTER 16

## Version Control and Branch Management

###### Written by Titus Winters

###### Edited by Lisa Carey

Perhaps no software engineering tool is quite as universally adopted throughout the
industry as version control. One can hardly imagine any software organization larger
than a few people that doesn’t rely on a formal Version Control System (VCS) to
manage its source code and coordinate activities between engineers.

In this chapter, we’re going to look at why the use of version control has become such
an unambiguous norm in software engineering, and we describe the various possible
approaches to version control and branch management, including how we do it at
scale across all of Google. We’ll also examine the pros and cons of various
approaches; although we believe everyone should use version control, some version
control policies and processes might work better for your organization (or in general)
than others. In particular, we find “trunk-based development” as popularized by
DevOps<sup>1</sup>  (one repository, no dev branches) to be a particularly scalable policy
approach, and we’ll provide some suggestions as to why that is.

### What Is Version Control?

```markdown
This section might be a little basic for many readers: use of version control is, after all, fairly ubiquitous. If you want to skip ahead, we suggest jumping to the section “Source of Truth” on page 334.

```

```
1 The DevOps Research Association, which was acquired by Google between the first draft of this chapter and publication, has published extensively on this in the annual “State of DevOps Report” and the book Accelerate.As near as we can tell, it popularized the terminology trunk-based development.
```



A VCS is a system that tracks revisions (versions) of files over time. A VCS maintains
some metadata about the set of files being managed, and collectively a copy of the
files and metadata is called a repository<sup>2</sup>  (repo for short). A VCS helps coordinate the
activities of teams by allowing multiple developers to work on the same set of files
simultaneously. Early VCSs did this by granting one person at a time the right to edit
a file—that style of locking is enough to establish sequencing (an agreed-upon “which
is newer,” an important feature of VCS). More advanced systems ensure that changes
to a collection of files submitted at once are treated as a single unit (atomicity when a
logical change touches multiple files). Systems like CVS (a popular VCS from the 90s)
that didn’t have this atomicity for a commit were subject to corruption and lost
changes. Ensuring atomicity removes the chance of previous changes being overwrit‐
ten unintentionally, but requires tracking which version was last synced to—at com‐
mit time, the commit is rejected if any file in the commit has been modified at head
since the last time the local developer synced. Especially in such a change-tracking
VCS, a developer’s working copy of the managed files will therefore need metadata of
its own. Depending on the design of the VCS, this copy of the repository can be a
repository itself, or might contain a reduced amount of metadata—such a reduced
copy is usually a “client” or “workspace.”

This seems like a lot of complexity: why is a VCS necessary? What is it about this sort
of tool that has allowed it to become one of the few nearly universal tools for software
development and software engineering?

Imagine for a moment working without a VCS. For a (very) small group of distributed 

developers working on a project of limited scope without any understanding
of version control, the simplest and lowest-infrastructure solution is to just pass
copies of the project back and forth. This works best when edits are nonsimultaneous
(people are working in different time zones, or at least with different working hours).
If there’s any chance for people to not know which version is the most current, we
immediately have an annoying problem: tracking which version is the most up to
date. Anyone who has attempted to collaborate in a non-networked environment will
likely recall the horrors of copying back-and-forth files named Presentation v5 - final

redlines - Josh’s version v2. And as we shall see, when there isn’t a single agreed-upon
source of truth, collaboration becomes high friction and error prone.

Introducing shared storage requires slightly more infrastructure (getting access to
shared storage), but provides an easy and obvious solution. Coordinating work in a
shared drive might suffice for a while with a small enough number of people but still
requires out-of-band collaboration to avoid overwriting one another’s work. Further,
working directly in that shared storage means that any development task that doesn’t

```
2 Although the formal idea of what is and is not a repository changes a bit depending on your choice of VCS,and the terminology will vary.
```

**328 | Chapter 16: Version Control and Branch Management**

keep the build working continuously will begin to impede everyone on the team—if
I’m making a change to some part of this system at the same time that you kick off a
build, your build won’t work. Obviously, this doesn’t scale well.

In practice, lack of file locking and lack of merge tracking will inevitably lead to colli‐
sions and work being overwritten. Such a system is very likely to introduce out-of-
band coordination to decide who is working on any given file. If that file-locking is
encoded in software, we’ve begun reinventing an early-generation version control like
RCS (among others). After you realize that granting write permissions a file at a time
is too coarse grained and you begin wanting line-level tracking—we’re definitely rein‐
venting version control. It seems nearly inevitable that we’ll want some structured
mechanism to govern these collaborations. Because we seem to just be reinventing
the wheel in this hypothetical, we might as well use an off-the-shelf tool.

##### Why Is Version Control Important?

While version control is practically ubiquitous now, this was not always the case. The
very first VCSs date back to the 1970s (SCCS) and 1980s (RCS)—many years later
than the first references to software engineering as a distinct discipline. Teams par‐
ticipated in “the multiperson development of multiversion software” before the
industry had any formal notion of version control. Version control evolved as a
response to the novel challenges of digital collaboration. It took decades of evolution
and dissemination for reliable, consistent use of version control to evolve into the
norm that it is today.<sup>3</sup>  So how did it become so important, and, given that it seems
like a self-evident solution, why might anyone resist the idea of VCS?

Recall that software engineering is programming integrated over time; we’re drawing
a distinction (in dimensionality) between the instantaneous production of source
code and the act of maintaining that product over time. That basic distinction goes a
long way to explaining the importance of, and hesitation toward, VCS: at the most
fundamental level, version control is the engineer’s primary tool for managing the
interplay between raw source and time. We can conceptualize VCS as a way to extend
a standard filesystem. A filesystem is a mapping from filename to contents. A VCS
extends that to provide a mapping from (filename, time) to contents, along with the
metadata necessary to track last sync points and audit history. Version control makes
the consideration of time an explicit part of the operation: unnecessary in a program‐

```
3 Indeed, I’ve given several public talks that use “adoption of version control” as the canonical example of how the norms of software engineering can and do evolve over time. In my experience, in the 1990s, version control was pretty well understood as a best practice but not universally followed. In the early 2000s, it was still common to encounter professional groups that didn’t use it. Today, the use of tools like Git seems ubiquitous even among college students working on personal projects. Some of this rise in adoption is likely due to better user experience in the tools (nobody wants to go back to RCS), but the role of experience and changing norms is significant.
```

ming task, critical in a software engineering task. In most cases, a VCS also allows for
an extra input to that mapping (a branch name) to allow for parallel mappings; thus:

```
VCS(filename, time, branch) => file contents
```

In the default usage, that branch input will have a commonly understood default: we
call that “head,” “default,” or “trunk” to denote main branch.

The (minor) remaining hesitation toward consistent use of version control comes
almost directly from conflating programming and software engineering—we teach
programming, we train programmers, we interview for jobs based on programming
problems and techniques. It’s perfectly reasonable for a new hire, even at a place like
Google, to have little or no experience with code that is worked on by more than one
person or for more than a couple weeks. Given that experience and understanding of
the problem, version control seems like an alien solution. Version control is solving a
problem that our new hire hasn’t necessarily experienced: an “undo,” not for a single
file but for an entire project, adding a lot of complexity for sometimes nonobvious
benefits.

In some software groups, the same result plays out when management views the job
of the techies as “software development” (sit down and write code) rather than “soft‐
ware engineering” (produce code, keep it working and useful for some extended
period). With a mental model of programming as the primary task and little under‐
standing of the interplay between code and the passage of time, it’s easy to see some‐
thing described as “go back to a previous version to undo a mistake” as a weird, high-
overhead luxury.

In addition to allowing separate storage and reference to versions over time, version
control helps us bridge the gap between single-developer and multideveloper pro‐
cesses. In practical terms, this is why version control is so critical to software engi‐
neering, because it allows us to scale up teams and organizations, even though we use
it only infrequently as an “undo” button. Development is inherently a branch-and-
merge process, both when coordinating between multiple developers or a single
developer at different points in time. A VCS removes the question of “which is more
recent?” Use of modern version control automates error-prone operations like track‐
ing which set of changes have been applied. Version control is how we coordinate
between multiple developers and/or multiple points in time.

Because VCS has become so thoroughly embedded in the process of software engi‐
neering, even legal and regulatory practices have caught up. VCS allows a formal
record of every change to every line of code, which is increasingly necessary for satis‐
fying audit requirements. When mixing between in-house development and appro‐
priate use of third-party sources, VCS helps track provenance and origination for
every line of code.

In addition to the technical and regulatory aspects of tracking source over time and
handling sync/branch/merge operations, version control triggers some nontechnical
changes in behavior. The ritual of committing to version control and producing a
commit log is a trigger for a moment of reflection: what have you accomplished since
your last commit? Is the source in a state that you’re happy with? The moment of
introspection associated with committing, writing up a summary, and marking a task
complete might have value on its own for many people. The start of the commit pro‐
cess is a perfect time to run through a checklist, run static analyses (see Chapter 20),
check test coverage, run tests and dynamic analysis, and so on.

Like any process, version control comes with some overhead: someone must config‐
ure and manage your version control system, and individual developers must use it.
But make no mistake about it: these can almost always be pretty cheap. Anecdotally,
most experienced software engineers will instinctively use version control for any
project that lasts more than a day or two, even for a single-developer project. The
consistency of that result argues that the trade-off in terms of value (including risk
reduction) versus overhead must be a pretty easy one. But we’ve promised to
acknowledge that context matters and to encourage engineering leaders to think for
themselves. It is always worth considering alternatives, even on something as funda‐
mental as version control.

In truth, it’s difficult to envision any task that can be considered modern software
engineering that doesn’t immediately adopt a VCS. Given that you understand the
value and need for version control, you are likely now asking what type of version
control you need.

##### Centralized VCS Versus Distributed VCS

At the most simplistic level, all modern VCSs are equivalent to one another: so long
as your system has a notion of atomically committing changes to a batch of files,
everything else is just UI. You could build the same general semantics (not workflow)
of any modern VCS out of another one and a pile of simple shell scripts. Thus, argu‐
ing about which VCS is “better” is primarily a matter of user experience—the core
functionality is the same, the differences come in user experience, naming, edge-case
features, and performance. Choosing a VCS is like choosing a filesystem format:
when choosing among a modern-enough format, the differences are fairly minor, and
the more important question by far is the content you fill that system with and the
way you use it. However, major architectural differences in VCSs can make configura‐
tion, policy, and scaling decisions easier or more difficult, so it’s important to be
aware of the big architectural differences, chiefly the decision between centralized or
decentralized.

**Centralized VCS**

In centralized VCS implementations, the model is one of a single central repository
(likely stored on some shared compute resource for your organization). Although a
developer can have files checked out and accessible on their local workstation, opera‐
tions that interact on the version control status of those files need to be communica‐
ted to the central server (adding files, syncing, updating existing files, etc.). Any code
that is committed by a developer is committed into that central repository. The first
VCS implementations were all centralized VCSs.

Going back to the 1970s and early 1980s, we see that the earliest of these VCSs, such
as RCS, focused on locking and preventing multiple simultaneous edits. You could
copy the contents of a repository, but if you wanted to edit a file, you might need to
acquire a lock, enforced by the VCS, to ensure that only you are making edits. When
you’ve completed an edit, you release the lock. The model worked fine when any
given change was a quick thing, or if there was rarely more than one person that
wanted the lock for a file at any given time. Small edits like tweaking config files
worked OK, as did working on a small team that either kept disjointed working hours
or that rarely worked on overlapping files for extended periods. This sort of simplistic
locking has inherent problems with scale: it can work fine for a few people, but has
the potential to fall apart with larger groups if any of those locks become contended.<sup>4</sup> 

As a response to this scaling problem, the VCSs that were popular through the 90s
and early 2000s operated at a higher level. These more modern centralized VCSs
avoid the exclusive locking but track which changes you’ve synced, requiring your
edit to be based on the most-current version of every file in your commit. CVS wrap‐
ped and refined RCS by (mostly) operating on batches of files at a time and allowing
multiple developers to check out a file at the same time: so long as your base version
contained all of the changes in the repository, you’re allowed to commit. Subversion
advanced further by providing true atomicity for commits, version tracking, and bet‐
ter tracking for unusual operations (renames, use of symbolic links, etc.). The central‐
ized repository/checked-out client model continues today within Subversion as well
as most commercial VCSs.

**Distributed VCS**

Starting in the mid-2000s, many popular VCSs followed the Distributed Version
Control System (DVCS) paradigm, seen in systems like Git and Mercurial. The

```
4 Anecdote: To illustrate this, I looked for information on what pending/unsubmitted edits Googlers had outstanding for a semipopular file in my most recent project. At the time of this writing, 27 changes are pending,12 from people on my team, 5 from people on related teams, and 10 from engineers I’ve never met. This is basically working as expected. Technical systems or policies that require out-of-band coordination certainly
don’t scale to 24/7 software engineering in distributed locations.
```

primary conceptual difference between DVCS and more traditional centralized VCS
(Subversion, CVS) is the question: “Where can you commit?” or perhaps, “Which
copies of these files count as a repository?”

A DVCS world does not enforce the constraint of a central repository: if you have a
copy (clone, fork) of the repository, you have a repository that you can commit to as
well as all of the metadata necessary to query for information about things like revi‐
sion history. A standard workflow is to clone some existing repository, make some
edits, commit them locally, and then push some set of commits to another repository,
which may or may not be the original source of the clone. Any notion of centrality is
purely conceptual, a matter of policy, not fundamental to the technology or the
underlying protocols.

The DVCS model allows for better offline operation and collaboration without inher‐
ently declaring one particular repository to be the source of truth. One repository
isn’t necessary “ahead” or “behind” because changes aren’t inherently projected into a
linear timeline. However, considering common usage, both the centralized and DVCS
models are largely interchangeable: whereas a centralized VCS provides a clearly
defined central repository through technology, most DVCS ecosystems define a cen‐
tral repository for a project as a matter of policy. That is, most DVCS projects are
built around one conceptual source of truth (a particular repository on GitHub, for
instance). DVCS models tend to assume a more distributed use case and have found
particularly strong adoption in the open source world.

Generally speaking, the dominant source control system today is Git, which imple‐
ments DVCS.<sup>5</sup>  When in doubt, use that—there’s some value in doing what everyone
else does. If your use cases are expected to be unusual, gather some data and evaluate
the trade-offs.

Google has a complex relationship with DVCS: our main repository is based on a
(massive) custom in-house centralized VCS. There are periodic attempts to integrate
more standard external options and to match the workflow that our engineers (espe‐
cially Nooglers) have come to expect from external development. Unfortunately,
those attempts to move toward more common tools like Git have been stymied by the
sheer size of the codebase and userbase, to say nothing of Hyrum’s Law effects tying
us to a particular VCS and interface for that VCS.<sup>6</sup>  This is perhaps not surprising:
most existing tools don’t scale well with 50,000 engineers and tens of millions of

```
5 Stack Overflow Developer Survey Results, 2018.
6 Monotonically increasing version numbers, rather than commit hashes, are particularly troublesome. Many systems and scripts have grown up in the Google developer ecosystem that assume that the numeric ordering of commits is the same as the temporal order—undoing those hidden dependencies is difficult.
```

commits.<sup>7</sup> The DVCS model, which often (but not always) includes transmission of
history and metadata, requires a lot of data to spin up a repository to work out of.

In our workflow, centrality and in-the-cloud storage for the codebase seem to be criti‐
cal to scaling. The DVCS model is built around the idea of downloading the entire
codebase and having access to it locally. In practice, over time and as your organiza‐
tion scales up, any given developer is going to operate on a relatively smaller percent‐
age of the files in a repository, and a small fraction of the versions of those files. As we
grow (in file count and engineer count), that transmission becomes almost entirely
waste. The only need for locality for most files occurs when building, but distributed
(and reproducible) build systems seem to scale better for that task as well (see
Chapter 18).

##### Source of Truth

Centralized VCSs (Subversion, CVS, Perforce, etc.) bake the source-of-truth notion
into the very design of the system: whatever is most recently committed at trunk is
the current version. When a developer goes to check out the project, by default that
trunk version is what they will be presented with. Your changes are “done” when they
have been recommitted on top of that version.

However, unlike centralized VCS, there is no inherent notion of which copy of the
distributed repository is the single source of truth in DVCS systems. In theory, it’s
possible to pass around commit tags and PRs with no centralization or coordination,
allowing disparate branches of development to propagate unchecked, and thus risk‐
ing a conceptual return to the world of Presentation v5 - final - redlines - Josh’s version
v2. Because of this, DVCS requires more explicit policy and norms than a centralized
VCS does.

Well-managed projects using DVCS declare one specific branch in one specific repos‐
itory to be the source of truth and thus avoid the more chaotic possibilities. We see
this in practice with the spread of hosted DVCS solutions like GitHub or GitLab—
users can clone and fork the repository for a project, but there is still a single primary
repository: things are “done” when they are in the trunk branch on that repository.

It isn’t an accident that centralization and Source of Truth has crept back into the
usage even in a DVCS world. To help illustrate just how important this Source of
Truth idea is, let’s imagine what happens when we don’t have a clear source of truth.

```
7 For that matter, as of the publication of the Monorepo paper, the repository itself had something like 86 TB of data and metadata, ignoring release branches. Fitting that onto a developer workstation directly would be...challenging.
```

