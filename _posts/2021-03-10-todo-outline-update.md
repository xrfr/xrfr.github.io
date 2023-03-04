---
title: An Outline Extension for Todo.txt
---

# Background

[todo.txt](https://github.com/todotxt/todo.txt) is a way of organizing your tasks by plain text files.

[todo.sh](https://github.com/todotxt/todo.txt-cli) is a way to manage such files from the command line.

# Introduction

`outline` is a extension to the `todo.sh` tool that converts outlines into the
`todo.txt` format, using indents to imply either dependency or inheritance
relationships.

For example, it converts this (sample.ol.txt):

```
(A)
  +project.whales
    write report on whales
      research phase
        reserve books about whales at the library
        read book 1
        read book 2

      complete analysis
        find data on whale migration patterns
        import data into computer
        do the magic

      write up findings

@buy
  (B)
    game console
    cables
    controller

  (C)
    television
```

into this (sample.todo.txt):

```
(A) +project.whales reserve books about whales at the library; research phase; write report on whales
(B) @buy game console
(B) @buy cables
(B) @buy controller
(C) @buy television
```

Notice that tasks that are children of projects/contexts/priorities or a
combination of them inherit them.
Alternatively, tasks that are children of other tasks make them dependent.
In this case, the task 'read book 1' won't show up in the todo file until the
'reserve books...' task is complete, and tasks under 'complete analysis...'
won't show up until all the research phase tasks are done.

This extends `todo.sh` to serve as an organizing tool for task breakdowns or
project management.

# Updates

A few additions have been made to the format of `*.ol.txt` files over time.

1. First is the divider: `---`.
   Anything after the divider in an `*.ol.txt` file will be ignored.
   This is space for notes and a scratchpad for the outline.

2. Done tasks get collected into the global `done.txt` file in your `todo`
   repository.

3. Tasks now show their parenst in addition to themselves. This is to get around
   the issue of multiple tasks that may have the same 'raw' text.
   Take this outline file for example.

   ```
   Chemistry Essentials
      chapter 1-3 @read due:2029-02-20
      chapter 4-6 @read due:2029-03-05

   Mathematics for adults
      chapter 1-3 @read due:2029-02-20
      chapter 4-6 @read due:2029-03-05
   ```

   In this case, the tasks from the second book would marked as duplicates.
   Instead, we store each task as follows, which ensures that each task has
   unique raw text. It also gives some context to the task.
   ```
   task text; <parent task text>; ... @<contexts>... +<projects>...
   ```
   The information overload is a slight issue, but I find that I read the
   beginning and ending of lines more closely that the middles, so it's not much
   of a problem.


## Properly Deleting Tasks

- [X] when finishing tasks, if the task is archived before `outline` is run
      again, we are adding it back to the `*.todo.txt` file. This can be worked
      around for the time being by completing and removing tasks from both the
      `todo` file and the `ol` file at once.

Let us work on fixing this issue.  So what do we need to do? Whenever we
complete a task, if there is an outline file (`*.ol.txt`) for the todo file we
are editing, we need to propagate the change to the outline file.

Most, if not all of the time when working with outlines, we are not working on
our main todo file `todo.txt` but some other subject specific file, like
`learn.todo.txt`. To complete tasks in these files, we already have a custom
command `df`, that works like the default command `do`, but takes an additional
argument for the file to act on. So for example, if we want to complete task
number $4$ in `learn.todo.txt`, we can run `t df learn 4`.

Thankfully, we have already stubbed out where we need to call into our `outline`
script. We will make a new subcommand called `__df` that we can hook into. It
will accept similar arguments as `df`, so borrowing from the previous example,
running `t df learn 4` will call `t outline __df learn.ol.txt 4`.

`outline` works by keeping all the tasks in a local SQLite database. This is the
source of truth that we use to keep tasks in sync between outline files and text
files. So, the first thing we need to do is look up the task. One of the
invariants we rely on is that all tasks have a unique `raw` text. This is the
text of the task with all projects, contexts, due dates, etc. stripped away.

Once we've parsed out the `raw` text, we can set `completed = True` and sync
this task to the db. This is straigtforward since we already have a function
`sync_task` that takes care of most of the details. We simply parse the line in
of the todo file that we are interested in, and `sync_task` will look the task
by it's `raw` text, and update the `completed` status.

Next, we need to sync the change to the outline file. We find the line in the
outline that that has the matching `raw` text and mark it as complete. After
flushing out some bugs, it seems like everything is working now.


# Reference

- **Repository**: [bitbucket](https://bitbucket.org/sachinrudr/todo.actions.d/src)
- **Stack**: GraphQL, SQLite, python, graphene (python)
