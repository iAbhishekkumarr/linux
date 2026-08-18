# Linux Job-Ready Workbook — Questions Only
### Happy Minds AI Learning Center · Linux Practical Track
**Environment:** WSL2 · Ubuntu on Windows
**Rule:** No answers are printed anywhere in this file. You will find every answer yourself.

---

## 0. READ THIS BEFORE QUESTION 1

### 0.1 What this workbook is
This is not a tutorial. It is **330+ problems**, in the order a real Linux engineer meets them. Some are 30 seconds. Some will take you an hour and make you angry. Both are on purpose.

By the end you should be able to walk into an interview and answer any practical Linux question — not because you memorised commands, but because you have already fixed that exact kind of problem with your own hands.

### 0.2 The rules of this course
1. **Type every command. Never copy-paste.** Your fingers must learn this, not your clipboard.
2. **You are not allowed to ask for the answer** until you have spent 15 minutes on a problem.
3. **You ARE allowed** to use `man`, `--help`, and the internet — exactly like a real job.
4. **Every problem ends with proof.** "It works" is not an answer. Show the command whose output proves it.
5. **Keep a notebook file.** `~/notes.md`. Write every new command in *your own words*. Notes you copy are forgotten; notes you write are permanent.
6. **Break things on purpose.** You have a backup (section 1.1). Fear is the only thing that can stop you here.

### 0.3 The diagnostic loop — memorise this, use it on every single problem
```
1. READ THE ERROR      — out loud. It usually names the problem.
2. REPRODUCE           — make it fail on purpose, so you know when it stops failing.
3. ISOLATE             — is it permission? path? process? disk? network? config? service?
4. CHECK THE LOGS      — journalctl -u <service> -n 50   |   /var/log/
5. HYPOTHESIS          — write down ONE guess.
6. CHANGE ONE THING    — never two.
7. VERIFY END TO END   — prove it with a command, not a feeling.
8. WRITE IT DOWN       — one line in ~/notes.md.
```
Interviewers are not testing whether you know `chmod`. They are testing whether you have this loop. **Say it out loud while you work.**

### 0.4 Score yourself after every problem (out of 6)
| # | Did you... |
|---|---|
| 1 | Read the full error message before typing anything else? |
| 2 | Reproduce the failure yourself? |
| 3 | Use `man` / `--help` / logs instead of asking someone? |
| 4 | Change only one thing at a time? |
| 5 | Verify the fix with a command? |
| 6 | Explain *why* it worked, in your own words? |

**Fixing it by luck = 1/6. Failing but following the process = 5/6.** You are being trained on how you think.

### 0.5 The answer format you must use for every question
For every numbered problem, write in `~/notes.md`:
```
Q<number>
COMMANDS: <every command you ran>
ANSWER:   <the result / what was wrong>
WHY:      <one sentence in your own words>
```
This file is your revision material before the interview. It is also a portfolio piece. Do not skip it.

---

## 1. SET UP YOUR MACHINE (do this once)

### 1.1 Take a backup of WSL first
Open **Windows PowerShell**:
```powershell
wsl --list --verbose
wsl --shutdown
mkdir C:\wsl-backup
wsl --export Ubuntu C:\wsl-backup\ubuntu-clean.tar
```
To restore if you ever destroy the machine:
```powershell
wsl --unregister Ubuntu
wsl --import Ubuntu C:\wsl\Ubuntu C:\wsl-backup\ubuntu-clean.tar
```
**You now cannot permanently break anything. Behave accordingly.**

### 1.2 Enable systemd (required for the services module)
In Ubuntu:
```bash
sudo nano /etc/wsl.conf
```
Add:
```
[boot]
systemd=true
```
Save (`Ctrl+O`, Enter, `Ctrl+X`). Then in **PowerShell**: `wsl --shutdown`. Reopen Ubuntu and check:
```bash
systemctl is-system-running
```

### 1.3 Install the tools
```bash
sudo apt update
sudo apt install -y vim nano tree htop curl wget net-tools dnsutils lsof unzip zip cron nginx bsdmainutils shellcheck ncdu traceroute openssh-server jq
```

### 1.4 Build the practice server ("Namma Server")
```bash
nano ~/setup-namma.sh
```
Paste:
```bash
#!/bin/bash
set -e
sudo mkdir -p /srv/namma/{app,logs,backup,config,uploads,archive,reports}
sudo chown -R $USER:$USER /srv/namma

LOG=/srv/namma/logs/app.log
: > $LOG
for i in $(seq 1 600); do
  H=$(printf "%02d" $((RANDOM % 24))); M=$(printf "%02d" $((RANDOM % 60)))
  IPS=("10.0.0.5" "10.0.0.9" "192.168.1.44" "172.16.2.7" "10.0.0.5" "10.0.0.5" "203.0.113.12")
  IP=${IPS[$((RANDOM % 7))]}
  LVLS=("INFO" "INFO" "INFO" "WARN" "ERROR" "INFO" "ERROR" "DEBUG")
  L=${LVLS[$((RANDOM % 8))]}
  MSGS=("user login ok" "payment processed" "cache miss" "db timeout" "file not found" "order created" "connection refused" "disk write slow")
  MSG=${MSGS[$((RANDOM % 8))]}
  echo "2026-08-01 $H:$M:00 $IP $L $MSG" >> $LOG
done

W=/srv/namma/logs/access.log
: > $W
for i in $(seq 1 500); do
  IPS=("10.0.0.5" "10.0.0.9" "192.168.1.44" "172.16.2.7" "10.0.0.5" "203.0.113.12" "198.51.100.7")
  IP=${IPS[$((RANDOM % 7))]}
  CODES=("200" "200" "200" "404" "500" "301" "404" "403")
  C=${CODES[$((RANDOM % 8))]}
  P=("/index.html" "/login" "/api/order" "/images/logo.png" "/admin" "/api/user" "/checkout")
  U=${P[$((RANDOM % 7))]}
  M=("GET" "GET" "GET" "POST")
  ME=${M[$((RANDOM % 4))]}
  echo "$IP - - [01/Aug/2026:$(printf '%02d' $((RANDOM % 24))):00:00] \"$ME $U HTTP/1.1\" $C $((RANDOM % 5000))" >> $W
done

printf 'db_host=localhost\ndb_port=5432\ndb_user=namma\nmax_connections=100\ntimeout=30\ndebug=false\n# cache settings\ncache_ttl=600\n' > /srv/namma/config/app.conf
cp /srv/namma/config/app.conf /srv/namma/config/app.conf.backup
printf 'name,qty,price,category\nrice,20,60,grain\ndal,15,120,pulse\noil,8,140,oil\nsugar,30,45,grain\nsalt,50,20,spice\nwheat,12,55,grain\nghee,5,600,oil\n' > /srv/namma/app/stock.csv
printf 'id,name,city,salary\n1,Ravi,Kalaburagi,32000\n2,Priya,Bangalore,58000\n3,Kiran,Hyderabad,45000\n4,Anita,Kalaburagi,39000\n5,Suresh,Bangalore,72000\n' > /srv/namma/app/employees.csv
printf '#!/bin/bash\necho "Backup started at $(date)"\ntar -czf /srv/namma/backup/data.tgz /srv/namma/app 2>/dev/null\necho "Backup done"\n' > /srv/namma/backup/backup.sh
chmod +x /srv/namma/backup/backup.sh

mkdir -p /srv/namma/uploads/{2024,2025,2026}
for y in 2024 2025 2026; do for n in 1 2 3 4 5; do head -c 200000 /dev/urandom > /srv/namma/uploads/$y/photo_$n.jpg; done; done
head -c 60000000 /dev/urandom > /srv/namma/archive/old_dump.bin
echo "SECRET_KEY=namma123" > /srv/namma/app/.env
touch "/srv/namma/uploads/my report final.txt"
echo "Sandbox ready at /srv/namma"
```
Then:
```bash
chmod +x ~/setup-namma.sh && ~/setup-namma.sh
```
**Run this script again any time you want to reset everything.**

### 1.5 Install the Lab Engine — ⚠️ DO NOT OPEN THIS FILE
Some problems in Part 13 require the machine to be broken. The lab engine breaks it for you. **The engine is deliberately encoded — reading it is cheating, and you will only be cheating yourself out of a job.**

```bash
mkdir -p ~/labs && nano ~/labs/labs.b64
```
Paste the block from **Appendix A** at the end of this workbook, then:
```bash
base64 -d ~/labs/labs.b64 > ~/labs/labs.sh
chmod +x ~/labs/labs.sh
```
Usage:
```bash
sudo bash ~/labs/labs.sh break 1      # break scenario 1
sudo bash ~/labs/labs.sh restore 1    # put it back
```

---

# PART 1 — NAVIGATION & THE FILESYSTEM (Q1–Q28)

> **Before you start:** every answer here can be found with `man`, `--help`, or `tldr`. Do not search Google for the first 10 minutes. Learn to read the manual — that is a job skill in itself.

**Q1.** Without looking at your prompt, print the exact folder you are standing in right now.

**Q2.** List everything in `/etc` in long format, with human-readable sizes, newest file last. Then do it with newest file *first*.

**Q3.** Go to `/var/log`, then to your home folder, then return to `/var/log` using a **single command with no path in it**.

**Q4.** Explain in your notes what each of these means, then prove each one with a command: `.` , `..` , `~` , `-` , `/`

**Q5.** Navigate from `/srv/namma/logs` to `/srv/namma/config` using a **relative** path only (your command must not contain a leading `/`).

**Q6.** How many items are inside `/etc`? Now count only the directories. Now only the files.

**Q7.** There is a hidden file inside `/srv/namma/app`. Find it, display its contents, and explain in your notes what makes a file "hidden" in Linux.

**Q8.** Create this exact structure with **one single command**:
```
~/project/{src,tests,docs,data/{raw,processed,archive},logs}
```
*Prove it:* show the tree.

**Q9.** Delete the whole `~/project` tree. Before you do, run a command that shows you exactly what will be removed. Write down in your notes why this habit matters.

**Q10.** Create an empty file called `report.txt`, then create 10 files named `report_01.txt` through `report_10.txt` with **one command**.

**Q11.** Move all `report_0*.txt` files into a new folder `~/reports/` — one command, no filenames typed.

**Q12.** Rename `report.txt` to `final_report.txt`. There is no `rename` command in base Linux — which command do you use, and why does that make sense?

**Q13.** Copy the entire `/srv/namma/config` folder to `/tmp/config_copy`. Your first attempt will probably fail. Read the error and fix it.

**Q14.** Find every file on the system whose name contains `hosts`. Hide the permission-denied noise from the output.

**Q15.** Find all `.jpg` files under `/srv/namma` that were modified in the last 24 hours.

**Q16.** Find all files under `/srv/namma` **larger than 1 MB**, and print each one's size next to its path.

**Q17.** Find all **empty** files and all **empty** directories under `/srv/namma`.

**Q18.** Find every file under `/srv/namma` owned by your user, but only go 2 levels deep.

**Q19.** There is a file at `/srv/namma/uploads` whose name contains spaces. List it, copy it to `/tmp`, and delete the copy — **without renaming it**. Write down in your notes what you had to do differently and why.

**Q20.** Create a **symbolic link** at `~/applog` that points to `/srv/namma/logs/app.log`. Prove it works by reading the log through the link.

**Q21.** Create a **hard link** to the same file. Now list both links showing their inode numbers. What is different between them?

**Q22.** Delete the original `app.log` (make a copy first!). Which of your two links still works? Explain why in one sentence. Then restore the file.

**Q23.** Which command tells you what *type* of file something is, regardless of its extension? Use it on `/srv/namma/archive/old_dump.bin`, on `/bin/ls`, and on `/srv/namma/logs/app.log`.

**Q24.** Print the 20 most recently modified files anywhere under `/srv/namma`, newest first.

**Q25.** Show the total size of `/srv/namma`, and then a breakdown of each subfolder, sorted largest first.

**Q26.** Explain — in your notes, with a command that proves each — what lives in these folders: `/etc`, `/var/log`, `/tmp`, `/usr/bin`, `/opt`, `/home`, `/proc`, `/mnt/c`.

**Q27.** What is the difference between `/mnt/c/Users` and `/home` on your WSL machine? Prove your answer with two commands.

**Q28.** Try to `chmod 600` a file inside `/mnt/c/`. Then run `ls -l` on it. Explain what happened and why you must never do permission exercises there.

---

# PART 2 — READING & EDITING FILES (Q29–Q56)

**Q29.** Print the first 5 lines and the last 5 lines of `/srv/namma/logs/app.log` in **one command line**.

**Q30.** Open `app.log` in a pager. Inside it: jump to the end, jump back to the start, search for `ERROR`, jump to the next match, then quit. Write down every key you used.

**Q31.** Count the lines, words, and characters in `app.log` — three numbers, one command.

**Q32.** How many lines in `app.log` contain `ERROR`? Now answer the same question **two different ways** (one with a pipe, one without).

**Q33.** Show `app.log` with line numbers, but only lines 100 to 120.

**Q34.** Watch `app.log` live. In a second Ubuntu window, append a line to it and confirm you see it appear. What key stops the watching?

**Q35.** Watch `app.log` live but show **only** lines containing `ERROR` as they arrive.

**Q36.** What is the difference between `tail -f` and `tail -F`? Which one would you use on a production server, and why?

**Q37.** Print `/srv/namma/config/app.conf` with all comment lines and all blank lines removed.

**Q38.** Someone modified the config. Compare `app.conf` with `app.conf.backup` and report exactly which lines differ — do not read them line by line yourself.

**Q39.** Show the same comparison in side-by-side format. Then show it in the format Git uses.

**Q40.** Two files are 900 MB each. You only need to know *whether* they differ, not how. Which command, and why is it faster?

**Q41.** Append the line `retries=3` to `app.conf` without opening an editor. Then verify.

**Q42.** Overwrite `app.conf` with a single line, then restore it from the backup. Write in your notes what one character caused the destruction.

**Q43.** Run a command that produces both normal output and errors (try `ls /etc /nothing`). Send the normal output to `out.txt` and the errors to `err.txt`. Then send both to the same file.

**Q44.** Discard all error output from a command entirely. What is `/dev/null` and why does it exist?

**Q45.** Turn on `noclobber` in your shell. Try to overwrite an existing file with `>`. What happens? How do you force it? Turn it back off.

### Vim survival — you WILL need this in your first week on a job
**Q46.** Open `app.conf` in vim, change `max_connections=100` to `max_connections=250`, save, and quit.

**Q47.** Open it again, make any change, and quit **without saving**.

**Q48.** In vim: delete a whole line, undo it, redo it, copy a line and paste it below.

**Q49.** In vim: turn on line numbers, jump to line 5, jump to the end of the file, jump back to the top.

**Q50.** In vim: search for `db_port`, jump to the next occurrence, then replace **every** occurrence of `localhost` with `127.0.0.1` in the whole file with a single vim command.

**Q51.** You opened a file with vim by mistake and cannot get out. Write down the exact key sequence that always escapes, whatever state you are in.

**Q52.** Open a file in vim as read-only. How? Why would you do that on a production server?

**Q53.** Create a file with `cat` and a heredoc (no editor at all) containing 3 lines of text.

**Q54.** Print `stock.csv` as a neat aligned table in the terminal. (Hint: there is a command whose whole job is columns.)

**Q55.** Show the file `/etc/services` sorted alphabetically, with duplicates removed, only the first column, first 20 entries.

**Q56.** Create a file whose name starts with a dash, e.g. `-weird.txt`. Now delete it. This is a real interview trick question — write the solution in your notes.

---

# PART 3 — PERMISSIONS & OWNERSHIP (Q57–Q90)

> **This is the most-tested Linux topic for freshers.** Do every single one.

**Q57.** Run `ls -l` on `/srv/namma/backup/backup.sh` and write down, in your notes, what **every single character and column** of that output means.

**Q58.** Convert these to numbers without a computer, then verify with a command: `rwxr-xr-x`, `rw-r--r--`, `rwx------`, `rw-rw-r--`, `r--r-----`, `rwxr-x---`.

**Q59.** Convert these to letters: `755`, `644`, `600`, `700`, `640`, `751`, `777`, `440`.

**Q60.** Create a file and set its permissions to exactly `640` using **numeric** mode. Then achieve the identical result using **symbolic** mode (`u`,`g`,`o`,`+`,`-`,`=`).

**Q61.** Remove execute permission from `/srv/namma/backup/backup.sh` and try to run it. Read the error. Fix it. Now explain: which single letter was missing?

**Q62.** Run the same script a second way that works **even without** execute permission. Why does that work?

**Q63.** Why must you type `./script.sh` and not `script.sh`? Prove your answer with a command that shows the relevant setting.

**Q64.** Create a directory, remove its execute bit, and then try to `cd` into it and to read a file inside it. Write down exactly what `r`, `w` and `x` mean **on a directory** — they are not the same as on a file.

**Q65.** A file inside a folder has `rw-rw-rw-` but you cannot delete it. What permission do you actually need, and on what? Prove it with a test you build yourself.

**Q66.** Create a nested path `~/a/b/c/file.txt`. Remove the execute bit from `~/a/b` only. Try to read the file. Explain the result.

**Q67.** Change the owner of a file to `root`, then try to edit it as yourself. Then change it back. Which command, and why does it need `sudo`?

**Q68.** Change only the **group** of a file, leaving the owner alone. Do it two different ways.

**Q69.** Recursively set `/srv/namma/app` so that all **directories** are `755` and all **files** are `644` — without touching them one by one. (Doing this with a single `chmod -R 755` is the wrong answer. Explain why in your notes.)

**Q70.** Run `umask`. Now create a new file and a new directory and check their permissions. Explain the arithmetic that produced those numbers.

**Q71.** Change your umask so new files are created as `600` and new directories as `700`. Prove it. Then make the change permanent for your user.

**Q72.** Why do new files never get execute permission automatically, no matter what the umask is?

**Q73.** Run `ls -ld /tmp`. There is a letter at the end that is not `r`, `w`, or `x`. What is it, what does it do, and why does `/tmp` need it?

**Q74.** `/tmp` is world-writable (`777`). Explain why you still cannot delete another user's file there.

**Q75.** Find every file under `/srv/namma` that is world-writable. Why is this the first thing a security audit looks for?

**Q76.** Find every file on the system with the **setuid** bit set. Pick one and explain why it needs that bit.

**Q77.** Create a shared team directory where every file created inside it automatically belongs to a group called `nammadev`, and members can edit each other's files. Nobody outside the group gets any access. (You will need `groupadd`, `usermod`, `chown`, `chmod`, and one special bit.)

**Q78.** Prove your Q77 setup works: create two users, create a file as one, edit it as the other.

**Q79.** Explain the difference between setuid, setgid, and the sticky bit — and give one real example of each on your machine.

**Q80.** Generate an SSH key pair. Set the private key to `777`. Try to use it. Read the error carefully. What are the correct permissions for `~/.ssh`, the private key, the public key, and `authorized_keys`?

**Q81.** Someone tells you "just run `chmod 777` and it works". Write a 5-sentence answer, as if to an interviewer, explaining why you would refuse.

**Q82.** What permission would you set on: a shell script, a config file, an `.env` file with passwords, a web root directory, a private key? Justify each.

**Q83.** Copy a file with `cp` and then with `cp -p`. Compare the timestamps and permissions of the results. What does `-p` do and when does it matter?

**Q84.** Run `id`. Explain every field of the output. Then run `groups`. What is the difference between your **primary** group and your **secondary** groups?

**Q85.** Add yourself to a new group. Run `groups` immediately. Is the new group listed? If not — why not, and what are the two ways to make it appear?

**Q86.** Install ACLs (`sudo apt install acl`). Give exactly one specific user read access to a file that is otherwise `600`, without changing its owner or group. Then list the ACLs. When is this better than groups?

**Q87.** A directory is owned by `root:root` with mode `750` and your user is not in its group. List three different technically correct ways to give your app write access — and rank them from best to worst practice.

**Q88.** Make a file that even **you as the owner** cannot read. Then read it anyway. Explain what just happened about root and permissions.

**Q89.** Create a file, make it immutable with `chattr +i`, then try to delete it as root. What happens? How do you undo it? (Note: this may behave differently in WSL — record what you observe.)

**Q90.** Write in your notes a one-paragraph answer to: *"Explain Linux file permissions to me as if I have never used Linux."* You will be asked this in an interview.

---

# PART 4 — TEXT PROCESSING & LOG FORENSICS (Q91–Q148)

> **This is what the job actually is.** Nobody will ask you to install an operating system. They will hand you a 2 GB log and say "find out why it broke at 3 AM". Do every question here twice.

**Know your data first.** Run `head -1` on both log files and write down which field number holds what:
```
app.log     :  2026-08-01 14:23:00 10.0.0.5 ERROR db timeout
access.log  :  10.0.0.5 - - [01/Aug/2026:15:00:00] "GET /login HTTP/1.1" 404 1234
```

### grep — finding the needle
**Q91.** Print every line in `app.log` containing `ERROR`.

**Q92.** Do the same, but case-insensitively, and with line numbers.

**Q93.** Count `ERROR` lines two ways: with a pipe, and with a single grep flag.

**Q94.** Print every line that is **not** `INFO`.

**Q95.** Print every `ERROR` line **plus the 2 lines before and 2 lines after** it. Why is this the flag real engineers use most?

**Q96.** Print lines matching `ERROR` **or** `WARN`, using one grep. Now do it a second way.

**Q97.** Print only lines that **start with** `2026-08-01 09`.

**Q98.** Print only lines that **end with** the word `timeout`.

**Q99.** Print every blank line in a file. Now print every line that is *not* blank and *not* a comment.

**Q100.** Search every file under `/srv/namma` for the string `SECRET`. Now show only the **filenames**, not the matching lines.

**Q101.** Search only `.conf` files under `/etc` for the word `port`, ignoring permission errors.

**Q102.** Find files under `/srv/namma` that do **not** contain the word `ERROR`.

**Q103.** Match a whole word only: find `cat` without matching `category` or `concatenate`.

**Q104.** Search inside a **compressed** `.gz` log without decompressing it. Which command? (Make one first: `gzip -k /srv/namma/logs/app.log`.)

**Q105.** Write a grep that matches any line containing an IP address in the `10.0.0.x` range.

**Q106.** Write a grep that matches lines containing exactly 3 consecutive digits.

**Q107.** Explain in your notes, with an example each: `^` `$` `.` `.*` `[0-9]` `\.` `|` `+` `?`

### cut, sort, uniq — the counting pipeline
**Q108.** Print only the IP address column of `access.log`.

**Q109.** Print only the timestamp and log level from `app.log`.

**Q110.** Print all usernames on this system from `/etc/passwd` — one per line, nothing else.

**Q111.** Print the **top 5 IP addresses** in `access.log` with their hit counts, highest first. Build it **one pipe at a time**, showing the output after each stage. Write down each stage in your notes.

**Q112.** Why must you `sort` before `uniq`? Prove it by running the pipeline **without** the sort and comparing the results.

**Q113.** How many **unique** IPs visited? Answer with one line.

**Q114.** List IPs that appear **exactly once**. Then IPs that appear **more than once**.

**Q115.** Produce a count of every HTTP status code in `access.log`, ranked.

**Q116.** Which **page** returns the most 404s? Which **IP** generates the most 404s? What would the second answer mean on a real server?

**Q117.** Count how many requests each HTTP method (GET/POST) received.

**Q118.** Which **hour of the day** had the most ERROR lines in `app.log`?

**Q119.** For that busiest hour, print the actual raw log lines so you can read what happened.

**Q120.** Count each log level (INFO/WARN/ERROR/DEBUG) in `app.log`, ranked.

**Q121.** Find the 3 most common **error messages** (not levels — the message text) in `app.log`.

**Q122.** `sort` a file of numbers correctly. Then explain why plain `sort` puts `100` before `99`, and which flag fixes it.

**Q123.** Sort `stock.csv` by price, highest first, skipping the header.

**Q124.** Sort `employees.csv` by city, and within each city by salary descending.

### awk — columns and calculations
**Q125.** Print the 1st and 4th fields of `app.log`.

**Q126.** Print the **last** field of every line, without knowing how many fields there are.

**Q127.** Print how many fields each line of `access.log` has. Are they all the same?

**Q128.** Print only lines of `app.log` where field 4 is exactly `ERROR` (not just "contains ERROR" — there is a difference; explain it).

**Q129.** From `stock.csv`, print item name and total value (qty × price) for each row, skipping the header.

**Q130.** Calculate the **total value of all stock** in one command.

**Q131.** Calculate the **average salary** in `employees.csv`.

**Q132.** Print employees earning above the average salary. (Hint: you may need two passes, or store values in an array.)

**Q133.** Count how many employees are in each city, using **awk only** — no `sort`, no `uniq`.

**Q134.** Print `stock.csv` as a formatted table with aligned columns and a TOTAL row at the bottom.

**Q135.** Sum the bytes column in `access.log` to find the total data served. Convert it to MB.

**Q136.** Print line numbers 50 to 60 of `app.log` using awk. Now do the same with `sed`. Now with `head` and `tail`.

**Q137.** Using awk, print only lines where the status code is 5xx (server errors).

### sed — changing text
**Q138.** Preview replacing `localhost` with `10.0.0.50` in `app.conf` **without modifying the file**.

**Q139.** Now apply it for real, keeping an automatic backup of the original.

**Q140.** Explain the difference between `s/a/b/` and `s/a/b/g`. Prove it on a line containing the same word twice.

**Q141.** Delete all comment lines and all blank lines from a copy of `app.conf`, in one sed command.

**Q142.** Print only lines 10–20 of a file using sed (and nothing else).

**Q143.** Print all lines between the first occurrence of `09:00` and the first occurrence of `10:00` in `app.log`.

**Q144.** Add the prefix `[NAMMA] ` to the beginning of every line in a file.

**Q145.** Find every file under `/srv/namma/config` containing `localhost` and replace it in all of them with one command line. **Run the "which files" part alone first** and say why in your notes.

**Q146.** Replace a path like `/srv/namma/logs` with `/var/log/namma` using sed. The slashes will cause a problem — solve it two different ways.

### Putting it together
**Q147.** In one pipeline: from `access.log`, for requests that returned 500, list the top 3 URLs with counts.

**Q148.** Write a single command that produces a mini traffic report showing: total requests, unique IPs, number of errors (4xx+5xx), and the busiest hour. (It may take you 20 minutes. That is fine. This is the exact task a junior gets in week one.)

---

# PART 5 — DISK & STORAGE (Q149–Q176)

**Q149.** How full is each filesystem on this machine? Which column would you look at first at 2 AM?

**Q150.** What is `/mnt/c` in that output, and why should you ignore it when debugging the Linux disk?

**Q151.** Check inode usage. What is an inode, and how can a disk be "full" with free space?

**Q152.** Show the total size of `/srv/namma`, then a per-subfolder breakdown sorted largest first.

**Q153.** Drill down repeatedly until you find the single largest **file** under `/srv/namma`. Write down the loop of commands you used — it is the same loop every time.

**Q154.** Do the same job with an interactive tool. Then explain why you must still be able to do it without that tool.

**Q155.** Find every file on the whole system larger than 100 MB.

**Q156.** Find every `.log` file under `/var/log` not modified in the last 30 days. Then write (but do not run) the command that would delete them, and explain the safety step you must take first.

**Q157.** Find the 10 largest files under `/srv` and print them with sizes, largest first.

**Q158.** `du -sh /srv/namma` and `ls -l` on the same folder show different numbers. Why?

**Q159.** Run `sudo bash ~/labs/labs.sh break 8`. Now: `df` shows space consumed that `du` cannot account for, and the file no longer exists. Find the cause, name the process, and **recover the space without rebooting**. Then explain the whole mechanism in your notes. *(This is a senior-level interview question.)* Restore with `restore 8`.

**Q160.** A 500 MB log file is being written to right now by a running application. Free the space **without deleting the file** and **without restarting the app**. Give three commands that all achieve this.

**Q161.** Why is `rm` the wrong way to empty an in-use log file? Connect your answer to Q159.

**Q162.** Configure `logrotate` for `/srv/namma/logs/*.log`: rotate daily, keep 7, compress, and do not require an app restart. Test it without waiting a day.

**Q163.** Which logrotate option is the one that avoids restarting the application, and what does it actually do?

**Q164.** Find which directory under `/var` contains the most **files** (not the most bytes). Write the loop yourself.

**Q165.** Run `sudo bash ~/labs/labs.sh break 12`. Investigate what changed on disk, identify **both** problems it created, and fix them. Restore afterwards.

**Q166.** Clear the apt package cache and report how much space you recovered.

**Q167.** What are the top 5 places disk space hides on a real Linux server? Prove each one exists on your machine with a command.

**Q168.** Create a 20 MB file filled with zeros. Then one filled with random data. Which command, and what is the difference between `/dev/zero` and `/dev/urandom`?

**Q169.** Create a 40 MB file, format it as ext4, mount it as a filesystem, write a file into it, then unmount it. (This is how you practise disks without a second hard drive.)

**Q170.** Mount that same image with only 64 inodes. Fill it with empty files until it refuses. Confirm with a command that the problem is inodes, not space.

**Q171.** Show the mount options currently in use for `/`. What does `rw,relatime` mean?

**Q172.** What is swap, how much do you have, and how would you check whether the machine is actively swapping? (Note the WSL-specific answer.)

**Q173.** Compare `du -h`, `du -sh`, `du -ah`, and `du --max-depth=1`. Write one line each in your notes.

**Q174.** Sort `du -h` output correctly by size. Plain `sort -n` gives the wrong answer — which flag fixes it, and why?

**Q175.** Write a one-line command that prints `WARNING` if the root filesystem is more than 80% full, and `OK` otherwise.

**Q176.** Explain, as if to an interviewer, your complete step-by-step procedure when a server reports "disk full". List your commands in order.

---

# PART 6 — PROCESSES & PERFORMANCE (Q177–Q208)

**Q177.** List every process running on the machine, with owner, CPU and memory.

**Q178.** Explain every column of `ps aux` in your notes. Which column shows *real* memory usage, and which one is misleading?

**Q179.** What do the STAT letters `R`, `S`, `D`, `Z`, `T` mean?

**Q180.** List only your own processes. Now only processes owned by `root`.

**Q181.** Find the PID of your current shell. Now find the PID of *its* parent.

**Q182.** `ps aux | grep nginx` includes the grep command itself in the output. Why? Give two different ways to stop that happening.

**Q183.** Show the process tree of this machine. What is PID 1, and why does it matter?

**Q184.** Start a CPU-heavy background process (`bash -c 'while true; do :; done' &`). Find it using three different tools.

**Q185.** Show the top 5 processes by CPU, then the top 5 by memory, without using an interactive tool.

**Q186.** Open `top`. Sort by memory, then by CPU, then show full command lines, then filter to your user, then quit. Write down every key.

**Q187.** What is "load average"? What number would count as overloaded on **this** machine? Find the command that tells you how many cores you have.

**Q188.** Kill your CPU-hog process politely. Then start another and kill it forcefully. What is the difference in what the process experiences?

**Q189.** List all signals available. What are signals 1, 2, 9, 15, 18, 19?

**Q190.** Which signal does `Ctrl+C` send? Which does `Ctrl+Z` send? Prove both.

**Q191.** Start `sleep 500`, suspend it, resume it in the background, then bring it back to the foreground, then kill it. Write down every command.

**Q192.** Kill a process by **name** instead of PID — two different commands. What is the danger of doing this on a shared server?

**Q193.** Start a long-running command so that it survives you closing the terminal. Do it three different ways and write down when you would use each.

**Q194.** Install `tmux`. Start a session, run a command, detach, close the terminal completely, reopen Ubuntu, and reattach to your still-running session.

**Q195.** Start a process listening on port 8080 (`python3 -m http.server 8080 &`). Now find out exactly what is using port 8080 — command, PID and user — using two different tools.

**Q196.** Free that port. Before you kill it, run a command to inspect what it actually is. Why is that step non-negotiable?

**Q197.** List every port this machine is listening on, with the process behind each.

**Q198.** What is the difference between a socket bound to `0.0.0.0:5432` and one bound to `127.0.0.1:5432`? Which one causes "my app can't reach the database"?

**Q199.** Create a zombie process on purpose. Show it in `ps`. Then try to kill it. Explain the result.

**Q200.** What is an orphan process, and what happens to it? Prove it by watching a child's PPID change.

**Q201.** Run `free -h`. Explain every column. Which one tells you whether you actually need more RAM?

**Q202.** Why does Linux deliberately keep almost no "free" memory? Answer in one sentence you could say in an interview.

**Q203.** Find out whether the kernel has ever killed a process for using too much memory on this machine. Which log, which command?

**Q204.** Show live memory usage refreshing every 2 seconds. Two different ways.

**Q205.** Use `vmstat` and `iostat` (install `sysstat`). Which columns show CPU wait on disk, and swap activity?

**Q206.** Find which process has a specific file open (`/srv/namma/logs/app.log`). Now find every file a given process has open.

**Q207.** Look inside `/proc/<PID>/`. Find that process's working directory, its command line, its environment variables, and its open file descriptors.

**Q208.** Run a command with lower CPU priority using `nice`. Then change the priority of an already-running process with `renice`. What range are the values, and which end is "nicer"?

---

# PART 7 — USERS, GROUPS, SUDO & SSH (Q209–Q240)

**Q209.** Who are you? Show your username, UID, primary group and all secondary groups — one command.

**Q210.** Create a user `ravi` with a home directory and bash as the login shell, using the **portable** command (not the Debian-only interactive one). Set a password.

**Q211.** Find `ravi`'s line in `/etc/passwd` and explain **all seven fields** in your notes.

**Q212.** Why is there no password in `/etc/passwd`? Where is it, and who can read that file? Prove it.

**Q213.** What is the difference between a UID below 1000 and above 1000? Find three system accounts on this machine and say what they are for.

**Q214.** Which users on this machine have a real login shell, and which are deliberately blocked from logging in? Write a single command that answers this.

**Q215.** Switch to `ravi` using `su ravi`. Check `$HOME`, `pwd` and `$PATH`. Now exit and switch using `su - ravi` and check the same three things. Explain the difference the dash makes and why it causes real bugs.

**Q216.** Which password does `su` ask for? Which does `sudo` ask for? Why is `sudo` preferred in a team?

**Q217.** Find the log file that records every `sudo` command on this machine. Show the last 10 entries.

**Q218.** Create a group `qa`. Add `ravi` and yourself to it. Prove membership three different ways.

**Q219.** What is the difference between `usermod -G` and `usermod -aG`? What exactly goes wrong if you forget the `a`? (Do not test this on your own account.)

**Q220.** You added yourself to a group but `groups` does not show it. Explain why, and give two ways to make it take effect.

**Q221.** Change `ravi`'s **primary** group. Which flag? Now create a file as `ravi` and check which group it belongs to.

**Q222.** Lock `ravi`'s password. Prove it is locked by inspecting `/etc/shadow` and with a status command.

**Q223.** A locked password is **not enough** to stop someone logging in. Explain what else you must disable, and do it.

**Q224.** Set `ravi`'s shell to `nologin` and expire his account. Then show his full account-ageing report.

**Q225.** Write, in your notes, a complete 8-step **employee offboarding checklist** for a Linux server. You will be asked this in interviews.

**Q226.** Show who is currently logged in and what they are running. Now kill all of `ravi`'s sessions.

**Q227.** Set a password policy so `ravi` must change his password every 30 days with a 7-day warning.

**Q228.** Grant `ravi` permission to run **only** `systemctl restart nginx` with sudo, without a password, and without giving him any other root access.

**Q229.** Which command must you use to edit sudoers, and what disaster does it prevent? What file permissions must a sudoers file have?

**Q230.** Verify exactly what `ravi` is allowed to run. Then log in as him and prove that one allowed command works and one disallowed command is refused.

**Q231.** Why would granting sudo access to `vim`, `less`, or `find` be equivalent to giving full root? Demonstrate the escape on your own machine.

**Q232.** Which group grants admin rights on Ubuntu? What is it called on RHEL/CentOS?

**Q233.** Generate an ed25519 SSH key pair. Identify which file is private and which is public, and state the rule about each in one sentence.

**Q234.** Install your public key so you can `ssh localhost` with no password. Do it manually (not with `ssh-copy-id`) so you understand what is happening.

**Q235.** Why is it safe to give away the public key? Explain the challenge–response mechanism in your own words, as if to an interviewer.

**Q236.** Break your key by setting the wrong permissions on it. Read the error. Fix it. List the required permissions for `~/.ssh`, the private key, the public key and `authorized_keys`.

**Q237.** Create a `~/.ssh/config` entry so that `ssh namma` connects to a host with a specific user, port and key. Test it.

**Q238.** Connect with maximum verbosity and read the negotiation. Where does the server log failed login attempts?

**Q239.** Find every failed SSH login attempt on this machine, and rank the source IPs by number of attempts. (Combine Part 4 skills.)

**Q240.** Harden SSH: disable root login and disable password authentication in `sshd_config`. Test the config **before** restarting the service. Which command tests it?

---

# PART 8 — SERVICES & SYSTEMD (Q241–Q268)

> Requires systemd enabled (section 1.2). Verify with `systemctl is-system-running` before starting.

**Q241.** Install nginx, start it, and prove it is serving a page using a command-line tool only.

**Q242.** Read the full output of `systemctl status nginx` and explain **every line** in your notes.

**Q243.** Stop nginx and prove it is stopped in two different ways (one with systemctl, one without).

**Q244.** Change the homepage text to `Namma Server is alive`. Your first attempt with `sudo echo ... >` will fail — explain why, and give the correct command.

**Q245.** What is the difference between `restart` and `reload`? Which would you use after a config change on a live site, and why?

**Q246.** Run `sudo bash ~/labs/labs.sh break 11`. nginx is running right now. Answer: **will the website survive a reboot?** Prove your answer with a command, then fix it. Restore afterwards.

**Q247.** Explain the four possible combinations of `is-active` and `is-enabled`, and which combination causes 3 AM outages.

**Q248.** What does `systemctl enable` physically do on the filesystem? Find the evidence.

**Q249.** List all currently running services. Then all enabled-at-boot services. Then all **failed** services.

**Q250.** Run `sudo bash ~/labs/labs.sh break 2`. The website is down. Diagnose it using your loop, fix it, and write the whole investigation into `~/notes.md` as an incident report. Restore afterwards.

**Q251.** Run `sudo bash ~/labs/labs.sh break 1`. Different symptom, different cause. Note that the service is **running**. Find the real problem. Restore afterwards.

**Q252.** Which command validates an nginx config before you restart? Find the equivalent for `sshd` and for a bash script.

**Q253.** Show the last 50 log lines for nginx. Then only today's. Then only errors. Then follow them live.

**Q254.** What are the journalctl priority levels, and which flag filters by them?

**Q255.** Show all kernel messages. Show all logs since the last boot. Show logs from the previous boot.

**Q256.** How large is the systemd journal on this machine, and how would you limit it?

**Q257.** Some applications do **not** use journalctl. Find nginx's own access and error log files and tail them.

**Q258.** Where does Ubuntu keep the general system log and the authentication log? What are the equivalent files on RHEL/CentOS?

**Q259.** Write a script `/usr/local/bin/namma-monitor.sh` that prints disk usage and a timestamp every 30 seconds in an infinite loop.

**Q260.** Turn it into a real systemd service that starts at boot, restarts automatically if it crashes, runs as a non-root user, and logs to journalctl.

**Q261.** Explain every line of your unit file in your notes: `[Unit]`, `After=`, `[Service]`, `Type=`, `ExecStart=`, `Restart=`, `RestartSec=`, `User=`, `[Install]`, `WantedBy=`.

**Q262.** You edited the unit file and nothing changed. Which command did you forget?

**Q263.** Prove the auto-restart works: find the main PID, kill it with `-9`, wait, and show that it came back with a new PID.

**Q264.** Make your service start only **after** the network is up and only **after** nginx. Which directives?

**Q265.** Pass an environment variable to your service two different ways (inline and from a file).

**Q266.** Why must `ExecStart` use an absolute path? Test what happens if it does not.

**Q267.** Stop, disable and completely remove your service, leaving the system clean. List every step.

**Q268.** Explain, as if to an interviewer, the difference between systemd, init scripts, and `service` — and why `systemctl` replaced them.

---

# PART 9 — NETWORKING (Q269–Q300)

**Q269.** Show this machine's IP addresses and network interfaces. What is `lo` and why does it matter?

**Q270.** What does `/20` or `/24` after an IP mean? What is this machine's subnet?

**Q271.** Show the routing table. Which line is the default gateway, and what does "default" mean here?

**Q272.** Find this machine's **public** IP as the internet sees it. Explain why it differs from the IP in `ip a`.

**Q273.** What is the difference between the WSL IP and your Windows IP? Prove it (check `ipconfig` in PowerShell).

**Q274.** Show which DNS servers this machine uses. Which file?

**Q275.** Resolve `google.com` to an IP three different ways.

**Q276.** Query a specific DNS server directly instead of your default one. Why is this useful when debugging?

**Q277.** Look up the mail servers (MX) and nameservers (NS) for a domain.

**Q278.** Watch the full resolution chain from the root servers downward. Which flag?

**Q279.** Run `sudo bash ~/labs/labs.sh break 10`. `namma-api.local` now resolves to an IP that does not exist, but DNS knows nothing about it. Find where the answer is coming from, explain the resolution order, and fix it. Restore afterwards.

**Q280.** Which file decides whether `/etc/hosts` or DNS is consulted first? Show its contents.

**Q281.** Add an entry to `/etc/hosts` that makes `namma.local` point at `127.0.0.1`, then prove it works in a browser or with curl. Then remove it.

**Q282.** List every listening port on this machine with the owning process. Memorise the flag combination.

**Q283.** Explain the socket states `LISTEN`, `ESTAB`, `TIME-WAIT`, `CLOSE-WAIT`. Which one, in large numbers, indicates an application bug?

**Q284.** Count how many sockets are in each state, ranked. (Part 4 pipeline again.)

**Q285.** Fetch a web page with `curl`. Then only its headers. Then both. Then follow redirects.

**Q286.** Print **only** the HTTP status code of a URL and nothing else.

**Q287.** Print the status code **and** the total response time in one line.

**Q288.** Send a POST request with a JSON body and a custom header.

**Q289.** Download a file with `wget` and with `curl`. What is the practical difference between the two tools?

**Q290.** Resume an interrupted download with each tool.

**Q291.** Download a file and verify its integrity with a checksum. Then modify one byte and show the checksum change.

**Q292.** Write a one-line health check that prints `UP` if a URL returns 200 and `DOWN` otherwise.

**Q293.** What ports are these: 22, 25, 53, 80, 443, 3306, 5432, 6379, 8080, 27017? Which file on your machine lists them all?

**Q294.** Test whether a specific TCP port on a remote host is open, without a browser — three different tools.

**Q295.** Ping a host 4 times and interpret every number in the output. Then explain why `ping` failing does **not** prove a server is down.

**Q296.** Trace the network path to `8.8.8.8`. What does a row of asterisks mean?

**Q297.** Check the firewall status on this machine. Allow port 80 and then remove that rule.

**Q298.** A user reports "the website is down". Write down, **in order**, the first five commands you would run and exactly what each one rules in or out. This is a top-5 interview question — rehearse it out loud.

**Q299.** `curl http://localhost` works on the server but external users cannot reach it. List every possible cause you can think of, and the command that would test each one.

**Q300.** What is the difference between a private IP and a public IP? Which ranges are private? Why does your phone and your laptop both have `192.168.1.x`?

---

# PART 10 — SHELL SCRIPTING (Q301–Q345)

> **Truth to remember:** a script is just the commands you already know, saved in a file. Everything else exists only to make that file smarter.

**Q301.** Write `health.sh` that prints the hostname, current date/time, uptime, disk usage of `/` and memory usage — each on a labelled line.

**Q302.** Run it. It will fail. Read the error and fix it. Which earlier question was this exactly the same as?

**Q303.** What is the shebang, why must it be the first line, and what changes if you write `#!/bin/sh` instead of `#!/bin/bash`?

**Q304.** Run your script three ways: `./health.sh`, `bash health.sh`, `source health.sh`. Write down the difference between the third one and the other two, and prove it with a script that does `cd /tmp`.

**Q305.** Make `health.sh` runnable from any directory just by typing `health`. Two different ways.

**Q306.** Write a script that takes a name and a city as arguments and greets the user. If no arguments are given, it must print a usage message and exit with a non-zero code.

**Q307.** Explain in your notes: `$0`, `$1`, `$#`, `$@`, `$*`, `$?`, `$$`. Write a script that prints all of them.

**Q308.** Demonstrate the bug caused by an unquoted variable containing a space. Then fix it. Write the rule in your notes.

**Q309.** Show the difference between `"$var"`, `'$var'`, and `${var}_backup` with a working example of each.

**Q310.** Write a script that asks the user for their name interactively, then for a password without echoing it to the screen.

**Q311.** Write `check.sh <path>` that reports whether the path is a file, a directory, or does not exist. If it is a file, also report whether it is empty and whether you can write to it.

**Q312.** List and test at least 8 file-test operators (`-f -d -e -s -r -w -x -L`).

**Q313.** Why does `[ -f $file ]` break but `[ -f "$file" ]` work? Why are the spaces inside the brackets mandatory?

**Q314.** Write a script that compares two numbers and prints which is bigger. Now do the same with two strings. Which operators change, and what happens if you mix them up?

**Q315.** What does `[ "$count" > 100 ]` actually do? Run it, then run `ls`. Explain the surprise.

**Q316.** Rewrite one of your `if` statements as a one-liner using `&&` and `||`.

**Q317.** Write a `case` statement that takes an argument `start|stop|status|restart` and prints a different message for each, with a default for anything else.

**Q318.** Write a `for` loop that prints numbers 1 to 10. Now 1 to 20 in steps of 5. Now using C-style syntax.

**Q319.** Loop over every `.log` file in `/srv/namma/logs` and print the filename, its size, and its ERROR count as an aligned table.

**Q320.** Make that loop safe when there are **no** matching files. What happens without the guard?

**Q321.** Write a `while` loop that retries a command up to 3 times with a 5-second gap, stopping as soon as it succeeds.

**Q322.** Read `stock.csv` line by line in pure bash (no awk), splitting each line into variables, and print a calculated total per row. Skip the header.

**Q323.** Count the lines of a file using a `while read` loop piped from `cat`, then print the count **after** the loop. The number will be wrong. Explain why, then fix it.

**Q324.** Why is `for line in $(cat file)` the wrong way to read lines? Prove it with a file containing spaces.

**Q325.** Write a function `mkcd` that creates a directory and enters it in one step. Add it permanently to your shell.

**Q326.** Write a function that takes a filename and an optional line count (defaulting to 20) and shows that many error lines from the log.

**Q327.** Explain and demonstrate `$?`. What does 0 mean? Why is that backwards from what you'd expect?

**Q328.** Write a script that exits with code 0, 1, or 2 depending on what happened, and prove each exit code from the shell.

**Q329.** What do `set -e`, `set -u` and `set -o pipefail` do individually? Write a small script that misbehaves without each one and behaves with it.

**Q330.** Demonstrate how `set -u` would prevent `rm -rf $DIR/` from becoming `rm -rf /`.

**Q331.** Write a logging function that prefixes every message with a timestamp and writes to both the screen and a log file simultaneously.

**Q332.** Write a **production-quality backup script** for `/srv/namma/app` that:
  1. Uses `set -euo pipefail`
  2. Keeps configuration in variables at the top
  3. Creates a dated `.tar.gz` archive
  4. Checks the source exists before starting, and fails clearly if not
  5. **Verifies the archive is readable** after creating it
  6. Deletes all but the newest 7 backups
  7. Logs every step with timestamps
  8. Exits with distinct codes for distinct failures

**Q333.** Test your backup script's failure paths: run it with the source folder missing, with the destination not writable, and with the disk path wrong. Does it fail loudly and correctly every time?

**Q334.** Run your backup script 10 times in a row. Prove that only 7 archives remain.

**Q335.** Write a script that checks whether a list of websites (read from a file) is up, and prints a report.

**Q336.** Write a script that finds and reports all files in a directory larger than a size given as an argument.

**Q337.** Write a script that reads `employees.csv` and produces a per-city salary summary.

**Q338.** Write a menu-driven script (using `case` and a `while` loop) offering: check disk, check memory, check services, show top processes, exit.

**Q339.** Use arrays: build a list of services, loop over it, and report whether each is active.

**Q340.** Write a script that accepts flags like `-v` (verbose) and `-f <file>` using `getopts`.

**Q341.** Debug a script with `bash -x`. Explain what the `+` lines are showing you.

**Q342.** Check a script for syntax errors **without running it**. Which flag?

**Q343.** Run `shellcheck` on every script you have written so far. Fix every warning and record what you learned.

**Q344.** Someone hands you this script. Find all five bugs **without running it**, then confirm with tools:
```bash
#!/bin/bash
LOGDIR = "/srv/namma/logs"
count=0
for f in $LOGDIR/*.log
do
    errors=$(grep -c "ERROR" $f)
    count=$count + $errors
done
echo "Total errors: $count"
if [ $count > 100 ]
then
    echo "Too many errors!"
fi
```

**Q345.** Rewrite that script correctly, with `set -euo pipefail`, proper quoting, and a guard for missing files. Explain why `grep -c` returning 0 matches can abort a script under `set -e`, and how to handle it.

---

# PART 11 — CRON & AUTOMATION (Q346–Q360)

**Q346.** Start the cron service on WSL and make it start at boot.

**Q347.** Schedule your `health.sh` to run every minute, appending its output to a log file. Verify it actually ran.

**Q348.** Decode each of these schedules and write them in your notes: `0 2 * * *`, `*/5 * * * *`, `30 9 * * 1-5`, `0 */4 * * *`, `0 0 1 * *`, `@reboot`.

**Q349.** Write cron expressions for: every Sunday at 3 AM; every 10 minutes between 9 AM and 6 PM; the first day of every quarter at midnight.

**Q350.** List your cron jobs. Then list another user's. Where are system-wide cron jobs stored?

**Q351.** Which single command deletes **all** your cron jobs with no confirmation? How would you protect yourself against typing it by accident?

**Q352.** Run `sudo bash ~/labs/labs.sh break 9`. A cron job has been installed that works when you run the script by hand, but produces nothing from cron. Find out why. **Prove** the cause with evidence, do not guess. Then fix it and confirm it works. Restore afterwards.

**Q353.** Schedule `env > /tmp/cron-env.txt` to run once. Compare that output with `env` in your terminal. Write down every difference you find, and explain the consequences.

**Q354.** List **four** distinct reasons a script can work manually but fail under cron. Give the fix for each.

**Q355.** Why must every cron line end with `>> /path/to/log 2>&1`? What happens to the output if you leave it out?

**Q356.** A cron command containing `date +%F` fails. Why? What character is special in crontab, and how do you handle it?

**Q357.** Your script uses relative paths and breaks under cron. Which one line at the top of the script fixes this permanently?

**Q358.** Schedule a job with `at` to run once, 2 minutes from now. How does `at` differ from cron?

**Q359.** Create a **systemd timer** that does the same job as one of your cron entries. List two advantages timers have over cron.

**Q360.** Where does cron itself log the fact that it ran a job? Find evidence of your jobs running.

---

# PART 12 — PACKAGES, ARCHIVES & ENVIRONMENT (Q361–Q385)

**Q361.** What is the difference between `apt update` and `apt upgrade`? Which one changes installed software?

**Q362.** Search for a package, show its details, install it, list the files it installed, and then find which package owns `/usr/bin/dig`.

**Q363.** What is the difference between `apt remove` and `apt purge`? Which one leaves config files behind?

**Q364.** List every manually installed package on this machine. Then find packages that are no longer needed.

**Q365.** Where does apt get its packages from? Show the file(s).

**Q366.** What is the relationship between `apt` and `dpkg`? Install a `.deb` file directly and observe what happens with dependencies.

**Q367.** Name the package manager for: Ubuntu, RHEL/CentOS, Fedora, SUSE, Alpine. Why does Alpine matter to you as a Docker user?

**Q368.** Compress `/srv/namma/config` into `config.tar.gz`. Then list its contents **without extracting**. Then extract only one file from it into `/tmp`.

**Q369.** What does each letter mean in `tar -czvf` and `tar -xzvf`? Invent your own memory trick and write it in your notes.

**Q370.** Why is it `.tar.gz` and not just `.gz`? What does each of the two tools actually do?

**Q371.** Create an archive that **excludes** all `.log` files.

**Q372.** Extract an archive into a specific directory. Why should you always list an unknown archive before extracting it?

**Q373.** Compress a single log file with `gzip`. What happened to the original? How do you keep it?

**Q374.** Read and search inside a `.gz` file without decompressing it. Which two commands? Why does this matter for rotated logs?

**Q375.** Create a `.zip` archive and extract it. When would you choose `.zip` over `.tar.gz` and why is it a bad choice for system backups?

**Q376.** Run `sudo bash ~/labs/labs.sh break 7`. A working program exists on this machine but typing its name gives "command not found". Find the program, explain the cause, and make it runnable from anywhere in a way that survives a reboot. Restore afterwards.

**Q377.** Print your `$PATH` and explain how the shell uses it. Why is the current directory deliberately not in it?

**Q378.** Compare `which`, `whereis`, `type` and `command -v`. Which one reveals aliases and shell builtins? Test each on `ls`, `cd`, and `python3`.

**Q379.** Add a directory to your PATH temporarily, then permanently. What is the danger of writing `export PATH="$HOME/bin"` instead of `export PATH="$PATH:$HOME/bin"`?

**Q380.** Which startup file runs for a new terminal tab, and which runs at login? On WSL, which one applies when you open Ubuntu? Prove it by adding an `echo` to each.

**Q381.** Create 8 useful aliases and make them permanent. Include one for tailing your app log.

**Q382.** Write a shell **function** that takes arguments — something an alias cannot do. Make it permanent.

**Q383.** How do you run the real `ls` when an alias is shadowing it? Two ways.

**Q384.** Show all environment variables. Explain the difference between a shell variable and an exported environment variable, with a demonstration using a child shell.

**Q385.** Set a variable for a single command only (without exporting it or affecting your shell). Prove the variable is gone afterwards.

---

# PART 13 — BREAK-FIX INCIDENT DRILLS (L1–L12)

**How to run a drill:**
1. Take a WSL backup first if you are nervous: `wsl --export Ubuntu C:\wsl-backup\before.tar` in PowerShell.
2. Run `sudo bash ~/labs/labs.sh break <N>`. **Do not read the script.**
3. Read only the **symptom** below — nothing else.
4. Set a **20-minute timer**.
5. Work through the diagnostic loop out loud. Write every command in `~/notes.md` as you go.
6. When fixed, run `sudo bash ~/labs/labs.sh restore <N>`.
7. Write an incident report (format at the end of this section).

**You are being judged on the investigation, not the fix.** A wrong hypothesis backed by evidence beats a lucky guess.

---

**L1 — `break 1`**
*Symptom:* "Customers say the website is not loading."
*Extra rule:* Before you restart anything, prove whether the service is running. If it is running, restarting it is the wrong move — find out why.

**L2 — `break 2`**
*Symptom:* "The website went down after a config change last night. Nobody remembers what was changed."
*Extra rule:* You must name the exact file **and line number** of the fault before you fix it.

**L3 — `break 3`**
*Symptom:* "The nightly backup log says `Permission denied`. Nothing was deployed."
*Extra rule:* Fixing this with `777` scores zero. Justify the number you choose.

**L4 — `break 4`**
*Symptom:* "The upload feature stopped working. Users get an error when saving files."
*Extra rule:* Identify **who** the process runs as before you change anything.

**L5 — `break 5`**
*Symptom:* "The server has become very slow and the disk is filling up fast."
*Extra rule:* You must find **both** problems. And you must free the disk space **without deleting the log file**.

**L6 — `break 6`**
*Symptom:* "New joiner `labuser` has the correct password (`Lab@12345`) but cannot get a working session."
*Extra rule:* There is more than one fault. Do not stop at the first one — verify the **whole** workflow: log in, land in the home folder, create a file.

**L7 — `break 7`**
*Symptom:* "The `namma-report` tool worked yesterday. Today it says command not found."
*Extra rule:* Prove the program still exists and works before you conclude anything.

**L8 — `break 8`**
*Symptom:* "`df` says the disk is filling but `du` cannot find the files. Nothing was deleted... allegedly."
*Extra rule:* Reboot is not an acceptable answer. Recover the space with the process still running if you can.

**L9 — `break 9`**
*Symptom:* "A scheduled task produces nothing. But when I run the script by hand it works perfectly."
*Extra rule:* You must produce **evidence** of the difference, not just a theory.

**L10 — `break 10`**
*Symptom:* "`namma-api.local` points to the wrong server. Our DNS team says they have no such record."
*Extra rule:* Explain the full name-resolution order in your report.

**L11 — `break 11`**
*Symptom:* "Everything looks fine right now — but the last two times we rebooted, the site never came back."
*Extra rule:* You must prove the fault with a command **before** you fix it.

**L12 — `break 12`**
*Symptom:* "Disk alerts are firing and the mail queue is misbehaving."
*Extra rule:* There are two separate storage problems here, and they are **not** the same kind of problem. Find both.

---

### Incident report format — write one for every drill
```
INCIDENT: L<n>
1. SYMPTOM REPORTED
2. INVESTIGATION      (every command, and what each told you)
3. ROOT CAUSE         (plain English, one paragraph)
4. FIX APPLIED        (exact commands)
5. VERIFICATION       (the command that proves it works, with output)
6. PREVENTION         (what would stop this happening again: monitoring, logrotate,
                       enable at boot, config validation in CI, alerting)
7. TIME TAKEN
```
**Section 6 is what makes you employable. A junior fixes. An engineer prevents.** Keep these reports — bringing printed incident reports to an interview beats any certificate.

---

# PART 14 — TIMED SPEED DRILLS (D1–D60)

**Rules:** 60 seconds per question. One command line each. No searching. Do the whole set, then repeat it a week later and beat your time.

| # | Task |
|---|---|
| D1 | Show your current directory |
| D2 | List all files including hidden, long format, human sizes |
| D3 | Go to the previous directory you were in |
| D4 | Create a nested folder path 3 levels deep in one command |
| D5 | Count the files in `/etc` |
| D6 | Find all `.conf` files under `/etc` |
| D7 | Show the last 20 lines of `/var/log/syslog` |
| D8 | Follow a log file live |
| D9 | Count `ERROR` lines in `app.log` |
| D10 | Show lines 40–50 of a file |
| D11 | Print only the 3rd column of a space-separated file |
| D12 | Print only the 2nd field of a comma-separated file |
| D13 | Sort a file numerically, descending |
| D14 | Remove duplicate lines from a file |
| D15 | Count occurrences of each unique line |
| D16 | Top 5 most frequent IPs in `access.log` |
| D17 | Replace all `foo` with `bar` in a file, in place, with a backup |
| D18 | Delete all blank lines from a file |
| D19 | Sum a numeric column with awk |
| D20 | Print the last field of every line |
| D21 | Show disk usage of all filesystems, human readable |
| D22 | Show the size of the current folder only |
| D23 | Top 5 largest subfolders, sorted |
| D24 | Find files bigger than 100 MB |
| D25 | Find files modified in the last 2 days |
| D26 | Empty a log file without deleting it |
| D27 | Check inode usage |
| D28 | List all processes with CPU and memory |
| D29 | Top 5 processes by memory |
| D30 | Find the PID of a process by name |
| D31 | Kill a process gracefully |
| D32 | Force-kill a process |
| D33 | Show what is listening on port 80 |
| D34 | Show all listening ports with process names |
| D35 | Show memory usage human readable |
| D36 | Show system uptime and load |
| D37 | Make a script executable |
| D38 | Set a file to 644 |
| D39 | Set a directory to 755 recursively (directories only) |
| D40 | Change a file's owner and group |
| D41 | Show your UID, GID and groups |
| D42 | Add a user to a group without removing others |
| D43 | Lock a user account |
| D44 | Show who is logged in |
| D45 | Start a service |
| D46 | Make a service start at boot |
| D47 | Show a service's last 30 log lines |
| D48 | Show all failed services |
| D49 | Test an nginx config |
| D50 | Show this machine's IP addresses |
| D51 | Show the default gateway |
| D52 | Resolve a domain to an IP |
| D53 | Print only the HTTP status code of a URL |
| D54 | Download a file and save it with a chosen name |
| D55 | Compress a folder to `.tar.gz` |
| D56 | List the contents of a `.tar.gz` without extracting |
| D57 | Extract a `.tar.gz` into `/tmp` |
| D58 | Search inside a `.gz` log file |
| D59 | Show your PATH |
| D60 | Show the exit code of the last command |

---

# PART 15 — THE VERBAL INTERVIEW BANK (V1–V100)

**How to use this:** laptop **closed**. Answer out loud, in under 60 seconds, in full sentences. Record yourself on your phone and listen back — you will hear the "umm" and the vagueness. **Knowing the answer and being able to say it are two different skills, and only one of them gets you hired.**

### Fundamentals
**V1.** What is the Linux kernel, and how is it different from a Linux distribution?
**V2.** Name five Linux distributions and one thing each is used for.
**V3.** Explain the Linux filesystem hierarchy. What lives in `/etc`, `/var`, `/usr`, `/opt`, `/proc`?
**V4.** What is the difference between an absolute and a relative path?
**V5.** What does "everything is a file" mean in Linux?
**V6.** What is a shell? Name three. What is the difference between `sh` and `bash`?
**V7.** What is the difference between a login shell and a non-login shell?
**V8.** What is an inode?
**V9.** What is the difference between a hard link and a soft link?
**V10.** What happens, step by step, when you type `ls` and press Enter?

### Files & text
**V11.** How would you find a file when you don't know where it is?
**V12.** What is the difference between `find` and `locate`?
**V13.** How do you search for text inside all files in a directory tree?
**V14.** What is the difference between `>` and `>>`?
**V15.** What does `2>&1` mean, and why does the order matter?
**V16.** What is `/dev/null` used for?
**V17.** Explain what a pipe does, with an example.
**V18.** What is the difference between stdout and stderr, and why are they separate?
**V19.** How do you view a 5 GB log file safely?
**V20.** How would you find the top 10 most frequent entries in a log file?
**V21.** What is the difference between `grep`, `sed` and `awk`?
**V22.** How do you replace text across many files at once, safely?
**V23.** How do you read a compressed log file without decompressing it?
**V24.** How do you count lines, words and errors in a file?
**V25.** How do you compare two config files?

### Permissions & users
**V26.** Explain Linux file permissions to someone who has never used Linux.
**V27.** What does `chmod 755` mean, and when do you use it?
**V28.** What do `r`, `w` and `x` mean on a **directory**?
**V29.** Why can you delete a read-only file? What permission actually controls deletion?
**V30.** What is umask?
**V31.** What is the sticky bit, and why does `/tmp` use it?
**V32.** What are setuid and setgid? Give a real example.
**V33.** Why is `chmod 777` a bad answer in an interview?
**V34.** Difference between `su`, `su -` and `sudo`?
**V35.** Where are user accounts stored, and where are the passwords?
**V36.** What are the seven fields of `/etc/passwd`?
**V37.** Difference between a primary group and a secondary group?
**V38.** How do you give a user permission to run exactly one command as root?
**V39.** Why must you use `visudo`?
**V40.** How would you offboard an employee from a Linux server?
**V41.** How does SSH key authentication work?
**V42.** Why is it safe to share a public key?
**V43.** What permissions must `~/.ssh` and the private key have, and why?
**V44.** How would you harden an SSH server?
**V45.** What is the difference between authentication and authorization?

### Processes & performance
**V46.** What is the difference between a program, a process and a thread?
**V47.** What is PID 1?
**V48.** What is a zombie process, and how do you kill one?
**V49.** What is an orphan process?
**V50.** Difference between `kill` and `kill -9`? Which do you use first?
**V51.** What are signals? Name five.
**V52.** What does load average mean, and what value is "too high"?
**V53.** How do you find what is using 100% CPU?
**V54.** How do you find what is using all the memory?
**V55.** Why does `free` show almost no free memory on a healthy Linux server?
**V56.** What is the OOM killer and how would you know it acted?
**V57.** What is swap, and is swapping good or bad?
**V58.** How do you run a job that survives you logging out? Give three ways.
**V59.** What is `nice`, and what range are its values?
**V60.** How do you find which process is using a particular file or port?

### Disk & storage
**V61.** A server reports "disk full". Walk me through your process.
**V62.** `df` says full, `du` says there is nothing. What is happening?
**V63.** You get "No space left on device" but `df -h` shows free space. What now?
**V64.** How do you empty a log file that an application is actively writing to?
**V65.** What is logrotate and which option avoids restarting the app?
**V66.** What is the difference between `df` and `du`?
**V67.** How would you find the 10 largest files on a server?
**V68.** What is a mount point? How do you see current mounts?
**V69.** What is `/proc` and is it on your hard disk?
**V70.** What is a filesystem? Name three Linux ones and one difference between them.

### Services & boot
**V71.** What is systemd, and what did it replace?
**V72.** Difference between `systemctl start` and `systemctl enable`?
**V73.** A service won't start. Walk me through your diagnosis.
**V74.** Difference between `restart` and `reload`?
**V75.** What is a unit file? Name five directives and what they do.
**V76.** How do you make a service restart automatically if it crashes?
**V77.** You edited a unit file and nothing changed. What did you forget?
**V78.** How do you read logs for one specific service, from today, errors only?
**V79.** Describe the Linux boot process in the order things happen.
**V80.** What is a runlevel / systemd target?

### Networking
**V81.** How do you find a machine's IP address? How is that different from its public IP?
**V82.** What is the difference between a private and a public IP address?
**V83.** How does a hostname become an IP address? What is the resolution order?
**V84.** What is `/etc/hosts` and when would you use it deliberately?
**V85.** How do you check what is listening on a port?
**V86.** A service is bound to 127.0.0.1 instead of 0.0.0.0. What breaks?
**V87.** Difference between TCP and UDP? Give an example of each.
**V88.** What ports do SSH, HTTP, HTTPS, MySQL and PostgreSQL use?
**V89.** What does a 500 error mean versus a 404 versus a 403?
**V90.** "The website is down." Give me your first five commands and what each rules out.

### Scripting & automation
**V91.** What is a shebang and why does it matter?
**V92.** What does `$?` return and what does 0 mean?
**V93.** What does `set -euo pipefail` do, and why is it in every good script?
**V94.** Why must you quote variables in bash?
**V95.** Difference between `$@` and `$*`?
**V96.** Difference between `[ ]` and `[[ ]]`?
**V97.** Your script works manually but fails in cron. Why?
**V98.** How do you debug a bash script?
**V99.** Explain the five fields of a crontab line.
**V100.** Write, out loud, the logic of a script that backs up a folder daily and keeps only 7 copies. No syntax needed — just the steps and the checks you would include.

---

# PART 16 — CAPSTONE PROJECTS (C1–C6)

These are portfolio pieces. Do them properly, keep the code and the documentation, and put them on your CV and GitHub.

---

**C1 — Server Health Reporter**
Write a script that produces a formatted health report: hostname, uptime, load average, disk usage per filesystem, memory, top 5 processes by CPU and memory, count of failed services, and number of ERROR lines in the app log in the last hour. Output must be readable in the terminal **and** appendable to a log file. Schedule it hourly. Add a `--email` flag placeholder.
*Deliverables:* the script, a README explaining every function, sample output, a crontab entry.

**C2 — Log Analysis Toolkit**
Write a script that takes a web access log as an argument and prints: total requests, unique visitors, requests per hour (as a text bar chart), top 10 URLs, top 10 IPs, status-code breakdown with percentages, all 4xx/5xx grouped by URL, and a "suspicious activity" section listing IPs with more than 20 4xx responses.
*Deliverables:* the script, README, and a written analysis of what your tool found in `access.log`.

**C3 — Backup & Restore System**
Build a pair of scripts: `backup.sh` and `restore.sh`. Backup must be dated, compressed, verified after creation, retention-limited to 7 copies, and fully logged with distinct exit codes. Restore must list available backups, let the user choose one, and restore safely without destroying current data. Schedule the backup nightly.
*Deliverables:* both scripts, README, a test log proving you tested every failure path.

**C4 — User Onboarding & Offboarding Automation**
Write `onboard.sh <username> <team>` that creates the user, sets the shell and home permissions, creates and assigns the team group, sets up `~/.ssh` with correct permissions, forces a password change on first login, and logs the action. Write `offboard.sh <username>` implementing your full 8-step checklist, archiving the home directory before disabling anything.
*Deliverables:* both scripts, README, and your written offboarding policy.

**C5 — Service Watchdog**
Write a monitoring script that checks a list of services and URLs, and: restarts a service if it is down (with a maximum of 3 attempts), logs every action with timestamps, writes a status file that other tools can read, and never restarts more than once every 5 minutes. Deploy it as a **systemd service with a timer** (not cron) and prove it recovers a service you kill by hand.
*Deliverables:* the script, unit and timer files, README, and a demonstration log.

**C6 — Full Incident Simulation (final exam)**
Ask your teacher to run **three different lab breaks at once without telling you which**. You get 45 minutes to find and fix everything, then produce a full written incident report using the format in Part 13.
*Deliverables:* the incident report. This is the single most valuable document you will produce in this course — take it to interviews.

---

# PART 17 — AM I JOB READY? (self-assessment)

Tick only what you can do **without looking anything up**. Be honest — an interviewer will be.

### Level 1 — Comfortable
- [ ] Navigate anywhere, find any file, without a GUI
- [ ] Read and write files, and survive vim
- [ ] Read `ls -l` output fluently and convert permissions to numbers instantly
- [ ] Use `grep`, `head`, `tail`, `less`, `wc` without thinking
- [ ] Redirect output and errors correctly
- [ ] Explain what a pipe does with an example

### Level 2 — Useful on a team
- [ ] Build a `cut | sort | uniq -c | sort -rn | head` pipeline from scratch
- [ ] Use `awk` for columns, filters and sums
- [ ] Use `sed` for in-place edits, with backups
- [ ] Diagnose a full disk end to end, including the `df` vs `du` case
- [ ] Find and kill the right process, using the right signal
- [ ] Manage users, groups and sudo correctly
- [ ] Start, enable, diagnose and read logs for any systemd service
- [ ] Set up SSH key authentication with correct permissions

### Level 3 — Interview ready
- [ ] Write a script with `set -euo pipefail`, argument handling, logging and exit codes
- [ ] Schedule work in cron and debug why cron jobs fail
- [ ] Diagnose "the website is down" in a structured order and explain each step
- [ ] Explain DNS resolution order and prove it on a machine
- [ ] Write and deploy a systemd unit with auto-restart
- [ ] Produce a written incident report with a prevention section
- [ ] Answer any 10 random questions from Part 15 out loud, fluently, in under a minute each

**If every box in Level 3 is ticked, you are ready to interview for a Linux support, NOC, DevOps-trainee or junior sysadmin role.** If not, the unticked boxes are your revision list.

---

# APPENDIX A — LAB ENGINE (encoded)

Paste this into `~/labs/labs.b64`, then run:
```bash
base64 -d ~/labs/labs.b64 > ~/labs/labs.sh && chmod +x ~/labs/labs.sh
```
**Do not decode this and read it before finishing Part 13. You would only be cheating yourself.**

```
IyEvYmluL2Jhc2gKIyBOYW1tYSBTZXJ2ZXIgTGFiIEVuZ2luZSAtIERPIE5PVCBSRUFEIFRISVMgRklMRS4KIyBVc2FnZTogc3Vk
byBiYXNoIGxhYnMuc2ggYnJlYWsgPE4+ICAgfCAgIHN1ZG8gYmFzaCBsYWJzLnNoIHJlc3RvcmUgPE4+CnNldCAtdQpOPSIkezI6
LTB9IgpBQ1RJT049IiR7MTotaGVscH0iClM9L3Nydi9uYW1tYQoKY2FzZSAiJEFDVElPTjokTiIgaW4KaGVscDoqKSBlY2hvICJV
c2FnZTogc3VkbyBiYXNoIGxhYnMuc2ggYnJlYWsgPDEtMTI+ICAgfCAgIHN1ZG8gYmFzaCBsYWJzLnNoIHJlc3RvcmUgPDEtMTI+
IjsgZXhpdCAwOzsKCmJyZWFrOjEpICBzeXN0ZW1jdGwgc3RvcCBuZ2lueCAyPi9kZXYvbnVsbDsgc2VkIC1pICdzL2xpc3RlbiA4
MCBkZWZhdWx0X3NlcnZlcjsvbGlzdGVuIDgwODAgZGVmYXVsdF9zZXJ2ZXI7LycgL2V0Yy9uZ2lueC9zaXRlcy1lbmFibGVkL2Rl
ZmF1bHQgMj4vZGV2L251bGw7IHNlZCAtaSAncy9saXN0ZW4gXFs6OlxdOjgwIGRlZmF1bHRfc2VydmVyOy9saXN0ZW4gWzo6XTo4
MDgwIGRlZmF1bHRfc2VydmVyOy8nIC9ldGMvbmdpbngvc2l0ZXMtZW5hYmxlZC9kZWZhdWx0IDI+L2Rldi9udWxsOyBzeXN0ZW1j
dGwgc3RhcnQgbmdpbnggMj4vZGV2L251bGw7OwpyZXN0b3JlOjEpIHNlZCAtaSAncy9saXN0ZW4gODA4MCBkZWZhdWx0X3NlcnZl
cjsvbGlzdGVuIDgwIGRlZmF1bHRfc2VydmVyOy8nIC9ldGMvbmdpbngvc2l0ZXMtZW5hYmxlZC9kZWZhdWx0IDI+L2Rldi9udWxs
OyBzZWQgLWkgJ3MvbGlzdGVuIFxbOjpcXTo4MDgwIGRlZmF1bHRfc2VydmVyOy9saXN0ZW4gWzo6XTo4MCBkZWZhdWx0X3NlcnZl
cjsvJyAvZXRjL25naW54L3NpdGVzLWVuYWJsZWQvZGVmYXVsdCAyPi9kZXYvbnVsbDsgc3lzdGVtY3RsIHJlc3RhcnQgbmdpbngg
Mj4vZGV2L251bGw7OwoKYnJlYWs6MikgIGNwIC9ldGMvbmdpbngvbmdpbnguY29uZiAvcm9vdC9uZ2lueC5jb25mLmxhYi5iYWs7
IHNlZCAtaSAnMCwvd29ya2VyX3Byb2Nlc3NlcyBhdXRvOy9zLy93b3JrZXJfcHJvY2Vzc2VzIGF1dG8vJyAvZXRjL25naW54L25n
aW54LmNvbmY7IHN5c3RlbWN0bCByZXN0YXJ0IG5naW54IDI+L2Rldi9udWxsOzsKcmVzdG9yZToyKSBjcCAvcm9vdC9uZ2lueC5j
b25mLmxhYi5iYWsgL2V0Yy9uZ2lueC9uZ2lueC5jb25mIDI+L2Rldi9udWxsOyBzeXN0ZW1jdGwgcmVzdGFydCBuZ2lueCAyPi9k
ZXYvbnVsbDs7CgpicmVhazozKSAgY2htb2QgNjQ0ICRTL2JhY2t1cC9iYWNrdXAuc2ggMj4vZGV2L251bGw7IGNobW9kIDY0NCAk
Uy9hcHAvKi5zaCAyPi9kZXYvbnVsbDs7CnJlc3RvcmU6MykgY2htb2QgNzU1ICRTL2JhY2t1cC9iYWNrdXAuc2ggMj4vZGV2L251
bGw7OwoKYnJlYWs6NCkgIGNob3duIC1SIHJvb3Q6cm9vdCAkUy91cGxvYWRzIDI+L2Rldi9udWxsOyBjaG1vZCA3NTUgJFMvdXBs
b2Fkczs7CnJlc3RvcmU6NCkgY2hvd24gLVIgIiR7U1VET19VU0VSOi0kVVNFUn0iOiIke1NVRE9fVVNFUjotJFVTRVJ9IiAkUy91
cGxvYWRzIDI+L2Rldi9udWxsOzsKCmJyZWFrOjUpICBub2h1cCBiYXNoIC1jICd3aGlsZSB0cnVlOyBkbyBlY2hvICIkKGRhdGUp
IGxhYjUgZmlsbGVyIGxpbmUgZmlsbGVyIGxpbmUgZmlsbGVyIiA+PiAvc3J2L25hbW1hL2xvZ3MvcnVuYXdheS5sb2c7IGRvbmUn
ID4vZGV2L251bGwgMj4mMSAmIG5vaHVwIGJhc2ggLWMgJ3doaWxlIHRydWU7IGRvIDo7IGRvbmUnID4vZGV2L251bGwgMj4mMSAm
IGRpc293biAtYSAyPi9kZXYvbnVsbDsgZWNobyBzdGFydGVkOzsKcmVzdG9yZTo1KSBwa2lsbCAtZiAibGFiNSBmaWxsZXIiIDI+
L2Rldi9udWxsOyBwa2lsbCAtZiAid2hpbGUgdHJ1ZTsgZG8gOjsgZG9uZSIgMj4vZGV2L251bGw7IHJtIC1mICRTL2xvZ3MvcnVu
YXdheS5sb2c7OwoKYnJlYWs6NikgIHVzZXJhZGQgLW0gLXMgL2Jpbi9iYXNoIGxhYnVzZXIgMj4vZGV2L251bGw7IGVjaG8gImxh
YnVzZXI6TGFiQDEyMzQ1IiB8IGNocGFzc3dkOyB1c2VybW9kIC1zIC91c3Ivc2Jpbi9ub2xvZ2luIGxhYnVzZXI7IGNobW9kIDAw
MCAvaG9tZS9sYWJ1c2VyOzsKcmVzdG9yZTo2KSB1c2VyZGVsIC1yIGxhYnVzZXIgMj4vZGV2L251bGw7OwoKYnJlYWs6NykgIG1r
ZGlyIC1wIC9vcHQvbmFtbWF0b29sczsgcHJpbnRmICcjIS9iaW4vYmFzaFxuZWNobyAibmFtbWEtcmVwb3J0IE9LIlxuJyA+IC9v
cHQvbmFtbWF0b29scy9uYW1tYS1yZXBvcnQ7IGNobW9kIDc1NSAvb3B0L25hbW1hdG9vbHMvbmFtbWEtcmVwb3J0OyBybSAtZiAv
dXNyL2xvY2FsL2Jpbi9uYW1tYS1yZXBvcnQ7OwpyZXN0b3JlOjcpIHJtIC1yZiAvb3B0L25hbW1hdG9vbHM7OwoKYnJlYWs6OCkg
IGhlYWQgLWMgMjUwMDAwMDAwIC9kZXYvdXJhbmRvbSA+IC90bXAvbGFiOC5kYXQ7IG5vaHVwIHB5dGhvbjMgLWMgImY9b3Blbign
L3RtcC9sYWI4LmRhdCcpO2ltcG9ydCB0aW1lO3RpbWUuc2xlZXAoNDAwMCkiID4vZGV2L251bGwgMj4mMSAmIHNsZWVwIDI7IHJt
IC1mIC90bXAvbGFiOC5kYXQ7IGVjaG8gc3RhcnRlZDs7CnJlc3RvcmU6OCkgcGtpbGwgLWYgImxhYjguZGF0IiAyPi9kZXYvbnVs
bDsgcm0gLWYgL3RtcC9sYWI4LmRhdDs7CgpicmVhazo5KSAgVT0iJHtTVURPX1VTRVI6LSRVU0VSfSI7IG1rZGlyIC1wIC9vcHQv
bmFtbWF0b29sczsgcHJpbnRmICcjIS9iaW4vYmFzaFxubmFtbWEtcmVwb3J0ID4+IC9zcnYvbmFtbWEvbG9ncy9jcm9uLXRhc2su
bG9nIDI+JjFcbicgPiAvaG9tZS8kVS9sYWI5LXRhc2suc2g7IGNobW9kIDc1NSAvaG9tZS8kVS9sYWI5LXRhc2suc2g7IGNob3du
ICRVIC9ob21lLyRVL2xhYjktdGFzay5zaDsgKGNyb250YWIgLXUgJFUgLWwgMj4vZGV2L251bGw7IGVjaG8gIiogKiAqICogKiAv
aG9tZS8kVS9sYWI5LXRhc2suc2giKSB8IGNyb250YWIgLXUgJFUgLTsgc2VydmljZSBjcm9uIHN0YXJ0IDI+L2Rldi9udWxsOzsK
cmVzdG9yZTo5KSBVPSIke1NVRE9fVVNFUjotJFVTRVJ9IjsgY3JvbnRhYiAtdSAkVSAtbCAyPi9kZXYvbnVsbCB8IGdyZXAgLXYg
bGFiOS10YXNrIHwgY3JvbnRhYiAtdSAkVSAtIDsgcm0gLWYgL2hvbWUvJFUvbGFiOS10YXNrLnNoOzsKCmJyZWFrOjEwKSBlY2hv
ICIyMDMuMC4xMTMuNzcgbmFtbWEtYXBpLmxvY2FsIiA+PiAvZXRjL2hvc3RzOzsKcmVzdG9yZToxMCkgc2VkIC1pICcvbmFtbWEt
YXBpLmxvY2FsL2QnIC9ldGMvaG9zdHM7OwoKYnJlYWs6MTEpIHN5c3RlbWN0bCBkaXNhYmxlIG5naW54IDI+L2Rldi9udWxsOyBz
eXN0ZW1jdGwgc3RhcnQgbmdpbnggMj4vZGV2L251bGw7OwpyZXN0b3JlOjExKSBzeXN0ZW1jdGwgZW5hYmxlIG5naW54IDI+L2Rl
di9udWxsOzsKCmJyZWFrOjEyKSBta2RpciAtcCAkUy9zcG9vbDsgZm9yIGkgaW4gJChzZXEgMSAzMDAwKTsgZG8gOiA+ICRTL3Nw
b29sL21zZ18kaS50bXA7IGRvbmU7IGhlYWQgLWMgMTgwMDAwMDAwIC9kZXYvdXJhbmRvbSB8IGJhc2U2NCA+ICRTL2xvZ3MvYXVk
aXQubG9nOyBjaG93biAtUiAiJHtTVURPX1VTRVI6LSRVU0VSfSIgJFMvc3Bvb2wgJFMvbG9ncy9hdWRpdC5sb2c7OwpyZXN0b3Jl
OjEyKSBybSAtcmYgJFMvc3Bvb2wgJFMvbG9ncy9hdWRpdC5sb2c7OwoKKikgZWNobyAiVW5rbm93bjogJEFDVElPTiAkTiI7IGV4
aXQgMTs7CmVzYWMKZWNobyAiZG9uZTogJEFDVElPTiAkTiIK
```

---

# APPENDIX B — THE COMMAND LIST YOU MUST OWN

If you cannot explain what one of these does and give an example, it goes on your revision list.

`pwd ls cd mkdir rmdir rm cp mv touch ln find locate file stat tree basename dirname`
`cat less more head tail wc sort uniq cut tr grep egrep zgrep sed awk diff cmp md5sum sha256sum tee xargs column`
`chmod chown chgrp umask id whoami groups useradd usermod userdel passwd chage groupadd getent su sudo visudo`
`ps top htop kill pkill killall pgrep jobs fg bg nohup disown nice renice free vmstat uptime lsof watch tmux`
`df du mount umount lsblk truncate dd mkfs logrotate`
`systemctl journalctl service dmesg`
`ip ifconfig ss netstat ping traceroute dig nslookup host curl wget ssh ssh-keygen ssh-copy-id scp rsync ufw`
`apt dpkg tar gzip gunzip zip unzip`
`echo printf read export env set unset alias unalias history which type man tldr crontab at date sleep`

---

*Workbook prepared for Happy Minds AI Learning Center · Linux Practical Track · 385 numbered problems, 12 incident drills, 60 speed drills, 100 verbal questions, 6 capstone projects.*
