This repository defines the process and tools to work on a project that is kept
up-to-date with a remote (so-called upstream) repository.

## The workflow

The workflow consists of a set of features where each feature is built on top of
another. Each new feature is checked out from its parent's HEAD. The last feature
branch holds all previous feature's commits. In general, the idea is merge upstream
fast and don't stop working locally until merged.

In a graph, this means:
```mermaid
gitGraph TB:
  commit
  commit
  branch feature1
  commit
  commit
  branch feature2
  commit
  branch feature3
  commit
  commit
  branch feature4
  commit
  commit
  commit
  branch feature5
  commit
```
So, feature 2 rebases on top of feature1, feature3 rebases on top of feature2, and so on ...

When a feature, let's say feature1, enters into reviewing mode due to a merge request,
feature1 will not be sent upstream but a new branch called feature1-reviewing.

```mermaid
gitGraph TB:
  commit
  commit
  branch feature1
  commit
```

Usually, during that reviewing process, new commits might be added or removed, for example
```mermaid
gitGraph TB:
  commit
  commit
  branch feature1
  commit
  branch feature1-reviewing
  commit
  commit
```

The process of generating the final branch will still be the rebasing against all feature
branches, but feature1 is marked as "merging" in the file.

Once all fixups, rebases, etc, of the feature branch going upstream are done, the branch is
marked as "merged", so the `sync` process is done by skipping feature1 but on top
of a new main, the one with feature1-reviewing merged.


## Configuration file
```TOML
[[remotes]]
name = "origin"
url = "git@github.com:fluendo/git-upstream-workflow.git"

[target]
remote = "origin"
branch = "final"

[source]
remote = "origin"
branch = "main"

[[features]]
remote = "origin"
name = "feature1"
pr = "https://github/fluendo/git-upstream-workflow/pull-requests/10"
status = "integrated"

[[features]]
remote = "origin"
name = "feature2"
pr = "https://github/fluendo/git-upstream-workflow/pull-requests/10"
status = "merged"

[[features]]
remote = "origin"
name = "feature3"
status = "pending"
```

The configuration must include the list of remotes under the `[[remotes]]` section. This is useful
when the upstream branch is done in a git provider like GitLab but the development is done in GitHub.

There are two special sections, `[source]` and `[target]`. The `[source]` section defines the branch
the project you want to contribute to uses as the main stable branch.  The `[target]` section defines
the branch that should hold all the features.

The `[[features]]` section defines the list of features you want to include upstream (the `target` branch).
Each feature has the following key/value pairs:
`remote`
: The remote name as listed in the `[[remotes]]` section

`name`
: The name of the branch

`pr`
: The URL used for the merge-request/pull-request

`status`
: This defines how `guw` should handle the `sync` process. In case of `pending`, this branch is still not
requested to be integrated on the upstream project. In case of `merging`, the branch has already opened a
merge-request/pull-request and is waiting for the community to be reviewed. In case of `merged`, the branch
has already being merged but the other features depending on this have not being rebased yet. In case of
`integrated`, the dependant features have been rebased already and the actual feature is no longer considered
in any process.

## Usage
First you need to install the package
```
pip install git+https://github.com/fluendo/git-upstream-workflow.git
```
After that, you will the command `guw`, with several options, the most used one is to sync branches based on
a configuration, simply do:
```
guw -l debug example1.toml sync -l -b
```
This will generate a temporary folder, fetch each remote, checkout each branch and finally rebase each feature.
Note the `-l` and `-b` option. The former specifies a local-only process, no branch will be pushed. The latter
does a backup branch of each feature processed.

## Recommendations
* Never push into the `target` branch by other means but through `guw`, otherwise your new commits will
  be lost after a `sync` process.
* Once a feature branch enters into an upstream reviewing process, use a -reviewing branch and never commit
  changes back to the original feature branch.
