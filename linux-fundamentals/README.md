# Linux Fundamentals — CTF Lab Write-ups

Hands-on Linux command-line challenges from the **CICSA** course labs (Part 1: Threats and Vulnerability Management). Each write-up covers the objective, the commands used and why, what went wrong along the way, and the security/SOC relevance of the skill.

> Flags are redacted (`CTF{REDACTED}`) in both the text and the screenshots. These write-ups document the method, not the answers.

**Status:** 🔄 in progress, updated as I complete more levels.

## Progress

| # | Level | Skill | Status | Write-up |
|---|---|---|---|---|
| 01 | Directory Traversal | `cd`, `ls`, `cat`, paths | ✅ Solved | [link](linux-basic-level-01-directory-traversal.md) |
| 02 | Hidden Files | dotfiles, `ls -a` | ✅ Solved | [link](linux-basic-level-02-hidden-files.md) |
| 03 | View Large File Contents | `head`, `tail`, `less`, `grep` | ✅ Solved | [link](linux-basic-level-03-view-large-file-contents.md) |
| 04 | File Permissions | `chmod`, rwx bits | ✅ Solved | [link](linux-basic-level-04-file-permissions-chmod.md) |
| 05 | Basic Grep Search | `grep`, `grep -o` on 10 MB noise | ✅ Solved | [link](linux-basic-level-05-basic-grep-search.md) |
| 06 | File Manipulation | `cp`, `mv`, directory permissions | ⛔ Lab misconfigured (diagnosed and reported) | [link](linux-basic-level-06-file-manipulation-cp-mv.md) |
| 07 | Symlink Following | `readlink`, `readlink -f` | ✅ Solved | [link](linux-basic-level-07-symlink-following.md) |
| 08 | Man Page Explorer | `man`, pager search | ✅ Solved | [link](linux-basic-level-08-man-page-explorer.md) |
| 09 | Tricky Filenames (Dashes) | `./-file`, `--` | ✅ Solved | [link](linux-basic-level-09-tricky-filenames-dashes.md) |
| 10 | Find in Large Directory Tree | `find -name`, `-exec` | ✅ Solved | [link](linux-basic-level-10-find-in-large-directory-tree.md) |
| 11 | I/O Redirection | stdout/stderr, `2>/dev/null` | ✅ Solved | [link](linux-beginner-level-11-io-redirection.md) |
| 12 | Unique Line Detection | `sort \| uniq -u`, stack counting | ✅ Solved | [link](linux-beginner-level-12-unique-line-detection.md) |
| 13 | Strings in Binary Blobs | `strings`, `file` | ✅ Solved | [link](linux-beginner-level-13-strings-in-binary-blobs.md) |

## Skills covered so far
- Navigating and searching the filesystem (`cd`, `ls -la`, `find`, `readlink`)
- Reading and filtering data (`cat`, `head`, `tail`, `less`, `grep`, `strings`, `sort`, `uniq`)
- Linux permissions for files and directories (`chmod`, `ls -ld`, ownership)
- Shell mechanics: pipes, stdout/stderr redirection, option parsing (`--`)
- Troubleshooting: diagnosing a broken lab (root-owned `700` directory) with evidence

## Why this matters for SOC work
Log triage (`grep`, `sort | uniq -c`), malware triage (`strings`, `file`), privilege-escalation awareness (permissions, symlinks, wildcard injection) and quiet scripting (`2>/dev/null`) are everyday analyst skills. These labs are the foundation.
