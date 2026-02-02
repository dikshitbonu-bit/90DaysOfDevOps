
# Day 10 Challenge – File Permissions & Operations

## Files Created
- devops.txt
- notes.txt
- script.sh
- project/

## Permission Changes

### script.sh
- Before: rw-r--r--
- After: rwxr-xr-x
- Executable and runnable

### devops.txt
- Before: rw-r--r--
- After: r--r--r--
- Read-only for all users

### notes.txt
- Set to 640
- Owner: read/write
- Group: read
- Others: no access

### project/
- Permissions: 755
- Owner full access, others read & execute

## Commands Used
- touch
- echo >
- vim
- cat
- head
- tail
- chmod
- ls -l
- mkdir

## What I Learned
1. Linux permissions control access using owner, group, and others.
2. Execute permission is required to run scripts.
3. Incorrect permissions result in permission denied errors.
