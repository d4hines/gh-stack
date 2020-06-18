# gh-stack [![Check if compilation works; no tests yet!](https://api.travis-ci.org/timothyandrew/gh-stack.svg?branch=master&status=passed)](https://travis-ci.org/timothyandrew/gh-stack)

- [Usage](#usage)
- [Strategy](#strategy)
- [Disclaimer](#disclaimer)

---

I use this tool to help managed stacked pull requests on Github, which are notoriously difficult to manage manually. Here are a few examples:

- https://unhashable.com/stacked-pull-requests-keeping-github-diffs-small
- https://stackoverflow.com/questions/26619478/are-dependent-pull-requests-in-github-possible
- https://gist.github.com/Jlevyd15/66743bab4982838932dda4f13f2bd02a

This tool assumes that:

- All PRs in a single "stack" all have a unique identifier in their title (I typically use a Jira ticket number for this).
- All PRs in the stack live in a single GitHub repository.
- All remote branches that these PRs represent have local branches named identically.

WIt then looks for all PRs containing this containing this identifier and builds a dependency graph in memory. This can technically support a "branched stack" instead of a single chain, but I haven't really tried the latter style. With this graph built up, the tool can:

- Add a markdown table to the PR description (idempotently) of each PR in the stack describing _all_ PRs in the stack.
- Log a simple list of all PRs in the stack (+ dependencies) to stdout.
- Automatically update the stack + push after making local changes.

## Usage

```bash
$ export GHSTACK_OAUTH_TOKEN='<personal access token>'

$ gh-stack

USAGE:
    gh-stack <SUBCOMMAND>

FLAGS:
    -h, --help    Prints help information

SUBCOMMANDS:
    annotate      Annotate the descriptions of all PRs in a stack with metadata about all PRs in the stack
    autorebase    Rebuild a stack based on changes to local branches and mirror these changes up to the remote
    log           Print a list of all pull requests in a stack to STDOUT
    rebase        Print a bash script to STDOUT that can rebase/update the stack (with a little help)

# Idempotently add a markdown table summarizing the stack
# to the description of each PR in the stack.
$ gh-stack annotate 'stack-identifier'

# Same as above, but precede the markdown table with the 
# contents of `filename.txt`.
$ gh-stack annotate 'stack-identifier' -p filename.txt

# Print a description of the stack to stdout.
$ gh-stack log 'stack-identifier'

# Automatically update the entire stack, both locally and remotely.
# WARNING: This operation modifies local branches and force-pushes.
$ gh-stack autorebase 'stack-identifier' -C /path/to/repo

# Emit a bash script that can update a stack in the case of conflicts.
# WARNING: This script could potentially cause destructive behavior.
$ gh-stack rebase 'stack-identifier'
```
  
## Strategy

This is a quick summary of the strategy the `autorebase` subcommand uses:

1. Find the `merge_base` between the local branch of the first PR in the stack and the branch it merges into (usually `develop`). This forms the boundary for the initial cherry-pick.
2. Check out the commit/ref that the first PR in the stack merges into (usually `develop`). We're going to cherry-pick the entire stack onto this commit.
3. Cherry-pick all commits from the first PR (stopping at the cherry-pick boundary calculated in 1.) onto `HEAD`.
4. Move the _local_ branch for the first PR so it points at `HEAD`.
5. The _remote tracking_ branch for the first PR becomes the next cherry-pick boundary.
6. Repeat steps 3-5 for each subsequent PR until all PRs have been cherry-picked over.
7. Push all refs at once by passing multiple refspecs to a single invocation of `git push -f`.

## Disclaimer

Use at your own risk (and make sure your git repository is backed up), especially because:

- This tool works for my specific use case, but has _not_ been extensively tested.
- I've been writing Rust for all of 3 weeks at this point.
- The `autorebase` command is in an experimental state; there are possibly edge cases I haven't considered.