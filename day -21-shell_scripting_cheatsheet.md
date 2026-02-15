# Bash Scripting Cheat Sheet

A comprehensive guide to Bash scripting for DevOps engineers, covering everything from basics to advanced patterns.

---

## Quick Reference Table

| Topic | Key Syntax | Example |
|-------|------------|---------|
| Variable | `VAR="value"` | `NAME="DevOps"` |
| Print Variable | `echo $VAR` | `echo $NAME` |
| Argument | `$1, $2` | `./script.sh arg1` |
| Argument Count | `$#` | `echo $#` |
| All Arguments | `$@` | `echo "$@"` |
| Exit Code | `$?` | `echo $?` |
| If Statement | `if [ condition ]; then` | `if [ -f file ]; then` |
| Else | `else` | `else echo "missing"` |
| For Loop | `for i in list; do` | `for i in 1 2 3; do` |
| While Loop | `while condition; do` | `while read line; do` |
| Function | `name() { }` | `greet() { echo Hi; }` |
| Function Arg | `$1` | `greet "DevOps"` |
| Local Var | `local VAR` | `local x=10` |
| Grep | `grep pattern file` | `grep -i "error" log.txt` |
| Grep Count | `grep -c` | `grep -c "error" log.txt` |
| Awk | `awk '{print $1}'` | `awk -F: '{print $1}' /etc/passwd` |
| Sed Replace | `sed 's/a/b/'` | `sed -i 's/foo/bar/g' file` |
| Cut | `cut -d: -f1` | `cut -d: -f1 /etc/passwd` |
| Sort | `sort` | `sort -n numbers.txt` |
| Uniq | `uniq -c` | `uniq -c file` |
| Word Count | `wc -l` | `wc -l file` |
| Head | `head -5` | `head -5 file` |
| Tail | `tail -f` | `tail -f app.log` |
| Logical AND | `&&` | `cmd1 && cmd2` |
| Logical OR | `||` | `cmd1 || cmd2` |
| Strict Mode | `set -euo pipefail` | `set -euo pipefail` |
| Debug Mode | `set -x` | `set -x` |
| Trap | `trap cleanup EXIT` | `trap cleanup EXIT` |

---

## Task 1: Basics

### Shebang

Tells the system which shell interpreter to use.

```bash
#!/bin/bash
```

Without this, the script may run using the wrong shell.

### Running a Script

**Make executable:**

```bash
chmod +x script.sh
```

**Run directly:**

```bash
./script.sh
```

**Run using bash:**

```bash
bash script.sh
```

### Comments

**Single-line comment:**

```bash
# This is a comment
```

**Inline comment:**

```bash
echo "Hello" # inline comment
```

Used for documentation and readability.

### Variables

**Declare:**

```bash
NAME="DevOps"
```

**Use:**

```bash
echo $NAME
```

**Quoting:**

```bash
"$NAME"   # expands variable (recommended)
'$NAME'   # literal text
```

### Reading User Input

```bash
read username
echo "Hello $username"
```

**With prompt:**

```bash
read -p "Enter your name: " username
```

### Command-Line Arguments

```bash
$0  # script name
$1  # first argument
$2  # second argument
$#  # total arguments
$@  # all arguments
$?  # last command exit status
```

**Example:**

```bash
echo "First arg: $1"
echo "Total args: $#"
```

---

## Task 2: Operators and Conditionals

### String Comparisons

```bash
[ "$a" = "$b" ]   # equal
[ "$a" != "$b" ]  # not equal
[ -z "$a" ]       # string is empty
[ -n "$a" ]       # string is not empty
```

**Example:**

```bash
if [ "$name" = "admin" ]; then
    echo "Welcome admin"
fi
```

### Integer Comparisons

```bash
[ $a -eq $b ]  # equal
[ $a -ne $b ]  # not equal
[ $a -lt $b ]  # less than
[ $a -gt $b ]  # greater than
[ $a -le $b ]  # less or equal
[ $a -ge $b ]  # greater or equal
```

### File Test Operators

```bash
-f file   # regular file
-d dir    # directory
-e path   # exists
-r file   # readable
-w file   # writable
-x file   # executable
-s file   # not empty
```

### if / elif / else Syntax

```bash
if [ condition ]; then
    command
elif [ condition ]; then
    command
else
    command
fi
```

### Logical Operators

```bash
cmd1 && cmd2   # run cmd2 if cmd1 succeeds
cmd1 || cmd2   # run cmd2 if cmd1 fails
! condition    # NOT
```

### Case Statement

```bash
case $choice in
    start)
        echo "Starting"
        ;;
    stop)
        echo "Stopping"
        ;;
    *)
        echo "Unknown option"
        ;;
esac
```

---

## Task 3: Loops

### For Loop (List Based)

```bash
for i in 1 2 3; do
    echo "$i"
done
```

### For Loop (C Style)

```bash
for ((i=1; i<=5; i++)); do
    echo "$i"
done
```

### While Loop

Runs while condition is true.

```bash
count=1

while [ $count -le 3 ]; do
    echo "$count"
    ((count++))
done
```

### Until Loop

Runs until condition becomes true.

```bash
num=1

until [ $num -gt 3 ]; do
    echo "$num"
    ((num++))
done
```

### Loop Control

**break** — exit loop

```bash
for i in 1 2 3 4; do
    [ $i -eq 3 ] && break
    echo "$i"
done
```

**continue** — skip iteration

```bash
for i in 1 2 3; do
    [ $i -eq 2 ] && continue
    echo "$i"
done
```

### Loop Over Files

```bash
for file in *.log; do
    echo "$file"
done
```

### Loop Over Command Output

```bash
cat file.txt | while read line; do
    echo "$line"
done
```

**Or safer:**

```bash
while read line; do
    echo "$line"
done < file.txt
```

---

## Task 4: Functions

### Defining a Function

```bash
greet() {
    echo "Hello DevOps"
}
```

### Calling a Function

```bash
greet
```

Functions run just by writing their name.

### Passing Arguments to Functions

Inside functions:
- `$1` = first argument
- `$2` = second argument

```bash
welcome() {
    echo "Welcome $1"
}

welcome Dikshith
```

**Output:**

```
Welcome Dikshith
```

### Return Values

Bash functions don't return strings — only numbers (0–255).

**Using return (status code):**

```bash
check() {
    return 0
}

check
echo $?
```

**Using echo to send data:**

```bash
add() {
    echo $(($1 + $2))
}

result=$(add 5 3)
echo "$result"
```

### Local Variables

Without `local`, variables leak outside function.

```bash
demo() {
    local x=10
    echo "$x"
}

demo
```

`x` exists ONLY inside function.

### Example Real Function

```bash
disk_usage() {
    echo "Top Disk Usage:"
    du -sh * | sort -hr | head -5
}

disk_usage
```

Functions help organize scripts, avoid repetition, and make code readable.

---

## Task 5: Text Processing Commands

These commands help you SEARCH, FILTER, MODIFY, and ANALYZE text/log files.

### GREP — Search Text

**Search for pattern:**

```bash
grep "error" logfile.txt
```

**Case-insensitive:**

```bash
grep -i "error" logfile.txt
```

**Show line numbers:**

```bash
grep -n "error" logfile.txt
```

**Count matches:**

```bash
grep -c "error" logfile.txt
```

**Invert match (NOT):**

```bash
grep -v "info" logfile.txt
```

**Recursive:**

```bash
grep -r "error" /var/log
```

**Extended regex:**

```bash
grep -E "error|failed" logfile.txt
```

### AWK — Column Processing

**Print first column:**

```bash
awk '{print $1}' file.txt
```

**Custom delimiter:**

```bash
awk -F: '{print $1}' /etc/passwd
```

**Pattern match:**

```bash
awk '/error/ {print $0}' logfile.txt
```

**BEGIN / END:**

```bash
awk 'BEGIN{print "Start"} {print $1} END{print "End"}' file.txt
```

### SED — Stream Editor

**Replace text:**

```bash
sed 's/old/new/' file.txt
```

**Global replace:**

```bash
sed 's/old/new/g' file.txt
```

**In-place edit:**

```bash
sed -i 's/foo/bar/g' config.txt
```

**Delete lines:**

```bash
sed '/error/d' logfile.txt
```

### CUT — Extract Columns

**By delimiter:**

```bash
cut -d: -f1 /etc/passwd
```

**Character range:**

```bash
cut -c1-5 file.txt
```

### SORT

**Alphabetical:**

```bash
sort file.txt
```

**Numerical:**

```bash
sort -n numbers.txt
```

**Reverse:**

```bash
sort -r file.txt
```

**Unique:**

```bash
sort file.txt | uniq
```

### UNIQ

**Remove duplicates:**

```bash
uniq file.txt
```

**Count duplicates:**

```bash
sort file.txt | uniq -c
```

### TR — Translate/Delete Characters

**Lower → Upper:**

```bash
tr a-z A-Z
```

**Delete spaces:**

```bash
tr -d ' '
```

### WC — Count

**Lines:**

```bash
wc -l file.txt
```

**Words:**

```bash
wc -w file.txt
```

**Characters:**

```bash
wc -c file.txt
```

### HEAD / TAIL

**First 10 lines:**

```bash
head file.txt
```

**Last 10 lines:**

```bash
tail file.txt
```

**Follow logs live:**

```bash
tail -f logfile.txt
```

---

## Task 6: Useful Patterns and One-Liners

### Find and Delete Files Older Than N Days

**Delete files older than 7 days:**

```bash
find /path -type f -mtime +7 -delete
```

**Dry run (see first):**

```bash
find /path -type f -mtime +7
```

### Count Lines in All .log Files

```bash
wc -l *.log
```

**Total lines across all logs:**

```bash
cat *.log | wc -l
```

### Replace a String Across Multiple Files

**Replace foo with bar:**

```bash
sed -i 's/foo/bar/g' *.txt
```

**Recursive:**

```bash
sed -i 's/foo/bar/g' $(grep -rl foo .)
```

### Check if a Service is Running

**Systemd:**

```bash
systemctl is-active nginx
```

**Old school:**

```bash
ps aux | grep nginx
```

### Disk Usage Above Threshold

```bash
df -h | awk '$5 > "80%" {print}'
```

Shows partitions above 80%.

### Parse CSV from Command Line

**Print first column:**

```bash
cut -d, -f1 file.csv
```

### Tail Logs and Filter Errors in Real Time

```bash
tail -f app.log | grep -i error
```

### Find Biggest Files

```bash
du -ah / | sort -hr | head -10
```

### Count ERRORs in Logs

```bash
grep -ci error logfile.txt
```

### Check Open Ports

```bash
ss -tuln
```

---

## Task 7: Error Handling and Debugging

This section teaches you how to make scripts SAFE, predictable, and debuggable.

**Production scripts MUST fail loudly and early.**

### Exit Codes

Every Linux command returns an exit code.

- `0` → success
- `non-zero` → failure

**Check last command status:**

```bash
echo $?
```

**Manually exit:**

```bash
exit 0   # success
exit 1   # failure
```

**Example:**

```bash
cp file1 file2 || exit 1
```

### set -e (Exit on Error)

Script stops immediately when any command fails.

```bash
set -e
```

**Example:**

```bash
set -e
ls goodfile
ls badfile   # script stops here
echo "Never runs"
```

**Why useful:** Prevents silent failures.

### set -u (Undefined Variables)

Script crashes if you use undefined variables.

```bash
set -u
```

**Example:**

```bash
set -u
echo $NOT_DEFINED   # crashes
```

**Why useful:** Avoids typo bugs and empty variables.

### set -o pipefail (Pipeline Safety)

By default pipelines hide failures.

```bash
false | true
echo $?   # returns 0 (BAD)
```

**Fix:**

```bash
set -o pipefail
false | true
echo $?   # returns 1 (GOOD)
```

Now ANY failure inside pipe breaks script.

### Full Strict Mode (Always Use in Real Scripts)

```bash
set -euo pipefail
```

This enables:
- ✔ stop on error
- ✔ stop on unset variables
- ✔ detect pipe failures

This is professional-grade scripting.

### set -x (Debug Mode)

Prints every command before execution.

```bash
set -x
```

**Turn off:**

```bash
set +x
```

**Example:**

```bash
set -x
echo "hello"
ls
```

Used when debugging broken scripts.

### Trap (Cleanup on Exit)

Run code when script exits or crashes.

**Example:**

```bash
cleanup() {
    echo "Cleaning up temp files"
}

trap cleanup EXIT
```

Now `cleanup` runs EVEN if script fails.

**Used for:**
- ✔ deleting temp files
- ✔ closing resources
- ✔ logging exits

### Real Example

```bash
#!/bin/bash
set -euo pipefail

cleanup() {
    echo "Script finished"
}

trap cleanup EXIT

FILE="demo.txt"

cat "$FILE"
```

If `demo.txt` missing → script exits safely.

### Summary

- `set -e` → stop on errors
- `set -u` → stop on undefined vars
- `set -o pipefail` → detect pipeline failures
- `set -x` → debug mode
- `trap` → cleanup on exit

---






