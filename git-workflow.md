## A (slightly less) simple (but more versatile) git branching model

Heavily borrowed from: https://gist.github.com/jbenet/ee6c9ac48068889b0912

The idea is to have an always deployable Master, like in the original simple workflow.

We simply introduce a layer of QA/Staging environment.

### Git Setup

```bash
# autosetup rebase so that pulls rebase by default
git config --global branch.autosetuprebase always

# if you already have branches (made before `autosetuprebase always`)
git config branch.<branchname>.rebase true
```

### Git Goodies
```bash
git config --global log.decorate short
git config --global color.ui auto
git config --global color.interactive auto
git config --global color.diff auto
git config --global color.branch auto
git config --global color.status auto
git config --global pager.status true
git config --global pager.show-branch true
git config --global format.numbered auto

git config --global alias.st status
git config --global alias.ci commit
git config --global alias.co checkout
git config --global alias.ru "remote update"
git config --global alias.br branch
git config --global alias.cam "commit -a -m"
git config --global alias.mnf "merge --no-ff"


git config --global alias.undo "reset --hard HEAD^1"
git config --global alias.tree "log --graph --decorate --pretty=oneline --abbrev-commit --all"
git config --global alias.hotfix "!git checkout master && git merge --no-ff --no-edit hotfix && git checkout staging && git merge --no-ff --no-edit hotfix && git branch -d hotfix"

```

### For the repo "owner" first time setup

```bash
git checkout master
git checkout -b staging
git push origin staging
```

### How to write commit messages

- write "card #XXX - [description of the commit]", where XXX is the Trello card number
- If it's a correction to an existing bug that was rejected, same syntax apply.
- If it doesn't fix an existing issue but references one ( should be _very_ rare ), "[description] - ref #XXXX" 

### Working on a BUG
```bash
git fetch --prune # deletes local tracked branches deleted from origin 
git checkout staging
git pull --rebase
git checkout -b bugXXX
# [Work on the ticket] 
git commit -am "my changes" 
git fetch origin
git rebase origin/staging # HANDLE conflicts here.
# IF YOU HAVE MORE THAN ONE COMMIT (eg. conflict resolution commit), squash them:
git rebase -i origin/staging
git checkout staging
git rebase bugXXX
git push origin staging
```

### Working on a RFC
```bash
git fetch --prune # deletes local tracked branches deleted from origin 
git checkout staging
git pull --rebase
git checkout -b rfcXXX
# [Work on the ticket] 
git commit -am "my changes"
# Once the ticket is ready for Code Review: 
git fetch origin
git rebase origin/staging # HANDLE conflicts here.
# IF YOU HAVE MORE THAN ONE COMMIT, squash them:
git rebase -i origin/staging
# Push to github:
git push origin rfcXXX
# [ Go to GITHUB and create a PULL REQUEST ]
# Have a teammate code review and approve your code.
# If rejected, go back to editing. Keep your commits clean. Repeat until accepted.
# Once the Reviewer approves, 
#     he has the authority to merge, close the pull request into staging and delete the branch.
# If QA finds bugs, treat them as bugXXX.
# [ IF THE BRANCH CANNOT BE AUTOMATICALLY MERGED ]
git checkout rfcXXX
git fetch origin
git rebase origin/staging # HANDLE conflicts here.
# SQUASH THE CONFLICT RESOLUTION COMMIT:
git rebase -i origin/staging
git push --force origin rfcXXX
# Reviewer merges the pull request on Github and DELETES THE BRANCH.
```

### Preparing a Production Release
```bash
git fetch
git checkout master
git pull --rebase master
git merge --no-ff origin/staging
git push origin master
```


### How to handle hotfixes

```bash
# Hotfixes should be a branch of master then merged into master and rebased into staging.

git checkout staging

# Commit the bug fix and note the commit hash

git checkout master
git cherry-pick [hash-of-the-commit-in-staging]
git push origin master
git push origin staging

```
That will provide clean, worry free hotfixes!

### Someone just pushed master/staging and my HEAD is no longer up to date. What do I do?

```bash
git fetch
git checkout [staging/master]
git rebase -p origin/[staging/master]
# rebase -p preserve merges instead of applying directly the commits.
```

### A dev merged a crap feature, holding up the production release. What do I do?

Sometimes, everything fails: the code review didn't work and a lemon got merged into Staging. QA only picks it up later, you already have 5 commits on top of this crappy merge. It's time to **cherry-pick** your production release!

```bash
git checkout master
git checkout -b release
# To cherry pick bug fixes ( aka not merged in features )
    git cherry-pick [non-merge commit1] [non-merge commit2] [non-merge commit3]
    #eg: git cherry-pick 585db93 f50c455 e7322a4
# To cherry pick RFC merges
    git cherry-pick -m1 [merge commit 1]
    #eg git cherry-pick -m1 5fae09c

# Make sure to ONLY TAKE COMMITS ON STAGING AND NOT FEATURE BRANCHES.
# Once you cherry picked necessary commit:
git checkout master
git merge --no-ff release
git branch -d release
```

Then whenever staging is finally fixed, do a regular production release:

```bash
git fetch
git checkout master
git merge --no-ff origin/staging
```

### I keep forgetting to delete my merged local branches. It's a mess now, what do I do?

This will get rid of all merged branch.

```bash
git branch --merged | grep -v "\*" | grep -v "staging" | xargs -n 1 git branch -d
```
