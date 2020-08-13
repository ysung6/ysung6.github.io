---
title:  "Fixing your last commit"
last_modified_at: 
categories: 
  - Git
tags:
  - git
toc: true
toc_label: "Getting Started"
---

If you've used git, you're probably familiar with `git commit`. It's basically saving your work, and marking a point where you can go back to. But sometimes you may have made a mistake in your last commit and want to fix the commit itself instead of making an additional commit.

## Amend your commit

The command for fixing your last commit is `git commit --amend` . 
So your workflow would looklike
```
git commit  # incomplete commit
# complete the task
git add .
git commit --amend  # Add the staged changes to your previous commit. 
```
It's simple just add the `--amend` the next time you commit.

## Fixing the commit message

Or you might just want to fix your commit message. There might have been a typo or you might have found a more appropriate message to descirbe your work.

In this case, you can just use 
`git commit -m --amend`

## Why bother fixing commits?
Making an incomplete commit or an inadequate commit message may not seem like a big deal.

But keeping your commits clear and concise will help you enjoy some convenient feature that github provides.

1. You can make use of github's search. You can search commits and pull requests on github. If you keep your commit messages relevant, it becomes significantly easier to look up your commit.

2. Your peers will be able to give you better feedback. When you make a pull request, some reviewers go through your commits to follow your thought process. By having commits separated in clear distinct tasks along with relevant messages, reviewers will be able to understand the changes you've made.

### Closing remarks

Personally I use this quite often when I have to make code style changes after a commit. Because I don't think I will ever look up a commit with minor style fix that should have been done in the previous commit, I use `git --amend` for such cases. This will help me reduce unnecesary commits.