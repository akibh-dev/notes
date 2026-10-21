---
layout: post
title: "You Need Another CLI for VCS"
description: "Why I keep GitHub's gh CLI next to git: logging in and switching accounts, opening pull requests, and running Actions and secrets from the terminal."
date: 2026-09-24
type: guide
tags: ["github", "cli", "secrets"]
featured: false
ai: none
---

## Git or Github

Disclaimer: Before we start, I am very much disgusted with `Github` and anything related to `Micro`~~slop~~ `soft` and `Copilot` and yada yada. Not because of [this](./github-up-time.png). Well if you have clicked the link there should be another *sane* hater, or you are a pure soul. Although I (*needed to*) use github; you know we do things we do not like mostly.

The debate is little bit forced in the community git or github: both serves different aspects. Using git requires a lot of cli work mostly we have a grab on those command. I mean we do understand whats happening (*until rebase*). And we almost always use github UI for rest of the time. Now and then useless developers spends their time and provides cli for everything. And github devs are no exception but giving us cli tool `gh` instead of fixing their bugs like [this](https://github.com/actions/runner/blob/v2.328.0/src/Misc/layoutroot/safe_sleep.sh). Okay enough with this lets go to the cli.

## GH for Github

`gh` is github in terminal. It supports almost all necessary github tools or actions via cli as advertised. We do not question that. It comes with it's own credential manager(*git sometimes annoyingly relies on system credential manager until you have time to setup ssh for each repo*) and automatically pushes to private repo. So cool. 

### Authentication

Now to start first install gh in your system what ever you like package manager, webi, or curl sh if they provides now. Just make sure the your repo's and `gh` has same level of access in the system(I have not tried otherwise).

Then comes the login. Simple and intuitive:

```bash

$ gh auth login

```
There are custom flags like `--web`, `--hostname`. My philosophy is not to make it overcomplex. However we can add multiple accounts login with similar command. Now, you guessed it right, we need to switch to that account.

```bash

$ gh auth switch

```

When a private repo is cloned with https before even installing, it might not work as expected. So we need to check the current pointing account, refresh, or even tell gh that *we are the the guy*.

```bash

$ gh auth refresh

$ gh auth switch

$ gh auth setup-git

$ gh auth status

```

## PRs

Now we have setup `gh`, we are not using it anymore. Instead we use `git` for all stashes, commits, reset, squashes etc. Now and then our job is done some how in our pc and the codes needed to be pushed. Well we use `git push origin <branch>` but `gh` helps sometimes to push in private repo without verifying. And now we should go to browser -> create new pull request and wait for code to be build failed. We can use `gh pr` for all of those.

```bash
# This is my list of flags, i need minimally
$ gh pr create --base <branch> --title <your title>

# Or use interactive approach
$ gh pr create

# Then check status if failed
$ gh pr status

# Then probably merge it, sometimes use -d flag for deleting the branch
$ gh pr merge <branch>

```

## Actions and Secrets

Thats it. Really, thats all i needed for my simple workflow. If I need any complex job, I would probably use UI. However sometimes to do a quick harmless tasks like run a action(*If it does not have any complex parameters*) we can use `gh workflow`.

```bash
# List all workflows
$ gh workflow list

# Run a workflow : gh workflow run deploy.yml -f env=development
$ gh workflow run <id or name> -f <key>=<value>

# If you forgot the name or parameters just use interactive mode
$ gh workflow run 

# To see the filure
# I know, you do not remember anything, just for interactive mode
$ gh workflow view <id or name or filename>

```

And now the expected part where it breaks. I have no idea how the authentication system works for `gh` and github. It's very crutial for these part. Because you could have elivated access to the repo from `gh` cli where you can be restricted in github website. Such as using `gh secrets` you can access and edit secrets of private repo where you do not have admin/manage access to the repo. Now if the deployment file does not sanitize the inputs, well well well, you have got a RCE XD.

```bash
# List all repo secrets
$ gh secrets list

# Secrets based on environment
$ gh secrets list --env <env-name>

# Set a secert
$ gh secrets set <secret-name>

# Or provide direct .env to a env
$ gh secrets set --env <env-name> <.env file>

```

Finally, doing github actions by cli helps to use less the website, increases peace. If you need more commands or flags details; all the commands list can be found  in [here](https://cli.github.com/manual/gh). 

Happy Coding!!
