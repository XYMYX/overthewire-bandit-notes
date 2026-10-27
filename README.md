# overthewire bandit notes

my notes while working through [overthewire bandit](https://overthewire.org/wargames/bandit/).
no passwords and no full solutions, only approach, commands and learnings.

## level 0
- goal: log into the bandit server via ssh
- commands: `ssh bandit0@bandit.labs.overthewire.org -p 2220`, `ls`, `ls -la`, `man ls`
- what i learned: `ls -la` shows hidden files (names starting with a dot) plus owner, permissions, size and timestamp. the password prompt shows nothing while typing, that's a security feature.

## level 0 -> 1
- goal: read the password from the `readme` file in the home directory
- commands: `cat`, `exit`, `ssh bandit1@... -p 2220`
- what i learned: `cat` prints a file's content. each level is a new ssh login with the next user.

## level 1 -> 2
- goal: read a file whose name is just `-`
- commands: `cat` with a `./` path in front of the filename
- what i learned: a lone `-` as an argument means stdin/stdout for many commands, so `cat -` waits for input. giving the path `./` makes it clear you mean the file.

## level 2 -> 3
- goal: read a file whose name starts with `--` and contains spaces
- commands: `file` and `cat`, with the filename in double quotes and `./` in front
- what i learned: the shell splits arguments at spaces, quotes keep a name together. `./` must stand directly in front of the name with no space, otherwise it's two separate arguments.

## level 3 -> 4
- goal: find a hidden file inside the `inhere` directory
- commands: `ls -la`, `find inhere`, `cat`, `ctrl+c`
- what i learned: hidden files only show up with `ls -a` or `find`. an unclosed quote leaves the shell waiting at a `>` prompt, `ctrl+c` gets you out. linux is case sensitive and mac "smart quotes" don't work in the shell.

## level 4 -> 5
- goal: find the only human-readable file in the `inhere` directory
- commands: `file`, `cat`
- what i learned: `file` tells you what type a file really is (text or binary data), no matter what its name says.

## level 5 -> 6
- goal: find the one file in `inhere` that matches given properties, including a size of 1033 bytes
- commands: `find` with `-size` and the `c` suffix for bytes, `man find` (searching with `/`), `cat`
- what i learned: `find` searches by file properties. `-size n` means exactly n, `+n` bigger, `-n` smaller, and bytes are `c`, not `b`. every condition needs its option in front of the value.

## level 6 -> 7
- goal: find a file somewhere on the whole server with a given owner, group and size
- commands: `find /` with `-user`, `-group` and `-size`, `cat`
- what i learned: `/` is the root of the filesystem, so `find /` searches everything. lines with "permission denied" are errors, the real hit is a line with only a path.

## level 7 -> 8
- goal: find the password next to the word "millionth" in `data.txt`
- commands: `grep`
- what i learned: `find` looks at file properties, `grep` searches text inside files. grep is a key tool for searching through logs.

## level 8 -> 9
- goal: find the only line in `data.txt` that appears exactly once
- commands: `sort`, `uniq` with the unique option, pipe `|` (`sort data.txt | uniq ...`)
- what i learned: `uniq` only detects duplicates that are directly next to each other, so you sort first. the pipe sends one command's output into the next.

## level 9 -> 10
- goal: find a readable string preceded by several `=` signs in a mostly binary `data.txt`
- commands: `strings data.txt | grep ...`
- what i learned: `strings` pulls readable text out of binary files. the same trick reveals hardcoded passwords in compiled programs.

## level 10 -> 11
- goal: decode `data.txt`, which is base64 encoded
- commands: `base64 -d data.txt`
- what i learned: base64 is an encoding, not encryption. anyone can reverse it without a key.

## level 11 -> 12
- goal: decode `data.txt`, which is rot13 encoded
- commands: `cat` piped into `tr` (translate) with letter ranges
- what i learned: rot13 shifts every letter by 13 places. `tr` replaces characters from a first set with a second set, using ranges like `a-z`, and upper and lower case need their own ranges.

## level 12 -> 13
- goal: get the password from `data.txt`, a hexdump of a file that was compressed many times
- commands: `mktemp -d`, `cd`, `cp`, `xxd -r`, `file`, `mv`, `gunzip`, `bunzip2`, `tar -xf`, `ls`
- what i learned: work in a temp dir from `mktemp -d` because the home dir is not writable. `xxd -r` turns a hexdump back into the real file. always run `file` first, it tells you the real type (gzip, bzip2, tar), and magic bytes like `1f8b` at the start mark gzip. `gunzip` and `bunzip2` want the right file ending, so rename with `mv` first. `tar -xf` unpacks archives and can even handle gzip and bzip2 on its own, and each step gives a new filename, so check with `ls`. a command without a file argument (like `xxd -r`) waits for input, ctrl+c gets you out. don't `cat` binary files (if the terminal breaks, `reset` fixes it).
- magic numbers: every file format starts with the same first bytes, no matter the size or content. `file` reads these bytes, not the file ending. examples: gzip `1f 8b`, bzip2 `BZh`, pdf `%PDF`, png `.PNG`, linux program (elf) `.ELF`, windows program (exe) `MZ`. you can check with `xxd file | head -1`.
- security: the file ending is just a name. a `photo.jpg` that starts with `MZ` is really a windows program. this is deception (social engineering), not a vulnerability. always check the real type.

## level 13 -> 14
- goal: log in as bandit14 using a private ssh key instead of a password
- commands: `ls -la`, `file`, `cat`, `man ssh`, `exit`, `nano`, `chmod`, `ssh -i`
- what i learned: a private ssh key is a file that replaces the password. logging in from one level to the next via localhost is blocked, so the key has to be copied to your own machine (copy and paste into `nano`). `ssh -i` sets the key file (identity file). ssh ignores a key that others can read ("unprotected private key file", 0644 was too open), and `chmod 600` fixes it so only the owner can read and write. typing only a path makes the shell try to run it as a program ("permission denied"), so use `cat` to read a file. never put private keys in a repo.
- permissions: three digits for owner, group and others. how to calculate:
  1. drop the first character (`-` = file, `d` = directory).
  2. split the rest into three blocks of three characters: owner | group | others.
  3. each position has a value: 1st = `r` (read) = 4, 2nd = `w` (write) = 2, 3rd = `x` (execute) = 1, `-` = 0.
  4. add up the values inside each block, so each block becomes one single digit (add, don't write them next to each other).
  - example `-rw-r--r--`: `rw-` = 4+2+0 = 6, `r--` = 4+0+0 = 4, `r--` = 4+0+0 = 4, so 644
  - example `drwxr-xr-x`: `rwx` = 4+2+1 = 7, `r-x` = 4+0+1 = 5, `r-x` = 4+0+1 = 5, so 755
  - example `-rw-------`: `rw-` = 6, `---` = 0, `---` = 0, so 600 (only the owner can read and write, needed for private keys)
- platform: done on macos (terminal, `nano`, `chmod 600`). on windows it works differently: `chmod` does not exist in powershell, file permissions are set with acls (e.g. `icacls`). in wsl ubuntu `chmod` works like on linux, but keep the key in the linux home (`~`), not under `/mnt/c`, because permissions there don't behave the same.

## level 14 -> 15
- goal: submit the bandit14 password to port 30000 on localhost to get the next password
- commands: `nc host port`
- what i learned: `localhost` / `127.0.0.1` always means the machine the command runs on. over ssh that is the bandit server, not my own computer, and localhost traffic never leaves the machine. `nc` (netcat) opens a raw tcp connection to a host and port and sends text, unencrypted. host and port are plain arguments, `-p` is the ssh option and doesn't belong here

## level 15 -> 16
- goal: submit the bandit15 password to port 30001 on localhost, but over an encrypted ssl/tls connection
- commands: `man openssl-s_client` (searching with `/connect`), `openssl s_client -connect host:port`
- what i learned: `s_client` is a subcommand of `openssl`, so its manual is `man openssl-s_client`, not `man s_client`. with `-connect` host and port are written together with a colon (`host:port`), unlike `nc` where they are two separate arguments. after connecting, openssl prints a lot of certificate info first, wait until it stops, then send the password. `nc` sends plain text, `s_client` encrypts it with tls (like https). the password for the current level is the one i logged in with, no file needed. files starting with a dot are hidden and the dot is part of the name. names like `name` or `dir` in my cheat sheet are placeholders, don't type them literally.

## level 16 -> 17
- goal: find the one port between 31000 and 32000 on localhost that speaks ssl and returns the next credentials, then log in as bandit17
- commands: `nmap host -p range`, `nmap -sV -p port,port,...`, `openssl s_client -quiet -connect host:port`, `nano`, `chmod 600`, `ssh -i`
- what i learned: a port scan shows which "doors" (services) are open, it's the first step of every attack and every pentest. `-p 31000-32000` scans a range, `-p a,b,c` only chosen ports (no spaces). `-sV` talks to each service to detect what runs there, so it's much slower. `-sV` (version) is not `-sO` (protocol scan), case matters. the right port was the one with `ssl` and not `echo`. in interactive `s_client`, letters at the start of a line are commands (`k` = keyupdate), so a password starting with `k` never reached the server. `-quiet` turns on `-nocommands` and `-ign_eof` and hides the session info. an option and its value belong together, nothing goes in between (`-connect host:port`). the answer was a private ssh key, not a password: save it on my own machine (outside the repo), `chmod 600`, then `ssh -i`. the password itself is readable afterwards in `/etc/bandit_pass/`.

## cheat sheet

### commands
most commands run on the bandit server (linux), so they are the same no matter if you connect from mac or windows. only the local steps on your own computer are different. "local" = typed on your own computer, "server" = typed after logging in.

| command                                | where  | what it does                                                               | windows (powershell)                                             |
|----------------------------------------|--------|----------------------------------------------------------------------------|------------------------------------------------------------------|
| `ssh user@host -p port`                | local  | log in to a remote machine                                                 | same, openssh is built into windows 10/11                        |
| `ssh -i keyfile user@host -p port`     | local  | log in with a private key instead of a password                            | same                                                             |
| `exit`                                 | both   | log out / leave the current shell                                          | same                                                             |
| `ls` / `ls -la`                        | server | list files / list all files incl. hidden ones with details and permissions | locally: `ls` works, for hidden files use `Get-ChildItem -Force` |
| `cd dir`                               | both   | change directory (`~` = home, `..` = one up, `.` = here)                   | same                                                             |
| `cat file`                             | server | print a file (only for text files)                                         | locally: `cat` or `type`                                         |
| `cat ./-` / `cat "./name with spaces"` | server | read files with special names                                              | (server only)                                                    |
| `file name`                            | server | show the real file type (reads magic bytes, not the ending)                | (server only)                                                    |
| `find dir -size 1033c`                 | server | find files by size (`c` = bytes, `+n` bigger, `-n` smaller)                | (server only)                                                    |
| `find / -user x -group y`              | server | find files by owner and group                                              | (server only)                                                    |
| `grep word file`                       | server | find lines containing a word                                               | (server only)                                                    |
| `sort file \| uniq -u`                 | server | show lines that appear only once (`uniq` needs sorted input)               | (server only)                                                    |
| `strings file`                         | server | show readable text inside binary files                                     | (server only)                                                    |
| `base64 -d file`                       | server | decode base64 (encoding, not encryption)                                   | (server only)                                                    |
| `tr 'set1' 'set2'`                     | server | replace characters, e.g. for rot13                                         | (server only)                                                    |
| `mktemp -d`                            | server | create a temporary directory you can write to                              | (server only)                                                    |
| `cp source target`                     | server | copy a file                                                                | (server only)                                                    |
| `mv old new`                           | server | rename or move a file                                                      | (server only)                                                    |
| `xxd file` / `xxd -r dump out`         | server | make a hexdump / turn a hexdump back into the file                         | (server only)                                                    |
| `gunzip file.gz` / `bunzip2 file.bz2`  | server | unpack gzip / bzip2                                                        | (server only)                                                    |
| `tar -xf file`                         | server | unpack a tar archive                                                       | (server only)                                                    |
| `nc host port`                         | server | open a raw tcp connection and send text (unencrypted)                      | (server only)                                                    |
| `openssl s_client -connect host:port`  | server | open an encrypted ssl/tls connection and send text                         | (server only)                                                    |
| `nmap host -p 31000-32000`             | server | scan a port range for open ports                                           | (server only)                                                    |
| `nmap -sV -p port1,port2 host`         | server | detect which service runs on each port (slower)                            | (server only)                                                    |
| `openssl s_client -quiet -connect h:p` | server | tls connection without session info and without letter commands            | (server only)                                                    |
| `nano file`                            | both   | simple text editor (ctrl+o save, ctrl+x exit), on mac and linux            | `notepad file`                                                   |
| `chmod 600 file`                       | local  | set permissions (owner / group / others)                                   | no `chmod`, use `icacls` to remove access for other users        |
| `man command`                          | server | manual, search with `/word`, quit with `q`                                 | locally: `Get-Help command`                                      |
| `reset`                                | both   | fix a broken terminal                                                      | `cls` clears the screen                                          |

tip: with wsl ubuntu on windows, everything works like on linux and mac. keep your files in the linux home (`~`), not under `/mnt/c`.

### shortcuts
- `tab`: autocomplete file and command names (also works in powershell)
- `ctrl+c`: stop a running command or a waiting prompt (same on windows)
- `|` (pipe): send the output of one command into the next

### man page navigation
- `/word`: search forward from the current position
- `?word`: search backward
- `n` / `N`: next / previous hit
- `g` / `G`: jump to the top / bottom
- `q`: quit
- tip: if a search finds nothing, press `g` first and search again

### how i learn
- try alone first, read the level page and the hint files.
- read error messages carefully, they usually tell you what's wrong.
- use `man` and search with `/` before asking.
- always `file` before `cat` on unknown files.
- a command without a file argument often waits for input, `ctrl+c` gets you out.
- on mac, watch out for smart quotes and long dashes when copying commands. on windows, copy and paste in the terminal can be right click or ctrl+shift+v.
- write the note right after solving a level.
- never publish passwords, keys or full solutions.
