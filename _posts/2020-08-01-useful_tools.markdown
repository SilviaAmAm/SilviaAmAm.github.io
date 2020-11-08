---
layout: post
title:  "After 6 months of software development, some nice tricks I learnt about"
date:   2020-08-01 15:13:11 +0100
categories: Software
mathjax: false
---

As a computational chemist starting out in software development, these are some tricks that I wish I had known while I was doing my PhD.

## Git hooks 

[Git hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) let you to run custom scripts before/after important actions. For example, before you  commit some changes. 

I am finding it useful for those python projects where the code needs to be formatted with [black](https://pypi.org/project/black/), [isort](https://pypi.org/project/isort/) and [flake8](https://pypi.org/project/flake8/) before it is allowed to be merged in the main branch. With git hooks, it is possible to run automatically all these tools on any file that will be committed. This prevents me from committing files where I forgot to format the code appropriately!  

#### Example use
Within a git repository, there is a `.git/hooks/` directory.
In this directory there are already some template scripts. I copied the `pre-commit.sample`template and renamed it to `pre-commit` (the name is important!). Then, I modified it to:

{% highlight bash %}
#!/bin/sh
# Run Black and iSort on all python files that are being that are being committed
python_files=$(git status --short | grep -E '^(A|M)' | awk '{ print $2 }' | grep -E '\.py$')

python .git/hooks/pre-commit.py $python_files

if [ $? -eq 1 ]; then
        exit 1
fi

echo -e "No reformatted files! 🎉"
{% endhighlight %}

This shell script extracts all the names of the python files that have been staged. It then calls a `pre-commit.py` script an all these files. The `pre-commit.py` script is as follows:

{% highlight python %}
import subprocess
import sysi

files_to_format = sys.argv[1:]
formatting_changes = False

for f_to_format in files_to_format:
    black_output = subprocess.run(["path/to/black", f_to_format], stdout=subprocess.PIPE, stderr=subprocess.PIPE)
    isort_output = subprocess.run(["path/to/isort", f_to_format], stdout=subprocess.PIPE, stderr=subprocess.PIPE)

    if "reformatted" in black_output.stderr.decode('utf-8') or "Fixing" in isort_output.stdout.decode('utf-8'):
        print(f"Reformatted {f_to_format}")
        formatting_changes = True

if formatting_changes:
    print("Formatted some files. Git add them and then recommit")
    sys.exit(1)
{% endhighlight %}

This runs `black` and `isort` on all the files passed by the shell script. If files are reformatted, the commit is aborted. The files need to be staged again and then recommitted. Otherwise, they are committed as usual.


## Command "pip install -e"

The `-e` option installs a python package in editable mode, either from a local path or from a version control system URL.

This is useful whenever you have a project that imports a package that you are actively developing (i.e. constantly modifying). No more making simlinks manually to the edited files or constantly reinstalling the modified package!!
Just `pip install -e <package-path>` once, and code away!


## Command "git pull --rebase"

When you pull from a remote repository, by default the remote branch will be merged in your local branch. With the `--rebase` option, you can instead rebase your local branch onto the remote branch that was fetched (for a nice explanation of the differences between merging and rebasing, go [here](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)). 

This will avoid all those merge commits with variations on the message "Merged from remote!".

In addition, if you do `git pull --rebase` often, you can add it to the git configurations so that every time you `git pull`, it automatically rebases instead of merging. To add it to the global `.gitconfig` file, you can use the following command:

{% highlight bash %}
git config --global pull.rebase true
{% endhighlight %}

Otherwise, to add it only in specific repositories add the following lines to the `.git/config` file:

{% highlight bash %}
[pull]
        rebase = true
{% endhighlight %}

## Gitmoji

[Gitmoji](https://gitmoji.carloscuesta.me/) is a guide for commit messages. It helps you keep your commits small and focused. For each 'type of commit' you include an emoji in the commit message which summarises what has been changed.
Whenever you feel like multiple emojis apply to the changes in your commit, that's a sign that you should maybe split the commit into 2 parts.

Since I started using it, I haven't found myself in the situation where I would like to revert a commit but keep *some* of the changes.
Another added bonus is the pretty commit history full of emojis! 🎉

## Command "git add -p"

If you have made many changes to a file, but you don't want to commit all of them at the same time, now you don't have to. `git add -p <filename>` gives you the opportunity to commit each change separately (and maybe give each one their own gitmoji! 😉).


