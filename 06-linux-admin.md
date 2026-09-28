# Linux Administration and Operations

## What this is / why it matters
Anyone can run commands on a Linux box, but administering it means controlling *who* can do *what*. That comes down to three connected pieces: proving who someone is (authentication), deciding what they're allowed to touch (authorization), and the day-to-day commands that enforce both — creating users, assigning groups, and setting file permissions.

Once users can log in, the next question is always the same: is the thing you actually care about — a web app, a database, a build agent — installed, running, and reachable? That's four more connected areas covered here: getting software onto the box (package management), keeping it running (service management), confirming it's reachable (network management), and confirming it's actually alive at the OS level (process management).

## How it works

**Users and the prompt:**
- `$` at the end of a prompt — a normal (non-root) user
- `#` at the end of a prompt — the root user
- `/root` — the root user's home folder
- `/home/<username>` — every other user's home folder (e.g. `/home/ec2-user`)

**Authentication vs. authorization:**
- **Authentication** — proving you are who you say you are (logging in with a password or SSH key)
- **Authorization** — once you're in, what you're actually allowed to do

Authorization is usually modeled as roles mapped to permissions, then roles assigned to groups:

| Role | Permissions |
|---|---|
| Trainee | Read only |
| Junior | Read, Write |
| Senior | Read, Write, Update |
| Team Lead | Read, Write, Update, Delete |

A user (e.g. `ramesh`) gets access by being added to the group that matches their role (e.g. `devops-trainees`, `devops-juniors`), rather than by setting permissions on that one user individually — it's easier to manage access for a whole group than to repeat the same setup per person.

**Users and groups:**
- A **user** is one person's account.
- A **group** is a named list of users, used to grant the same access to all of them at once.
- Every user has exactly one **primary group** and can belong to zero or more **secondary (supplementary) groups**.
- `useradd <username>` creates a new user. By default, it also creates a new group with the same name and makes that the user's primary group.
- UID `0` is always the root user, regardless of username.
- `/etc/passwd` — stores user account info (username, UID, home directory, shell, etc.)
- `/etc/group` — stores group info and membership

Assigning groups:
```
usermod -g devops ramesh      # sets devops as ramesh's primary group
usermod -aG sre ramesh        # adds ramesh to sre as a secondary group (-a = append)
```
The `-a` (append) matters: `usermod -G sre ramesh` without `-a` replaces *all* of a user's secondary groups with just `sre`, wiping out any others they were already in.

Setting a password:
```
passwd ramesh
```

**SSH access config:**
- `/etc/ssh/sshd_config` — the SSH server's configuration file (e.g. `PasswordAuthentication yes/no`)
- `sshd -t` — checks the config file for syntax errors before you restart the service
- `systemctl restart sshd` — applies config changes

## File permissions

Three permission types, each with a numeric value:
- **R**ead = 4
- **W**rite = 2
- e**X**ecute = 1

A permission string like `-rw-r--r--` breaks into three groups of three:
```
u        g        o
Owner    Group    Others
rw-      r--      r--
```
Only the file's owner or the root user can change a file's permissions.

Changing permissions:
```
chmod u+x devops.txt          # add execute permission for the owner
chmod 751 devops.txt          # owner=rwx(7), group=r-x(5), others=x(1)
```

**Ownership** is a separate thing from permissions — even the file's owner can't change who owns it; only root can:
```
chown user:group devops.txt
chown user:group -R some-folder   # -R applies it recursively to everything inside
```

## Granting admin (sudo) access

Adding a user to the **wheel** group gives them sudo access (root-equivalent, via `sudo`) on RHEL-family systems (Debian-family systems use a `sudo` group instead):
```
usermod -aG wheel ramesh
```

The same access can be granted more precisely through the sudoers config, without touching group membership:
- `/etc/sudoers` — the main sudo config file
- `/etc/sudoers.d/` — a directory for drop-in sudo config files, one per user/purpose, so you don't have to hand-edit the main file
- Always edit sudo config with `visudo` (not a regular editor) — it validates syntax before saving, so a mistake can't lock you out of sudo entirely
- `visudo -c` — checks the sudoers file (and everything in `sudoers.d`) for syntax errors without opening an editor

Examples, in `/etc/sudoers.d/ramesh`:
```
ramesh  ALL=(ALL:ALL) ALL                                          # full sudo access — same effect as wheel membership
ramesh  ALL=(ALL:ALL) NOPASSWD:ALL                                 # full sudo access, no password prompt
ramesh  ALL=(ALL:ALL) NOPASSWD: /usr/sbin/useradd, /usr/sbin/usermod   # sudo, but only for these two specific commands
```

Removing access:
```
gpasswd -d ramesh wheel     # removes ramesh from the wheel group
```

## Setting up key-based access for a new user
1. The user generates their own SSH key pair (`ssh-keygen`) and keeps the private key to themselves.
2. They send the admin only the **public** key — never the private one.
3. The admin creates the user (`useradd ramesh`), then creates a `.ssh` folder inside that user's home directory, and inside it an `authorized_keys` file containing the public key.
4. Permissions matter here — SSH will refuse to use a key that's too open:
   ```
   chmod 700 /home/ramesh/.ssh              # only ramesh can access the folder at all
   chmod 600 /home/ramesh/.ssh/authorized_keys   # only ramesh can read/write the file
   ```
5. The user connects with their private key:
   ```
   ssh -i ramesh-private-key ramesh@<server-ip>
   ```

## Offboarding a user
When someone leaves, access needs to come off cleanly and in the right order — cutting them off before you've captured their data means losing it, and capturing data before cutting them off leaves a window where they still have access.

1. **Remove from privileged groups first** — `gpasswd -d ramesh wheel` and reset their primary group so nothing still traces back to sensitive access: `usermod -g ramesh ramesh`
2. **Lock the account** so no new login is possible: `usermod -e 1 ramesh` (sets the account's expiry date to a date in the past)
3. **Kill any sessions already in progress**: `pkill -u ramesh`
4. **Back up their home folder** before deleting anything: `tar -czvf /backup/ramesh-offboarding-backup.tar.gz /home/ramesh/`
5. **Check for files owned by them outside their home folder** — a user can own files anywhere on the system, not just in `/home`: `find / -user ramesh`
6. **Delete the account**: `userdel ramesh`

`find` is the general-purpose tool for step 5 — it searches a directory tree for files matching a condition:
```
find <where-to-search> <options> <what-to-search>
find / -name "*.log"          # every .log file on the system
find / -type d -name "ramesh" # every directory named "ramesh"
```

## Package management
Servers typically have a URL configured to reach package repositories over the internet, so installing software is a single command instead of manually downloading and compiling it.

- `dnf install <package>` — install a package
- `dnf remove <package>` — remove a package
- `dnf update <package>` — update a package
- `dnf search <package>` — search available packages
- `dnf list installed` — list everything currently installed

On RHEL-family systems, `yum` was the original package manager; `dnf` is its modern replacement (faster, better dependency resolution) — on current systems `yum` is just a symlink to `dnf`, kept around so old scripts and habits still work. Debian-family systems (Ubuntu, etc.) use `apt-get` instead — same idea, different tool, because the two families use different package formats.

## Service management
A **service** is a program that runs continuously in the background, waiting to do work — `sshd` (handles SSH connections) is one example; a web server is another. The Linux server itself is the physical/virtual machine; a service like `nginx` is a logical process running inside it — the two are related but not the same thing.

`nginx` and Apache HTTPD are both web servers (they serve HTTP/HTTPS traffic); nginx is the more modern, commonly used default today, with Apache seen more often in older setups.

`systemctl` manages services:
- `systemctl start nginx` — start it now
- `systemctl stop nginx` — stop it
- `systemctl restart nginx` — stop then start (used after a config change)
- `systemctl enable nginx` — start automatically on every future boot
- `systemctl disable nginx` — don't start automatically on boot
- `systemctl status nginx` — check whether it's currently running, and see recent log lines

## Network management
A server has 65,536 possible ports (0–65,535) that a service can listen on. Some are conventional defaults:

| Service | Port |
|---|---|
| SSH | 22 |
| HTTP | 80 |
| HTTPS | 443 |
| DNS | 53 |
| SMTP | 25 |
| MySQL | 3306 |
| Common backend / Jenkins | 8080 |

`netstat -lntp` lists which ports are currently listening and which process owns them (`l`=listening, `n`=numeric addresses instead of resolved hostnames, `t`=TCP, `p`=the owning process). Note: `netstat` is considered legacy on modern distros — `ss` (e.g. `ss -tulnp`) does the same job faster and is the current recommended tool, though `netstat` still works wherever it's installed.

## Process management
Everything that runs on Linux — even something as simple as `cat devops.txt` — creates a **process**: it starts, does its work, and completes (or keeps running, for a service).

Every process has a parent, forming a tree — much like an org chart (a Team Lead's process spawned a Senior Engineer's process, which spawned a Junior's, and so on). The process that spawns another is the **parent**; the spawned one is the **child**.

- `ps` — shows processes started by the current user, in the current session
- `ps -ef` — shows every process running on the server, in full detail (`e`=every process, `f`=full-format output, including PID and parent PID)
- A command normally runs in the **foreground**, blocking your terminal until it finishes. Adding `&` runs it in the **background** instead, so you get your terminal prompt back immediately:
  ```
  sleep 60 &
  ```

## Troubleshooting checklist
When something "isn't working," check these in order:
1. **Is the service running?** `systemctl status nginx`
2. **Is the port open?** `netstat -lntp` (or `ss -tulnp`)
3. **Is the process actually alive?** `ps -ef | grep nginx`
4. **What do the logs say?** — the last resort, but usually where the real answer is

## Common problems and how to solve them
A common mistake is running `usermod -G <group> <user>` (capital `-G` alone) intending to *add* a group, when it actually *replaces* every secondary group the user had. The fix is always using `-aG` (append) when adding to an existing set of groups, and reserving plain `-G` for when you deliberately want to reset the full list.

Another common one: editing `/etc/sudoers` directly with a normal text editor. A syntax mistake in that file can break `sudo` for everyone, including yourself. `visudo` avoids this because it won't let you save a file with a syntax error — always use it (or drop a new file in `/etc/sudoers.d/` instead of editing the main file).

A common assumption is that if `systemctl status` shows a service as running, the whole problem space is covered. It isn't — the service can be "running" while still not listening on the expected port (wrong config), or the process can be running but consuming all its resources without doing useful work. Checking service, port, and process independently (rather than just one of the three) is what actually confirms something is working end-to-end.

Another common mix-up: expecting `yum` and `dnf` to behave like entirely separate tools on a modern system. On current RHEL-family distros they're the same underlying tool — `yum` commands still work because it's symlinked to `dnf`, not because two package managers are installed side by side.

## Key takeaways
- Authentication proves identity; authorization decides what that identity is allowed to do. Both matter, and they're solved differently (keys/passwords vs. groups/permissions).
- Manage access through groups, not per-user permissions — assign a role's permissions to a group once, then add/remove users from that group.
- `usermod -aG` appends a secondary group; `usermod -G` (no `-a`) replaces the whole secondary group list — this trips people up constantly.
- Permissions (`chmod`) and ownership (`chown`) are different concepts — only the owner or root can change permissions, but only root can change ownership.
- The `wheel` group and a direct `/etc/sudoers.d` entry both grant sudo access — the sudoers file lets you be far more precise (e.g. specific commands, no password prompt) than just adding someone to a group.
- Always edit sudo config with `visudo`, never a plain editor — it validates syntax before saving so you can't lock yourself out.
- `.ssh` must be `700` and `authorized_keys` must be `600` — SSH will silently refuse a key set up with looser permissions than that.
- Offboarding order matters: cut off access (groups, lock, kill sessions) before you delete anything, but back up their data before you actually delete the account.
- Package management installs software; service management keeps it running; network management confirms it's reachable; process management confirms it's actually alive at the OS level — troubleshooting walks through all four.
- `dnf` (RHEL-family) and `apt-get` (Debian-family) do the same job with different syntax and package formats; `yum` is `dnf`'s predecessor, now just an alias for it.
- `systemctl enable` vs `start` — one controls whether a service survives a reboot, the other controls whether it's running right now. They're independent; you often want both.
- Every process has a parent (except the very first one) — this parent-child relationship is why killing a parent process can take its children down with it.
- The full health check for "is X really working" is service → port → process → logs, not just one of those.

See also: [04-linux.md](04-linux.md), [05-linux-commands.md](05-linux-commands.md)
