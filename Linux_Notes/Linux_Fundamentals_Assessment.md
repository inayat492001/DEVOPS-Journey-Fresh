**LINUX FUNDAMENTALS  
PRACTICAL ASSESSMENT**

*45 Questions • WSL Ubuntu or KillerCosa*

Instructions: Complete the questions in order using your Ubuntu
terminal. Try each task yourself. Or can work in groups

**Safety note:** Do not run destructive commands against system
directories. Avoid sudo for ordinary practice. User-management, service,
disk, LVM, and system-log commands may be restricted or behave
differently in WSL.

# Part A — Linux Basics & Navigation

**1. What is an operating system, and what role does Linux play?**

**2. What is a Linux distribution? Name any three from the course
material.**

**3. Display the current directory path.**

**4. Create a directory named linux-assessment inside your home
directory.**

**5. Move into linux-assessment and confirm your location.**

**6. Inside linux-assessment, create directories named files, logs, and
backup.**

**7. List contents including hidden files and detailed information.**

**8. Move one directory level up, then return to linux-assessment.**

# Part B — File & Directory Operations

**9. Create empty files app.log, users.txt, and notes.txt inside
files.**

**10. Add three lines of sample text to notes.txt.**

**11. Display notes.txt.**

**12. Copy notes.txt into backup.**

**13. Move users.txt into logs.**

**14. Create an empty directory named temporary, then remove it.**

**15. Create remove-me.txt and then delete only that file.**

**16. Display the first two lines of notes.txt.**

**17. Display the last two lines of notes.txt.**

**18. Search notes.txt for a word in your sample text.**

**19. Find every file named notes.txt under linux-assessment.**

# Part C — Text Processing, Pipes & Redirection

**20. Count lines, words, and characters in notes.txt.**

**21. Create names.txt with five names, one per line, and sort them
alphabetically.**

**22. Sort names.txt in reverse order.**

**23. Use awk to print the first field from a space-separated text
file.**

**24. Use sed to replace one word with another in a file's output.**

**25. Use a pipe to display only .txt entries from a directory
listing.**

**26. Save the output of ls -l into listing.txt.**

**27. Count lines in listing.txt by connecting commands with a pipe.**

# Part D — Permissions, Users & Links

**28. View the permissions, owner, and group of notes.txt.**

**29. Set notes.txt so the owner can read/write and others can only
read.**

**30. Add execute permission for the owner of test.sh.**

**31. Explain chown and write an example to change a file's owner.**

**32. Display your current username and user/group IDs.**

**33. Explain the purpose of adduser/useradd and passwd.**

**34. Create a hard link to a sample file and compare inode numbers.**

**35. Create a symbolic link and explain what happens if the target
moves or is deleted.**

# Part E — System Information, Monitoring & Networking

**36. Display the system hostname.**

**37. Display disk-space usage in human-readable form.**

**38. Show how much space a practice directory uses.**

**39. Display memory usage in a human-readable format.**

**40. Display system uptime and load averages.**

**41. Find the path of the ls executable.**

**42. Run df -h repeatedly with a two-second refresh, then exit.**

**43. Display network interface/IP information.**

**44. Look up DNS information for example.com.**

**45. Request HTTP headers from https://example.com and explain why curl
is useful in DevOps.**

# Final Mini Challenge

Inside linux-assessment, create a folder named final-task. Create
app.log with at least five lines, including ERROR on two lines. Use grep
to show only the ERROR lines, wc to count lines, head and tail to
inspect the file, chmod to set permissions to 644, and find to locate
app.log. Finally, show the directory structure and current path.

# Quick Command Checklist

man • mkdir • cd • ls • pwd • touch • rmdir • rm • cp • mv • cat •
nano/vi • history • clear • head • tail • grep • chmod • chown •
hostname • wc • sort • echo • find • adduser/useradd • passwd • id • awk
• sed • ln • ip • ping • top • df • du • wget • curl • nslookup • telnet
• lsof • ss • journalctl • lsblk • free • uptime • which • watch • sar •
pipes (|) • redirection (\>, \>\>) • LVM concepts • Cron concepts

Instructor note: LVM and Cron are assessed conceptually in this
worksheet. Do not have beginners run disk-modifying LVM commands on
their everyday WSL installation. Cron availability and service behavior
can vary in WSL; use a suitable Linux VM or server for a full practical.
