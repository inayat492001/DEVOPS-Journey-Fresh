**Advanced Practical Assessment • 45 Scenarios + Capstone**

Adapted from the uploaded “Linux Fundamentals — Practical Assessment”.
Each original question (1–45) has been upgraded into a production-style
diagnostic, command-writing, or reasoning exercise. Answers
intentionally omitted for closed-book testing.

<table>
<tbody>
<tr class="odd">
<td>Recommended environment<br />
Ubuntu VM / WSL Ubuntu (nonproduction)</td>
<td>Suggested assessment<br />
45 × 2 points = 90; capstone = 10</td>
</tr>
</tbody>
</table>

## Assessment rules

Write the exact command(s), predict relevant results, and briefly
explain your reasoning. Assume GNU coreutils and Bash on Ubuntu unless a
scenario says otherwise. Use placeholders rather than real usernames or
production paths. No answer key is included.

Safety: do not run destructive commands on system directories, and do
not use sudo for normal exercises. Account, service, filesystem, network
and disk-related behavior can differ in WSL; use a disposable VM for
privileged experiments.

# PART A Linux Architecture, Navigation & Safe Automation

**01** A container on an Ubuntu host and a full Ubuntu VM both report
Linux in \`uname -s\`. Explain which components they share with the
host, which are isolated, and one operational consequence for incident
response.

**02** Your team deploys Bash scripts on Ubuntu, Debian and RHEL.
Identify the distribution-specific differences to check before
automating package installation, service management and firewall
configuration. Give two detection commands.

**03** You enter a directory through a symbolic link. \`$PWD\` shows
\`/home/dev/current\`, but its real directory is
\`/srv/releases/2026-10\`. Show how to display both logical and physical
paths, and explain why the distinction matters in deployment scripts.

**04** Write an idempotent command that creates
\`$HOME/linux-assessment\`, even if the directory already exists. Ensure
it works when \`$HOME\` contains spaces. Explain why hard-coding \`\~\`
inside double quotes would be unsafe.

**05** A deploy script must enter \`$HOME/linux-assessment\` or
terminate immediately if the directory does not exist. Give a safe
command sequence that changes directory, prints the resolved physical
path and does not continue in the wrong location.

**06** Inside \`$HOME/linux-assessment\`, create \`files\`, \`logs\` and
\`backup\` in a single idempotent command. Set a restrictive umask
first, and explain why changing the umask will not repair permissions on
directories that already exist.

**07** A release directory contains hidden dotfiles and names with
spaces. Show a command to inspect ownership, permissions, inode numbers
and hidden entries. Explain why \`ls \*\` is insufficient for auditing
this directory.

**08** You must move from a project directory to its parent, return to
the previous working directory without typing its path, then report the
physical location. Give the command sequence and explain one source of
unexpected behavior when symlinks are involved.

# PART B File Operations, Logging & Robust Search

**09** Create \`files/app.log\`, \`files/users.txt\` and
\`files/notes.txt\` without truncating any existing content. Explain how
your chosen command differs from shell redirection with \`\>\`.

**10** Append exactly three log lines to \`files/notes.txt\` using a
heredoc while preserving previous contents and literal dollar signs in
the input. Provide the command and explain your delimiter quoting.

**11** An application log contains blank lines, tabs and nonprinting
characters. Display it with line numbers including blank lines, then
provide a separate command that reveals invisible characters for
debugging.

**12** Copy \`files/notes.txt\` into \`backup/\` while preserving its
modification time and file mode. Specify the command, explain what it
preserves, and identify an extra concern if copying through symlinks.

**13** Move \`files/users.txt\` into \`logs/\`. Explain how the
operation changes when source and destination are on different
filesystems and what this may imply for inode identity and atomicity.

**14** A cleanup task may remove only an empty \`temporary\` directory,
never its contents. Give a command that fails safely if the directory is
nonempty, and explain why \`rm -r\` is inappropriate.

**15** A directory contains \`remove-me.txt\` and a filename beginning
with \`-\`. Write a safe command to delete only \`remove-me.txt\`, then
explain how to pass a leading-hyphen filename safely to \`rm\` without
broad wildcards.

**16** A 12 GB log file cannot be loaded into an editor. Display exactly
its first two lines using a streaming command. Explain why this is
preferable to printing the entire file.

**17** Show the latest two lines of \`app.log\`, then follow it through
log rotation where the file may be renamed and recreated. Name the
correct \`tail\` option and one limitation.

**18** Search \`notes.txt\` for the literal string \`error\[500\]\` with
matching line numbers. The square brackets must not behave as a regular
expression. Give the command and explain how you prevent option parsing
of the pattern.

**19** Find only regular files named \`notes.txt\` under
\`$HOME/linux-assessment\`, including nested directories. Provide a
null-delimited form suitable for paths containing whitespace or
newlines, and explain why parsing \`ls\` is dangerous.

# PART C Text Pipelines, Data Integrity & Redirection

**20** A UTF-8 file contains non-ASCII characters and a final line
without a newline. Show commands to count newline characters, words,
bytes and characters separately. Explain why byte and character counts
can differ and why \`wc -l\` may not match the visible number of text
lines.

**21** A names file contains duplicates and mixed-case entries. Produce
a separate, reproducibly sorted unique output using a fixed locale
without modifying the source. Explain what the chosen locale means for
ordering.

**22** Sort names in reverse lexicographic order without changing the
source file. Explain why this is not equivalent to reverse numeric order
for values such as \`2\` and \`10\`.

**23** A space- and tab-separated report has several consecutive
delimiters and occasional blank lines. Use \`awk\` to print the first
field of nonblank records. Explain why this would not correctly parse
quoted CSV data.

**24** Stream a log through \`sed\` and replace every occurrence of
\`timeout\` with \`retry\` in the output only, leaving the original file
untouched. Then explain what changes when you add an in-place edit
option.

**25** List only regular \`.txt\` files in the current directory (not
subdirectories), output their filenames, and pipe the results into
\`sort\`. State one limitation of newline-delimited filenames and how to
handle arbitrary names safely.

**26** Save a detailed listing of the current directory into
\`listing.txt\`, overwriting its previous contents. Give the command and
explain one subtlety about whether \`listing.txt\` itself may appear in
the result.

**27** Count nonempty lines in \`listing.txt\` using a pipe without
changing the file. Give a pipeline and explain why simply piping \`ls
-l\` to \`wc -l\` is not a reliable count of directory entries.

# PART D Permissions, Identity & Filesystem Links

**28** For \`notes.txt\`, report symbolic permissions, numeric mode,
owner, group and inode without editing it. Provide a command suitable
for scripted output, and explain what inspecting a symlink itself versus
its target changes.

**29** A shared report must be readable by everyone but writable only by
its owner. Set mode 644 using both octal and symbolic \`chmod\` forms.
Explain why granting write to the group or execute to ordinary data
files is inappropriate.

**30** Make \`test.sh\` executable by its owner while preserving
existing group and other permission bits. Show the command and explain
why blindly assigning mode 755 could violate least privilege.

**31** A root-operated provisioning task must change ownership of a file
to user \`deploy\` and group \`ops\`. Show the command, identify the
required privileges, and explain why the operation may be unavailable in
restricted WSL or containers.

**32** Your shell session seems to lack access despite an admin claim
that a group was added. Show commands to inspect the effective username,
UID, primary GID and supplementary groups. Explain why a new login
session may matter.

**33** On a disposable Linux VM, explain the practical differences
between \`adduser\` and \`useradd\`, how to create a home directory when
needed, and how password authentication and account locking differ. Do
not actually modify a production account.

**34** Create a hard link to a regular sample file, compare inode and
link counts, then explain what happens when the original filename is
deleted. Why would a hard link across different filesystems fail?

**35** Create a relative symbolic link \`logs/latest.log\` pointing to
\`../files/app.log\`. Explain precisely what happens if the target is
renamed, if the symlink is moved elsewhere, and how to detect a dangling
link.

# PART E Production Diagnostics, Monitoring & Networking

**36** A monitoring dashboard shows the host name of a VM, but its DNS
FQDN differs. Show how to retrieve the configured hostname and attempt
to resolve its FQDN. Explain why the two values need not match.

**37** A filesystem has free gigabytes but cannot create new small
files. Show how to inspect block usage and inode exhaustion, and explain
how each measurement can cause an out-of-space failure.

**38** An application reports low disk space even after a giant log file
was deleted. Compare \`du\` and \`df\`, show how to detect a deleted but
still-open file when available, and state the safe remediation
principle.

**39** A server shows almost no "free" RAM. Display human-readable
memory information and explain why the \`available\` field is often more
useful than the \`free\` field when investigating pressure.

**40** A 4-vCPU host reports load averages of 8.0, 6.0 and 3.0. Show
commands to retrieve uptime/load and CPU count. Explain why load average
is not CPU utilization and name one cause of elevated load despite
modest CPU usage.

**41** An interactive shell runs an \`ls\` alias while a script calls
\`/usr/bin/ls\`. Show commands to determine whether \`ls\` is an alias,
function or executable and to locate its executable path. Explain why
\`which\` alone can mislead.

**42** During a safe disk investigation, refresh filesystem block usage
every two seconds without editing any files. Give the interactive
command, state how to stop it and name one reason its output can be
misleading in WSL.

**43** A VM has multiple interfaces and routes. Display interface
addresses concisely, inspect the default route and determine which
interface the kernel would select toward \`1.1.1.1\`, without sending
network packets.

**44** A host can reach an IP address, but resolving \`example.com\`
fails. Give two distinct DNS-resolution troubleshooting commands (one
showing DNS records and one following the OS name-service path where
available), and explain what different results suggest.

**45** An HTTPS health check may redirect. Retrieve response headers
with \`curl\`, then make a separate request that follows redirects and
outputs only the final HTTP status code. Include a timeout, preserve TLS
verification, and explain why status alone is insufficient to prove
application health.

# CAPSTONE Incident-Response Mini Challenge

Scenario: Inside a disposable \`$HOME/linux-assessment\` workspace,
investigate recurring application errors while preparing evidence for a
teammate. Never assume you have root privileges.

1.  Create \`final-task/{logs,backup,reports}\` idempotently. Add at
    least eight timestamped sample lines to \`logs/app.log\` (include
    exactly three \`ERROR\` entries, one \`WARN\`, and at least one
    message containing spaces). Avoid erasing an existing log.

2.  Show only error lines with numbers, count only exact \`ERROR\`
    lines, and display the first and last two records. Explain whether
    your filter is case-sensitive.

3.  Copy the log into \`backup\` preserving modification time and mode;
    set the active log mode to 640, verify the permissions and determine
    which users or groups may read it.

4.  Find \`app.log\` files under the workspace without following
    symlinks, save a clean report of file metadata, and print the
    physical working directory.

5.  Use \`df\`, \`du\`, \`free\`, \`ip\`, a DNS tool and \`curl\` to
    write down a minimal host-health snapshot without modifying system
    settings. Identify two signs that require further investigation.

6.  Explain how your commands are safe to rerun, what can break under
    WSL, and one way to avoid accidentally inspecting the wrong
    directory.

## Scoring rubric

Each of 45 questions: 1 point for a correct and safe command or
technical conclusion; 1 point for precise explanation, expected behavior
or necessary caveat. Capstone: 10 points for completeness (4),
correctness (3), safety (2), and operational reasoning (1). 85–100 =
strong; 70–84 = competent; 50–69 = foundations need reinforcement; below
50 = revisit Linux fundamentals.
