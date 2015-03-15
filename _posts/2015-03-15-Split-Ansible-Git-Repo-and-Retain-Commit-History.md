---
title: Split an Ansible Git Repo and Retain the Commit History
layout: post
tags:
 - ansible
 - git
---

Starting with a jumbled git repo of various [Ansible](http://www.ansible.com/) roles, playbooks, inventories, group_vars, etc. I want to create a new repo out of a selection of the subdirectories and retain the commit history.

I have an `ansible-test` repo with a tree that looks roughly like this:

```
.
├── adhoc/
│   ├── rolling-reboot.yml
│   └── scripts/
├── README.md
└── runtime/
    ├── roles/
    │   ├── foo-role/
    │   └── zimbra/
    │       ├── ansible.cfg
    │       ├── hosts
    │       ├── tasks/
    │       └── ...
    ├── group_vars/
    │   ├── foo-group
    │   └── zimbra-prod
    ├── hosts
    ├── host_vars/
    ├── library/
    │   ├── foo-lib
    │   ├── zmlocalconfig
    │   └── zmprov
    ├── foo.yml
    └── zimbra-playbook.yml
```

I want to split that so that the Zimbra role, it's playbook, and any _'runtime'_ context like group_vars, and libraries are managed together in a new repo called `playbook-zimbra`.

I poked around Google, and took more than inspiration from these pages:

- [Extracting Parts of Git Repository and Keeping the History](http://ariya.ofilabs.com/2014/07/extracting-parts-of-git-repository-and-keeping-the-history.html)
- Stackoverflow [Howto extract a git subdirectory and make a submodule out of it?](http://stackoverflow.com/questions/920165/howto-extract-a-git-subdirectory-and-make-a-submodule-out-of-it)
- [git-subtree.txt](https://github.com/git/git/blob/master/contrib/subtree/git-subtree.txt)
- [How to tear apart a repository: the Git way](http://blogs.atlassian.com/2014/04/tear-apart-repository-git-way/)

I then tried 2 options:

- [Option 1: Use git subtree - Split and Pull](#option1)
- [Option 2: Use git filter-branch - Clone and Prune](#option2)

<a name="option1"></a>
# Option 1: Use git subtree - Split and Pull #

I tried doing a `git-subtree` split and that worked great for the first directory I split: `runtime/roles/zimbra`. One drawback is the files in `zimbra/` wind up in the root of the new repo instead of `roles/zimbra` as I want. 

I was only able to keep the commit history of the first branch I split, but when I split out the other directories (like `library/` and `group_vars/`) and pulled them into the new git repo the commit history was squashed. Maybe there's a way to make it work, but skip to [option 2](#option2) to see what might be a better solution.

- With the orig repo in `~/src/ansible-test` I created a new clone in `~/src/testing/ansible-test`

```
mkdir ~/src/testing
cd ~/src/testing
git clone ~/src/ansible-test 
cd ~/src/testing/ansible-test
```

- Make branches holding the commits we care about

```
git subtree split --prefix=runtime/roles/zimbra  --branch=zimbra
git subtree split --prefix=runtime/library       --branch=library
git subtree split --prefix=runtime/group_vars    --branch=groupvars
```

- Create a new repo to pull these branches into

```
cd ~/src/testing
mkdir playbook-zimbra
cd playbook-zimbra
git init
```

- Pull in the branches and rebase the files into the subdir that was in the original repo. All these git moves  feel very wrong.

```
git pull ~/src/testing/ansible-test zimbra
mkdir -p roles/zimbra
git add roles
for f in \
	README.md \
	defaults \
	files \
	handlers \
	meta \
	tasks \
	templates \
	vars \
	; do
	git mv $f roles/zimbra
done
git commit -m 'restore dir lost in split'
```

- Continue merging in the other branches created by `git subtree`.

```
git co -b library

git pull ~/src/testing/ansible-test library
mkdir library
git add library
for f in \
	zmprov \
	zmlocalconfig \
	; do
	git mv $f library;
done
git commit -m 'restore dir lost in split'

git pull ~/src/testing/ansible-test groupvars
mkdir group_vars
git add group_vars
for f in \
	zimbra-prod \
	zimbra-test \
	; do
	git mv $f group_vars;
done

#git rm foo-group
git rm trac*
git commit -m 'restore dir lost in split'
```

## Clean up the old original repo ##

Now back in the original `ansible-test` repo, let's get rid of the things we just split out to our new repo.

- Remove the temp branches

```
cd ~/src/testing/ansible-test
git branch -D zimbra
git branch -D library
git branch -D groupvars
```

- Remove the extracted directories. They are still in the commit history, which is fine, but we don't want any more commits on them.

```
cd ~/src/testing/ansible-test
git rm -rf library
git rm -rf roles/zimbra
git rm groupvars/zimbra-prod
git rm groupvars/zimbra-test
```

<a name="option2"></a>
# Option 2: Use git filter-branch - Clone and Prune #

Instead of cherry pick the directorires I want, another option is to clone the repo and prune away what I don't want.

- With the orig repo in `~/src/ansible-test` I created a new clone in `~/src/testing/ansible-test`

```
mkdir ~/src/testing
cd ~/src/testing
git clone ~/src/ansible-test 
cd ~/src/testing/ansible-test
```

- The least common denominator directory in my old `ansible-test` repo is the `runtime/` directory. So, I'll start by doing a split there to a new branch called `runtime`.

```
cd ~/src/testing/ansible-test
git subtree split --prefix=runtime  --branch=runtime
```

- Create a new repo and pull in the runtime branch

```
cd ~/src/testing
mkdir playbook-zimbra
cd playbook-zimbra
git init
git pull ~/src/testing/ansible-test runtime
```

- Now filter away the commits we don't want.

```
git filter-branch --tree-filter 'rm -rf roles/foo-role group_vars/foo-group library/foo-lib foo.yml' HEAD
rm -rf .git/refs/original
```

- And finalize the location of the playbooks and ansible config.

```
git mv roles/zimbra/ansible.cfg .
git mv roles/zimbra/hosts* .
git mv roles/zimbra/*yml .
git commit -m 'move playbook config out of role dir'
```

- Commit and push to origin out on github, gitlab, etc.

```
git remote add origin gitlab@gitlab:ansible/playbook-zimbra.git
git push -u origin master
git checkout -b develop
git push -u origin develop
```

## Clean up the old original repo ##

Now back in the original `ansible-test` repo, get rid of the things now in the `playbook-zimbra` repo.

- Remove the extracted directories. They are still in the commit history, which is fine, but we don't want any more commits on them.

```
cd ~/src/testing/ansible-test/runtime
git rm -rf library
git rm -rf roles/zimbra
git rm groupvars/zimbra-prod
git rm groupvars/zimbra-test
commit -m 'move to playbook-zimbra repo'
```

