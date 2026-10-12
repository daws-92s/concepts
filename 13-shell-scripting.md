# Shell Scripting and Version Control Basics

## What this is / why it matters
Doing deployments and server changes by hand (typing commands one at a time, every time) doesn't scale and doesn't stay safe for long. Shell scripting turns a sequence of manual steps into a single, repeatable program. Version control (Git) is what makes it safe to change those scripts over time — tracking who changed what, keeping old versions recoverable, and letting more than one person work on the same code.

## How it works

### The problem with doing things manually
Running commands by hand, one at a time, has the same handful of problems every time:
- **Only you know the steps** — there's no written record of what was actually done, only whatever is in someone's head or a one-off note.
- **It's slow** — every repeat of the same deployment takes the same amount of manual effort again.
- **Human mistakes** — a skipped step or a typo is easy to make and easy to miss.
- **No easy way back** — if something goes wrong, there's no recorded "previous state" to restore to.
- **No accountability** — when several people can log in and make changes, it's hard to tell who changed what.

A shell script fixes the first four: it's a written, repeatable, exact record of the steps, run the same way every time.

### Why scripts still aren't enough on their own
A script sitting on a server (or a laptop) by itself still has gaps:
- **No security** — anyone with access to the file can change it, with no record of the change.
- **No backup** — if the file is deleted or corrupted, that version of the logic is gone.
- **No versioning** — there's no way to tell "this is version 1.0.0" from "this is version 2.0.0," or to go back to an older one.
- **No collaboration** — two people editing the same file by hand will overwrite each other.
- **No review** — nobody gets a chance to check a change before it takes effect.
- **No tracking** — no history of what changed, when, or why.

This is exactly what a **version control system (VCS)**, also called **source code management (SCM)**, is for.

### VCS/SCM, and where Git fits
Different tools store different kinds of things:
- **Git** — stores and versions source code (and scripts).
- **Google Drive** (and similar) — general file/object storage, not built for tracking code changes.
- **An artifact repository** (e.g. Nexus, Artifactory) — stores build outputs (artifacts), not source code.

A **repository** is just storage for something — in Git's case, storage for a project's code and its full history of changes.

Basic Git workflow:
```bash
# one-time setup, per machine (or per repo with the file flag removed)
git config --global user.email "you@example.com"
git config --global user.name "Your Name"

git init                       # turn the current folder into a git repo
git branch -M main             # name the default branch "main"
git add .                      # stage all changes
git commit -m "my first commit"
git remote add origin <URL>    # point this repo at one on GitHub
git push origin main           # upload commits to the remote repo
```

A normal folder becomes a **git repo** the moment it has a hidden `.git` subfolder in it — that's what Git uses to track history. No `.git` folder means it's just a regular folder, with no version tracking at all.

Typical day-to-day flow for one file:
```bash
git add 01-hello-world.sh              # stage the file
git commit -m "my first shell script"  # commit to the local repo
git push origin main                   # push to the remote repo (e.g. GitHub)
```

The usual review habit before touching an important shared file: make a copy, make the change in the copy, show it to a lead/reviewer, and only then replace the original once it's approved. Git's review tools (pull requests) are the structured version of this same idea.

### Shebang
Every shell script should start with a **shebang** line:
```bash
#!/bin/bash
```
It has to be the very first line in the file. The shebang tells Linux which interpreter should run the rest of the file — without it, Linux has no way to know the file is a set of bash commands rather than, say, a Python script. "Shell script" and "bash script" mean the same thing here, since bash is the shell being used.

### Variables
A variable holds a value under a name, so the value only has to be written once:
```bash
NAME="John"
```
This matters for the same reason **DRY (Don't Repeat Yourself)** matters in any code: if a value (a path, a username, a number used in a calculation) is typed out directly in ten places and it needs to change, all ten have to be found and fixed by hand. Held in one variable, it only has to change in one place, and that one change takes effect everywhere the variable is used — which also prevents accidental, inconsistent edits.

**Arguments** are values passed into a script when it's run:
```bash
sh 04-variables.sh John Raheem
```
Inside the script, these show up as numbered variables — `$1` is the first argument, `$2` is the second, and so on:
```text
$1 = John
$2 = Raheem
```

**Command substitution** runs a command and stores its output in a variable, using `$( )`:
```bash
START_TIME=$(date +%s)
```
This is how a script can measure its own runtime — capture the time at the start, capture it again at the end, and subtract:
```text
Script executed at: 08:34 AM
Script completed at: 08:35 AM
Total time for execution is 1 min
```

## Common problems and how to solve them
- **`fatal: unable to auto-detect email address` / "Please tell me who you are"** — Git refuses to commit until it knows who's committing. Fix: run `git config --global user.email "..."` and `git config --global user.name "..."` once per machine (drop `--global` to set it for just one repository).
- **Git push asks for authentication** — on a fresh setup, pushing to GitHub for the first time prompts for a login, usually handled by signing in through the browser when prompted.
- **A script runs "command not found" or behaves oddly depending on how it's run** — usually a missing or wrong shebang line. Make sure `#!/bin/bash` is the exact first line, with nothing above it.

## Key takeaways
- Manual deployment has five recurring problems: no documentation of the steps, slow repetition, human error, no way to restore a previous state, and no accountability for who changed what.
- A shell script solves the manual-process problems by making the steps repeatable and written down — but a script by itself still has no security, backup, versioning, collaboration, review, or change tracking.
- Git (a VCS/SCM) solves that second set of problems; it's a different kind of tool from general file storage (Google Drive) or an artifact repository, which store different things entirely.
- A folder is a git repo only once it contains a `.git` subfolder — that's what's actually tracking the history.
- `#!/bin/bash` must be the first line of a shell script — it tells Linux which interpreter runs the file.
- Variables (including positional arguments `$1`, `$2`, ... and command substitution `$( )`) exist so a value is written once and reused — the same DRY principle behind avoiding repeated code anywhere else.

See also: [06-linux-admin.md](06-linux-admin.md)
