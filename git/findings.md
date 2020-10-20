# git useful tips and tricks

### File in repo but ignoring local changes
If you have a file you need to keep in your repo but you need to ignore local changes to it, use:
git update-index --skip-worktree FILENAME
https://stackoverflow.com/questions/9794931/keep-file-in-a-git-repo-but-dont-track-changes
https://stackoverflow.com/questions/13630849/git-difference-between-assume-unchanged-and-skip-worktree#

### Gitignore
If a file is checked in to the repo but then the gitignore is updated to ignore that file later, the file will not be ignored
as git will not ignore a file which is already being tracked. Remove the cache then re-add all the files after the gitignore is updated.
This will remove the files in the remote repo that are being ignored.