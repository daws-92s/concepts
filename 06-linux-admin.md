# Linux Administration: Users, Permissions, and Sudo Access

## What this is / why it matters
Anyone can run commands on a Linux box, but administering it means controlling *who* can do *what*. That comes down to three connected pieces: proving who someone is (authentication), deciding what they're allowed to touch (authorization), and the day-to-day commands that enforce both — creating users, assigning groups, and setting file permissions.

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
3. The admin adds that public key to the user's `authorized_keys` file on the server (typically `/home/<username>/.ssh/authorized_keys`).
4. The user connects with their private key:
   ```
   ssh -i ramesh-private-key ramesh@<server-ip>
   ```

## Common problems and how to solve them
A common mistake is running `usermod -G <group> <user>` (capital `-G` alone) intending to *add* a group, when it actually *replaces* every secondary group the user had. The fix is always using `-aG` (append) when adding to an existing set of groups, and reserving plain `-G` for when you deliberately want to reset the full list.

Another common one: editing `/etc/sudoers` directly with a normal text editor. A syntax mistake in that file can break `sudo` for everyone, including yourself. `visudo` avoids this because it won't let you save a file with a syntax error — always use it (or drop a new file in `/etc/sudoers.d/` instead of editing the main file).

## Key takeaways
- Authentication proves identity; authorization decides what that identity is allowed to do. Both matter, and they're solved differently (keys/passwords vs. groups/permissions).
- Manage access through groups, not per-user permissions — assign a role's permissions to a group once, then add/remove users from that group.
- `usermod -aG` appends a secondary group; `usermod -G` (no `-a`) replaces the whole secondary group list — this trips people up constantly.
- Permissions (`chmod`) and ownership (`chown`) are different concepts — only the owner or root can change permissions, but only root can change ownership.
- The `wheel` group and a direct `/etc/sudoers.d` entry both grant sudo access — the sudoers file lets you be far more precise (e.g. specific commands, no password prompt) than just adding someone to a group.
- Always edit sudo config with `visudo`, never a plain editor — it validates syntax before saving so you can't lock yourself out.

See also: [04-linux.md](04-linux.md), [05-linux-commands.md](05-linux-commands.md)
