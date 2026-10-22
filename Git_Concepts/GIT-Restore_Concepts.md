## <span class="underline">GIT Restore Concepts:</span>

In Git, git restore is used to bring a file back to a previous version
or copy a file version from another branch.

It is one of the most important commands to understand when learning
Git, especially because you're practising with DEVOPS-Journey-Fresh.

Let's learn it from the beginning, with practical Linux examples.

**1. First, understand where Git stores your changes**

Before learning git restore, you need to understand these three areas.

![](git-github-image/media/image1.png)

git restore primarily operates on your working directory and staging
area. It doesn't automatically change GitHub.

**2. Let's create a practical example**

Suppose you have a file called linux.txt in your fresh repository.

cd \~/DEVOPS-Journey-Fresh

<span class="underline">Create a file:</span>

echo "Linux is an operating system" \> linux.txt

<span class="underline">Stage and commit:</span>

git add linux.txt

git commit -m "Add Linux introduction"

*Your first committed version is:*

Linux is an operating system

*Now suppose you accidentally replace its content:*

echo "Wrong information" \> linux.txt

*Your current file contains*:

Wrong information

*But your last commit still contains:*

Linux is an operating system

How can you get the original content back?

This is where ***git restore*** becomes useful.

1.  > **Case 1 — Restore changes that have NOT been staged**

Run:

git restore linux.txt

Git replaces the working-directory version with the version from the
staging area.

If you haven't staged anything new, that will normally be the version
from your last commit.

![](git-github-image/media/image2.png)

Warning⚠️: git restore linux.txt discards unstaged changes to that file.
Make sure you no longer need them.

**4. Case 2 — Restore a file that has already been staged**

Now let's take another example.

Suppose you modify linux.txt:

echo "Linux supports multiple users" \>\> linux.txt

Then stage it:

git add linux.txt

Now you have staged changes.

But you realize you don't want to commit them yet.

Use:

git restore --staged linux.txt

What happens?

  - Git removes the changes from the staging area.

  - Your modified file content remains in your working directory.

  - Your last commit is unchanged.

So git restore --staged is commonly used to unstage a file.

This does not delete the modifications you've made.

**5. Case 3 — Restore both staged and unstaged changes**

Suppose you edited a file, staged it, and then edited it again.

Now you want to discard all changes to that file and return to the last
committed version.

Run:

git restore --source=HEAD --staged --worktree linux.txt

Here:

  - **--source=HEAD** means use the latest commit on your current
    branch.

  - **--staged** means update the staging area.

  - **--worktree** means update the file in your working directory.

This discards both the staged and unstaged modifications to the file.

**6. Interactive comparison — What will git restore affect?**

Choose the command to understand which area Git modifies.

![](git-github-image/media/image3.png)

![](git-github-image/media/image4.png)

![](git-github-image/media/image5.png)

**7. Case 4 — Restore a deleted file**

*Suppose you accidentally deleted a tracked file:*

rm linux.txt

*If the deletion hasn't been staged, you can recover the file from the
staging area:*

git restore linux.txt

The file appears again.

But remember, this works for a file Git is already tracking. If you
create a new untracked file and delete it before adding or committing
it, Git normally cannot restore it.

**8. Case 5 — Restore a file from another branch**

This example directly relates to your old DEVOPS-Journey repository.

You previously wanted to copy:

Linux\_Notes/linux\_assignment\_questions.txt

from Linux\_Notes to main.

You can do this using git restore.

First, in your original repository:

cd \~/DEVOPS-Journey

git fetch origin

git switch main

Then:

gitrestore--source=origin/Linux\_Notes--Linux\_Notes/linux\_assignment\_questions.txt

![](git-github-image/media/image6.png)

After verifying the file, you can stage and commit:

git add Linux\_Notes/linux\_assignment\_questions.txt

git commit -m "Add Linux assignment questions"

git push origin main

Note that the restore command itself doesn't push changes to GitHub.

Also, origin/Linux\_Notes is your locally fetched reference to the
remote branch. Running git fetch origin first updates it.

**9. Case 6 — Restore a file from an older commit**

Suppose your commit history looks like:

A → B → C → D

And commit B contains an older version of linux.txt that you want to
recover.

First, find the commit IDs:

git log --oneline

Example:

e8f42a1 (HEAD -\> main) Update Linux commands

c6d73b2 Add shell scripting

b4a21c9 Add Linux introduction

a19b235 Initial commit

*To restore the file version from commit b4a21c9:*

git restore --source=b4a21c9 -- linux.txt

This replaces the working-directory version of linux.txt with the one
from that older commit.

Your existing commit history remains intact. If you want to make this
restored content the latest version in your branch, stage and commit it.

**10. Difference between Git restore, reset, revert and checkout**

These commands can be confusing because they all relate to previous
versions of your project.

| **Command**  | **Main purpose**                                                      |
| ------------ | --------------------------------------------------------------------- |
| git restore  | Restore file content or unstage changes                               |
| git reset    | Move the current branch pointer, or reset the staging area            |
| git revert   | Create a new commit that reverses changes made by an earlier commit   |
| git checkout | Switch branches or restore file versions (older multipurpose command) |

<span class="underline">Important distinction</span>: If a bad commit is
already pushed to a shared GitHub branch, git revert is often safer than
rewriting commit history using git reset.

For example:

git revert e8f42a1

This creates a new commit reversing the selected commit's changes, when
Git can apply that reversal. It doesn't erase the old commit from
history.

**11. Does GitHub have the same restore command?**

GitHub is where your remote Git repository is hosted. You run git
restore in your local Git terminal, not directly on the GitHub website.

For example, if you want to restore an older file version and upload it
to GitHub:

git restore --source=\<commit-id\> -- linux.txt

git add linux.txt

git commit -m "Restore previous Linux file version"

git push origin main

You can also use GitHub's website to inspect commit history, view file
versions, and revert certain pull requests. However, the website doesn't
provide the same general-purpose git restore command interface as your
Linux terminal.

**12. Quick reference — Git restore commands**

| **Requirement**                                | **Command**                                            |
| ---------------------------------------------- | ------------------------------------------------------ |
| Discard unstaged modifications                 | git restore file.txt                                   |
| Remove a file from staging                     | git restore --staged file.txt                          |
| Discard staged and unstaged modifications      | git restore --source=HEAD --staged --worktree file.txt |
| Recover an unstaged deletion of a tracked file | git restore file.txt                                   |
| Copy file from another branch                  | git restore --source=branch\_name -- file.txt          |
| Restore file version from older commit         | git restore --source=commit\_id -- file.txt            |
| View current file modifications                | git diff -- file.txt                                   |
| View staged modifications                      | git diff --staged -- file.txt                          |

**One rule I recommend while you're learning Git**

Before running any restore command, always check:

git status

git diff

git diff --staged

These three commands help you understand which changes are in your
working directory and which changes are staged.

Final concept to remember:

  - **git restore** = Bring back file content or unstage changes.

  - **git commit** = Save a version in local Git history.

  - **git push** = Upload your local commits to GitHub.

  - **git revert** = Reverse an earlier committed change with a new
    commit.

The most important distinction is between restoring a file and reversing
a commit. They are not the same operation.
