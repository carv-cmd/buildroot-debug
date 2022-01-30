# buildroot-debug
Debug process to get buildroot stable for production.

## Brief overview of `git`.

Git is a ***distributed*** version control system. 

Although the more intuitive description would be a distributed database for code.

One of Gits primary goals is to remove single points of failure in a VCS. (a **single** copy lives on a **single** server)
 * This means anyone that has `git clone(d)` **any** *repository under Git VCS* has the complete history, source code, info files, etc.
 * You can see the most spectcular example of this by cloning the Linux Kernel to your local machine:
 ```bash 
 cd ~/tmp && git clone https://github.com/torvalds/linux.git
 cd linux && ls -la
 git log  # 1,072,XXX commits lol
 ```
---
Listed below are a few places you can `git (clone|pull|push)` *to and from*:
| Location | Examples | Use Cases |
|---|---|---|
| Hosted Git | Github.com, Bitbucket, GitLabs | collaboration/common_upstreams |
| Local Machine | Coworker on LAN, buildroot server | dev/test/staging |
| Cloud services | AWS EC2 | dev/test/staging/**builds** |
| Physical Media | SSD/HDD, `scp builder@private-nas.net` | hard_backups | 

---
While I listed *Physical Media* as hard_backups, in reality anywhere that clone exists is technically a backup.

So regardless of where it comes from we get the entire version history for that repository.

Exceptions to that clause exist when a *.gitignore* file has been added to any directory under Git VCS.
 * Whenever a *.gitignore* exists, any names/extensions/globs included in that file are ignored by git.
 * A top level directory can have the repos main *.gitignore* then additional ignore files can be added to the heiarchy when needed.
 * On Linux can gitignore things like **'\*.swp'**, **'\*~'**. 
 * In applications ignore  **'logs/\*'**, **__pycache__**.
 * See all *buildroot/monorepo* ignored files by grepping for all it's *.gitignore* files and `cat` their contents.

On the flip side consider a ***.env*** entry that's always added to the top level *.gitignore*.
 * Values associated with a particular hosts runtime can be stored locally in *.env* files.
 * Such as: environment variables, domains, hardware/software presets, API keys, tokens, auth, etc.

---
What are the cons of git?
 * The distributed nature is a blessing and a curse. 
 * Problems can arise when these full histories, stored in different locations, become out of sync (think CAP).
 * Luckily with some care its a relatively easy fix as we're essentially just merging historical diffs together.

---
### Basic Git Workflows 
These concepts stay valid as you scale your workflows. 
```bash 
cd ~/tmp

#[Create||UseExisting] directory and initialize a '.git' directory in it.
REPO_NAME='${1:-monorepo}'

#[git init] Fail-Fast
if ! git init "$REPO_NAME"; then 
  exit 1
fi

#[.git] Note the '.git' directory in $REPO_NAME, this is where our complete history resides.
cd $REPO_NAME && ls -la

#[new-files] Touch a README.md and check the `git status`
touch 'README.md' && git status 

#[git add] You can select files or add all with '.' then check the git status
# You must add all tracked changes before trying to commit.
{ git add . || git add README.md; } && git status

# By now you have created a version controlled directory and added new files to be tracked.
#[git commit] Now commit your changes to the git history.
git commit -m 'initial commit'
git status && git log

#[modify&&new] Now modify the README.md and add a new file.
echo "Modifications" >> README.md && git status
touch main.py && git status

# Repeat the steps above and you can see how Git manages changes to a repository.
git add . && git status
git commit -m 'Updated README.md. Added main.py' && git status
git log
```

## CLEANUP PREPARATION 
### BACKUPS ARE INDUSTRY BEST PRACTICE. 
### DO NOT SKIP THIS STEP
---
***Before*** making any changes; **create a directory on your local machine to store the following**;
 * Copy *local buildroot monorepo* with `scp buildroot:/home/wirediq/monorepo` or another remote-copy utitlity.
 * Clone *upstream monorepo* with `git clone git@github.com:wirediq/monorepo.git`.
 * We're just creating initial checkpoints independent of the VCS changes we're about to make.

#### Git cloning the upstream repo gathers the current upstream state and makes a local copy of it.
 * This creates a backup of our upstream *monorepo* and it's state ***before*** we go pushing any changes to it.

#### When creating our local copies compare the following:
 * In the *local-copy buildroot:/monorepo* run `git status`.
 * In the *actual-source buildroot:/monorepo* run `git status`.

#### If they dont match up, copy the necessary *tracked* files to your local machine until the `git status` outputs match on both. 
 * The goal is to *sync* your **local "backup" state** with the **current state** of *actual buildroot/monorepo*.

---
## Cleaning initial buildroot repository state
Running `git status` will show all *new* or *modified* files that haven't been committed yet.
 * Running `git status` on my local machine *~/Downloads/monorepo-local-bakup* will show the monorepo state we were looking at on Thursday.

The majority of these files were recipes and config files, as well as the changes we saw on Github commit history
 * You'll also notice the *untracked* files, this is determined by buildroots *.gitignore* as mentioned above.

Note when we run `git status` in the **actual/backup local buildroot:/monorepo** we see its *1 commit behind the upstream source*.
 * Although the local repository is technically a few commits ahead of the upstream repository as we saw with the local `git status`.
 * The following describes how to merge these histories.

## GIT ADD
Preparing the current local *buildroot:/monorepo*
 * I was informed by Joe that the missing recipe files **should** be present in the upstream repository.
 * Loren verified the config files and confirmed they should be in the upstream repository as well.
 * I additionally confirmed this by the missing *'\*_asterisk'* configs amoung others **not present** in the upstream monorepo config files.
 * A file by the name of *comm* (commit hash) was removed from the tracking list as its used locally.

Then `git add $TRACKING_FILES` until our `git status` shows all changes now tracked and ready for commit.
 * I took screenshots of these git status's for record

## GIT STASH -> PULL -> APPLY -> (VIM|ADD) -> COMMIT -> PUSH
As noted above, our local monorepo is *1 commit behind upstream*
 * We could commit our current local state and attempt to send it upstream, although this isn't ideal.
 * This would likely fail since we don't know if ***merge conflicts** exist nor do we really want to ***git fast-forward*** our history.

Running `git stash push` in our local *buildroot:/monorepo* will *remove* all changes made since last commit
 * Essentially resetting the repo/directory to the state at the last commit. 
 * Note changes ***stashed*** "to the side" not ***deleted***
 * The goal is to have a clean repository when we `git pull` those upstream changes.

### A walkthough of the commands used to remedy this
```bash
# First we need to be in our directory under Git VCS.
cd /home/wirediq/monorepo

# If any stashes exist just note the commit hash so you know which one is new.
git stash list

# This "resets/clears" all changes made since the last commit
git stash push

# Check your stash
git stash list

# View the new git status. 
# You should still see "1 commit behind main" but the all the color(changes) is now gone.
git status 

# Now we can `git pull` the upstream changes into our local repository
git pull

# Running `git stash apply` attempts to merge the stashed state back onto the current(upstream updated) state
git stash apply

# In this case we got a "merge conflict with: .../tools/init.sh".
# Other than that all our previous changes have been merged back into our local repository.
# We can see the color has returned. Also note the 'both modified .../tools/init.sh'
git status

# We need to hand edit the file and to fix the merge conflict
# Git tells you where the conflict is and thats the file you need to edit.
# File path may be wrong here, but I do have the correct path saved on my sys76 host.
vim /home/wirediq/monorepo/openwrt-builder/tools/init.sh 

# Next we remove the conflicting line and the markup done by git.
# Here the choice is arbitrary since that change was present local and upstream. 
# This wont always be the case. For example if one was [ "$VAR"], that line gets deleted.
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
merge conflict ...[ "$VAR" ]...
===============================
merge conflict ...[ "$VAR" ]...
<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<

# Our merge conflicts are now resolved, and we can `git add .../tools/init.sh`
git add ./full/path/to/tools/init.sh 
git status  # .../tools/init.sh should now appear with the greens

# Now we can commit our changes
git commit -m "High level description of the steps taken to resolve repository states"
git status  # shows X commits ahead of main

# We can then `git push` our changes upstream
git push 
git status  # shows our local is up-to-date with our remotes
```

## Attempting AP Build
***The build failed.***

Things I noted watching the build server change realtime:
 1. A new directory is created with the datestamp when the build runs
    * These should have time added so same-day builds dont overwrite eachother: `$(date +'%F-%H-%M')`
 2. **midbrain.bin** is not getting created during the build process.
 3. **ap.bin** is getting created, only its not an executable, just a regular file.
 4. The latest directory is just a pointer to a dated directory, this is not automatically updated.
    * We put ***latestrr

Things I noted reading the build log:
 1. .../compile.sh (Permission denied error)

With this in mind I looked for the most pulled the image from 22-01-25 and flashed that to an AP
 * I ran through the tests, queue accepted both the new SSID and modifying the existing SSID
 * 



















