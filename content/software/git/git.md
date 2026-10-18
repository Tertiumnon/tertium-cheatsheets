# Git

## Configure

### Configure user preferences

#### Setup user email

```bash
git config --global user.email "{email}"
```

#### Setup user name

```bash
git config --global user.name "{name}"
```

### Configure line endings

#### Setup all line endings to LF

```bash
git config --global core.autocrlf input
```

### Configure file/folder permissions

#### Don't change file/folder permissions

```bash
git config core.fileMode false
```

## Usage

### Get remote URL's

```bash
git remote -v
```

### Remove current remote origin

```bash
git remote rm origin
```

### Add remote origin via SSH

```bash
git remote add origin ssh://git@{host}:{path}.git
```

### Switching remote URLs from HTTPS to SSH

```bash
git remote set-url origin ssh://git@{host}:{path}.git
```

### Get existing branches

#### Existing branches are listed; the current branch will be highlighted in green and marked with an asterisk

```bash
git branch --list
```

### Get changed files

#### Get changed in last commit files

```bash
git diff --name-only HEAD HEAD~1
```

### Remove file or directory from git repository

#### Remove file from git repository

```bash
git rm --cached {file to remove}
```

#### Remove folder from git repository

```bash
git rm --cached -r {folder to remove}
```

#### Remove commit history in git repository

```bash
git commit --amend -CHEAD
```

#### Push your commits

```bash
git push
```

### Remove multiple branches using pattern

#### Test your pattern

```bash
git branch | grep "{pattern}"
```

#### Remove multiple branches using pattern

```bash
git branch | grep "{pattern}" | xargs git branch -D
```

### Clear all stash

```bash
git stash clear
```

### Clean git repository

- Remove the history

```bash
rm -rf .git
```

- Init new git repository

```bash
git init
git add .
git commit -m "Initial commit"
```

- Add remote origin

```bash
git remote add origin git@github.com:<YOUR ACCOUNT>/<YOUR REPOS>.git
```

- Push

```bash
git push -u --force origin master
```

### Pull another branch without switching

```bash
git fetch origin {branch_name}:{branch_name}
```

### Rename branch

#### Rename active branch

```bash
git branch -m {new branch name}
```

#### Rename another branch

```bash
git branch -m {old branch name} {new branch name}
```

#### Delete old branch and push branch with new name

```bash
git push origin: {old branch name} {new branch name}
```

#### Reset the upstream

```bash
git push origin -u {new branch name}
```

### Tags

#### Add tag to commit

```bash
git tag {tag name} {commit}
```

#### Rename tag locally

```bash
git tag {new tag name} {previous tag name}
git tag -d {previous tag name}
```

#### Rename tag remotely

```bash
git push origin :refs/tags/{previous tag name}
git push origin --tags
```

## Troubleshooting

### Encoding

#### Fix Unicode file names

```bash
git config --global core.quotePath false
```

### Error: Remote ref does not exist

- Error

```bash
error: unable to delete '{branch name}': remote ref does not exist
error: failed to push some refs to 'https://github.com/{repository name}.git'
```

- Recipe: Fix your local Git cache.

```bash
git fetch -p origin
```

## References

- [Git. Documentation. References](https://git-scm.com/docs)

### Changelog

- [Changelog convention](https://keepachangelog.com/en/1.0.0/)
- [Changelog](https://www.npmjs.com/package/changelog)
- [Commitizen](https://www.npmjs.com/package/commitizen)

### Workflows

- [Gitflow Workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)
- [Comparing Workflows](https://www.atlassian.com/git/tutorials/comparing-workflows)
