# Linux Commands Reference

## What this is / why it matters
In Linux, everything you do happens through a command: `<command-name> <options> <inputs>`. Once that pattern clicks, learning a new command is just learning what it does and which options change its behavior — not memorizing syntax from scratch each time. This is a quick-reference for the commands you'll use constantly when working on a Linux server.

## Paths

- **Absolute path** — the full path from the root of the filesystem, works no matter where you currently are.
  ```
  cd /c/devops/daws-92s
  ```
- **Relative path** — a path from wherever you currently are.
  ```
  cd daws-92s
  ```
Same idea applies to any command that takes a file path — for example connecting with a key file:
```
ssh -i siva-92s ec2-user@107.23.134.115                       # relative
ssh -i /c/devops/daws-92s/siva-92s ec2-user@107.23.134.115    # absolute
```

## User and system info

- `whoami` — prints the current logged-in user
- `id` — prints the current user's UID, GID, and group memberships
  ```
  id
  uid=1000(ec2-user) gid=1000(ec2-user) groups=1000(ec2-user),4(adm),10(wheel)
  ```
- `uname` — prints system/kernel information
- `pwd` — prints the current directory (present working directory)

## Create, read, update, delete (CRUD) on files

- `touch <file>` — creates an empty file
  ```
  touch devops.txt
  ```
- `mkdir <folder>` — creates a directory
- `ls` — lists files and folders. Useful flags:
  - `ls -l` — long format (permissions, owner, size, date)
  - `ls -la` — long format, including hidden files
  - `ls -lr` — reverse alphabetical order
  - `ls -ltr` — long format sorted by time, most recent last
  ```
  -rw-r--r--.  1  ec2-user  ec2-user  0  Sep 23 02:08  devops.txt
  <permissions> <links>  <owner>     <group>  <size>  <date>       <name>
  ```
- `cat <file>` — prints a file's contents. Also used to create/edit short files directly from the terminal:
  ```
  cat > devops.txt
  Hi, I am learning DevOps
  ```
  Type the content, then press Enter and Ctrl+D to save.
  - `>` overwrites the file's contents
  - `>>` appends to the end instead of overwriting
- `cp <source> <destination>` — copies a file. Add `-r` to copy a folder and everything inside it.
- `mv <source> <destination>` — moves a file (this is also how you rename — moving a file to a new name in the same folder renames it).
- `rm <file>` — deletes a file. Add `-r` to delete a folder and its contents.
  ```
  rm aiops.txt
  rm -r aws
  ```

Linux is case-sensitive throughout — `Devops` and `devops` are two different names, whether for files, folders, or commands.

## Downloading and viewing content

- `wget <url>` — downloads a file from the internet and saves it locally
- `curl <url>` — fetches a URL and prints the content to the screen (doesn't save to a file by default)

## Searching and filtering

- `grep <word> <file>` — searches a file for lines containing a word
  ```
  cat 04-linux.md | grep -inv linux
  ```
  Common flags:
  - `-i` — case-insensitive
  - `-n` — show line numbers
  - `-c` — show a count of matches instead of the lines
  - `-v` — invert the match (show lines that *don't* match)
- `head <file>` — prints the first 10 lines by default; `head -n 4 <file>` prints the first 4
- `tail <file>` — prints the last 10 lines by default
- Piping (`|`) chains commands together, feeding one command's output into the next. To get lines 5–13 of a file:
  ```
  head -n 13 04-linux.md | tail -n 8
  ```
- `cut -d "<delimiter>" -f<field-number>` — splits each line on a delimiter and pulls out one field
  ```
  cut -d ":" -f1 /etc/passwd
  echo "https://raw.githubusercontent.com/.../04-linux.md" | cut -d "/" -f9
  ```
- `awk` — more powerful field-based text processing; `$NF` means "the last field"
  ```
  echo "https://raw.githubusercontent.com/.../04-linux.md" | awk -F "/" '{print $NF}'
  awk -F ":" '{print $1}' /etc/passwd
  awk -F ":" '$3 > 999 {print $1,$3}' /etc/passwd
  ```
  That last example lists usernames with a UID over 999 — Linux reserves UIDs 0–999 for system accounts, so anything above that is a manually created (human) user.

## Archiving

- `tar -czf <archive-name>.tar.gz <file1> <file2>` — bundles files into a compressed archive
  - `c` = create, `z` = gzip-compress, `f` = the archive filename
  ```
  tar -czf devops.tar.gz 04-linux.md devops.txt
  ```
- `tar -xzf <archive-name>.tar.gz` — extracts a `.tar.gz` archive

## Other commands worth knowing
`history` (past commands you've run), `echo` (print text), `clear` (clear the terminal screen).

## Editors

- `vim` ("visually improved" editor) has three modes:
  - **Insert mode** — actually typing/editing text
  - **Command mode** — the default mode, for moving around and issuing editor commands
  - **Esc** — takes you back to command mode from insert mode; from command mode, typing `:` opens the colon/command-line mode for saving, quitting, etc.

## Common problems and how to solve them
The most common early mistake is using `>` when you meant `>>` — `>` silently overwrites a file's entire contents, so redirecting output to an existing file by accident can wipe out data you needed. Default to `>>` unless you specifically want to replace the file.

Another common trip-up: assuming a relative path will work the same from any location. A relative path only resolves correctly from the directory you're actually in — if a script or command is run from a different working directory than expected, the same relative path can point to the wrong place or fail entirely. When in doubt (e.g. in scripts or cron jobs), use an absolute path.

## Key takeaways
- Every Linux action follows the same shape: `<command> <options> <inputs>` — learning a new command means learning its options, not a new syntax.
- Absolute paths always work regardless of your current location; relative paths depend on where you currently are. Prefer absolute paths in anything automated.
- `grep`, `cut`, and `awk` are the core text-filtering toolkit — `grep` finds lines, `cut`/`awk` pull specific fields out of them, and piping (`|`) lets you chain them together.
- UID 0–999 = system accounts, 1000+ = human-created accounts — this is how `awk -F ":" '$3 > 999 {print $1}' /etc/passwd` finds real user accounts.
- Linux is case-sensitive everywhere — filenames, commands, arguments.

See also: [04-linux.md](04-linux.md)
