# GitHub Action: Push git subdirectory as branch

[![](https://github.com/s0/git-publish-subdir-action/workflows/Scheduled%20tests/badge.svg)](https://github.com/s0/git-publish-subdir-action/actions?workflow=Scheduled+tests) [![codecov](https://codecov.io/gh/s0/git-publish-subdir-action/branch/master/graph/badge.svg)](https://codecov.io/gh/s0/git-publish-subdir-action)

This GitHub Action will take any subdirectory in your repository, and push it as the contents of a git branch to a repository and branch of your choosing, either over SSH or to the current repo.

You could use this for example to:

* Publishing a subdirectory to a repo's `gh-pages` branch, after optionally running a build step.
* Publishing build artifacts / binaries to another repository

The target repository can be anywhere accessible by a [Git SSH URL](https://git-scm.com/book/en/v2/Git-on-the-Server-The-Protocols#_the_ssh_protocol) (or the current repository).
If the target branch doesn't exist yet, it will be created automatically.

## Usage

Simply include the action `s0/git-publish-subdir-action@develop` in the appropriate point in your workflow, and pass in the required configuration options:

```yml
jobs:
  deploy:
    name: Deploy
    runs-on: ubuntu-latest    
    steps:

    # Any prerequisite steps

    # Deploy to local repo
    - name: Deploy
      uses: s0/git-publish-subdir-action@develop
      env:
        REPO: self
        BRANCH: gh-pages
        FOLDER: build
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

    # Deploy to another repo
    - name: Deploy
      uses: s0/git-publish-subdir-action@develop
      env:
        REPO: git@github.com:owner/repo.git
        BRANCH: gh-pages
        FOLDER: build
        SSH_PRIVATE_KEY: ${{ secrets.DEPLOY_PRIVATE_KEY }}
        KNOWN_HOSTS_FILE: resources/known_hosts # Needed if target repo is not on github.com
```

## Examples

### When pushed to master, push `/public/site` to the `gh-pages` branch on the same repo

```yml
name: Deploy to GitHub Pages
on:
  push:	
    branches:	
      - master

jobs:
  deploy:
    name: Deploy to GitHub Pages
    runs-on: ubuntu-latest    
    steps:
    - uses: actions/checkout@master

    - name: Deploy
      uses: s0/git-publish-subdir-action@develop
      env:
        REPO: self
        BRANCH: gh-pages
        FOLDER: public/site
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### When pushed to master, run a build step, then push `/build` to the `gh-pages` branch on another repo on GitHub

```yml
name: Deploy to GitHub Pages
on:
  push:	
    branches:	
      - master

jobs:
  deploy:
    name: Deploy to GitHub Pages
    runs-on: ubuntu-latest    
    steps:
    - uses: actions/checkout@master
    - name: Use Node.js
      uses: actions/setup-node@v1
      with:
        node-version: 10.x
    - name: npm install and build
      run: |
        npm install
        npm run build

    - name: Deploy
      uses: s0/git-publish-subdir-action@develop
      env:
        REPO: git@github.com:owner/repo.git
        BRANCH: gh-pages
        FOLDER: build
        SSH_PRIVATE_KEY: ${{ secrets.DEPLOY_PRIVATE_KEY }}
```

Note: the SSH Key needs to have write access to the given repo. It's recommended you use [Deploy Keys](https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys) for this (see below),
and store the SSH private key as as Secret in the repository settings.


### When pushed to master, run a build step, then push `/dist` to the `artifacts` branch on a repo hosted at mydomain.com

```yml
name: Deploy to GitHub Pages
on:
  push:	
    branches:	
      - master

jobs:
  deploy:
    name: Deploy to GitHub Pages
    runs-on: ubuntu-latest    
    steps:
    - uses: actions/checkout@master
    - name: Use Node.js
      uses: actions/setup-node@v1
      with:
        node-version: 10.x
    - name: npm install and build
      run: |
        npm install
        npm run build

    - name: Deploy
      uses: s0/git-publish-subdir-action@develop
      env:
        REPO: git@mydomain.com:path/to/repo.git
        BRANCH: artifacts
        FOLDER: dist
        SSH_PRIVATE_KEY: ${{ secrets.DEPLOY_PRIVATE_KEY }}
        KNOWN_HOSTS_FILE: resources/known_hosts # Path relative to the root of the repository
```

You can generate a `known_hosts`  file for a given domain by using `ssh-keyscan`, e.g:

```bash
> ssh-keyscan github.com
# github.com:22 SSH-2.0-babeld-f345ed5d
github.com ssh-rsa AAAAB3NzaC1yc2EAAAABIwAAAQEAq2A7hRGmdnm9tUDbO9IDSwBK6TbQa+PXYPCPy6rbTrTtw7PHkccKrpp0yVhp5HdEIcKr6pLlVDBfOLX9QUsyCOV0wzfjIJNlGEYsdlLJizHhbn2mUjvSAHQqZETYP81eFzLQNnPHt4EVVUh7VfDESU84KezmD5QlWpXLmvU31/yMf+Se8xhHTvKSCZIFImWwoG6mbUoWf9nzpIoaSjB+weqqUUmpaaasXVal72J+UX2B+2RPW3RcT0eOzQgqlJL3RKrTJvdsjE3JEAvGq3lGHSZXy28G3skua2SmVi/w4yCE6gbODqnTWlg7+wC604ydGXA8VJiS5ap43JXiUFFAaQ==
# github.com:22 SSH-2.0-babeld-f345ed5d
# github.com:22 SSH-2.0-babeld-f345ed5d
```


## Configuration

All configuration options are passed in via `env`, as environment variables.

| Env Variable       | Description                                            | Required?     |
| ------------------ | ------------------------------------------------------ | ------------- |
| `REPO`             | Either `self`, or an SSH url to the target repository. | Yes           |
| `BRANCH`           | The target branch to publish to.                       | Yes           |
| `FOLDER`           | The target subfolder you would like to publish         | Yes           |
| `SSH_PRIVATE_KEY`  | The private key that should be used to authenticate on SSH. Don't include this directly in the workflow file, instead you must use [Secrets](https://help.github.com/en/articles/virtual-environments-for-github-actions#creating-and-using-secrets-encrypted-variables) | When `REPO` is an SSH URL |
| `KNOWN_HOSTS_FILE` | Path to a file in the repository that contains the known SSH fingerprints for the target host. | When the target host is not github.com |
| `GITHUB_TOKEN`     | Should always be equal to `${{ secrets.GITHUB_TOKEN }}` | When `REPO = self` |
| `SQUASH_HISTORY`   | If set to `true`, all previous commits on the target branch will be discarded. For example, if you are deploying a static site with lots of binary artifacts, this can help the repository becoming overly bloated. | No |
| `MESSAGE`          | A custom template to use as the commit message pushed to the target branch. See [custom commit messages](#custom-commit-messages). | No |
| `TAG` | A string following the [git-check-ref-format](https://git-scm.com/docs/git-check-ref-format) that tags the commit with a lightweight git-tag. | No |

## Custom commit messages

You can specify a custom string to use in the commit message
when pushing to your target repository.
These strings support a number of placeholders that will be replaces with
relevant values:

| Placeholder        | Description                                           |
| ------------------ | ----------------------------------------------------- |
| `{target-branch}`  | The name of the target branch being updated           | 
| `{sha}`            | The 7-character sha of the HEAD of the current branch | 
| `{long-sha}`       | The full sha of the HEAD of the current branch        | 
| `{msg}`            | The commit message for the HEAD of the current branch | 

Example Usage:

```yml
jobs:
  deploy:
    - uses: s0/git-publish-subdir-action@develop
      env:
        # ...
        MESSAGE: "This updates the content to the commit {sha} that had the message:\n{msg}"
```

## Usage with [Deploy Keys](https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys)

When pushing to other repositories on GitHub or GitHub Enterprise,
the recommended mechanism is to use [Deploy Keys](https://developer.github.com/v3/guides/managing-deploy-keys/#deploy-keys) rather than your own private SSH key.
Deploy keys are SSH keys that can be added to specific repositories to be given write access to only
those repositories.
This is preferable to adding your own personal ssh private key to a repository's secrets store,
as it means that any actions that have access to this repository's secrets can only push
to repositories that have explicitly had the SSH key added as a deploy key,
and not *all* repositories that your user account have access to.

Use `ssh-keygen` to create a new ssh key, add these to GitHub in the relevant repo's deploy keys and Secrets, then delete them off your computer.

```
> cd /tmp
> ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/sam/.ssh/id_ed25519): temp-deploy-key
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in temp-deploy-key.
Your public key has been saved in temp-deploy-key.pub.
The key fingerprint is:
SHA256:tQBSeWjZ4Er4YYjK4XQ4npfiK2xJPJLbGjTsYJq/9JI sam@optimus
The key's randomart image is:
+--[ED25519 256]--+
|    ..+*         |
| ..o o=.o        |
|.=o.+.... .      |
|B =+.o   o .     |
|+% oo   S .      |
|X=+              |
|B=+.             |
|oBE.             |
|+ooo.            |
+----[SHA256]-----+
> cat temp-deploy-key.pub
ssh-ed25519 XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX user@localhost
> cat temp-deploy-key
-----BEGIN OPENSSH PRIVATE KEY-----
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX=
-----END OPENSSH PRIVATE KEY-----
```