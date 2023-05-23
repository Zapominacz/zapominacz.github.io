---
title: "Port is already in use issue on the macOS"
date: 2023-05-20T20:23:00+02:00
authors: ["Mikołaj Wilczek"]
---
## TLDR
```bash
lsof -i:<port> | awk 'NR==2{print $2}' | xargs kill
```

Replace `<port>` with your port number.

## Why do you need it?

Many things happen on the computer during coding. Your favourite IDE wants to restart and update, the system requires a security update, or you already run that process, but you cannot find it in the crowd on the screen.

The best idea is to restart the computer then, but there are cases you cannot afford to do that. You can then find and terminate that process.

## How to do that?

You can use that command. The command should be available both on Linux and macOS:
```bash
lsof -i:<port> | awk 'NR==2{print $2}' | xargs kill
```

## What does it:
1. lsof is a list open files utility. Remember: in Linux (and macOS), everything is the file
2. `-i:<port>` selects the files (open connections) that use your TCP/UDP port
3. `awk` - one of the many command line text editors available and preinstalled
4. `NR==2` - select the second line (the first is the header)
5. `{print $2}` - prints second column - the process which uses your port.
6. `xargs` - helper utility. Allows to rewrite `kill $(lsof -i:3100 | awk 'NR==2{print $2}')` to the command above
7. `kill` - terminates the program

Remember that the process which uses your port is usually not the same one launched by you (usually, this is a child process), so you will only make this port available.

## Variations:
You can manually find that process using the command:
```bash
lsof -P -n -i:<port>
```
(`-P`, `-n` hides hostname and ports from the listing - they do not give practical information in this case)
And then terminate the process using
```bash
kill <process id (pid)>
```