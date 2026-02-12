# ============================================================
# DAY 19 — TASK 1: LOG ROTATION SCRIPT
# ============================================================

# Goal:
# 1. Take a log directory as input
# 2. Compress .log files older than 7 days
# 3. Delete .gz files older than 30 days
# 4. Count how many files were compressed + deleted
# 5. Exit if directory does not exist

# Example run:
# ./log_rotate.sh /var/log/myapp


#!/bin/bash
set -euo pipefail

LOG_DIR="$1"

# ------------------------------------------------------------
# Step 1 — Validate input
# ------------------------------------------------------------

# If directory is missing or does not exist → exit

if [ -z "$LOG_DIR" ] || [ ! -d "$LOG_DIR" ]; then
  echo "Usage: $0 <log_directory>"
  exit 1
fi


# ------------------------------------------------------------
# Step 2 — Compress .log files older than 7 days
# ------------------------------------------------------------

# find → locate files
# -name "*.log" → select .log files
# -mtime +7 → older than 7 days
# -exec gzip {} \; → gzip each file
# -print → print filename after compression
# wc -l → count printed lines

compressed=$(find "$LOG_DIR" -name "*.log" -mtime +7 -exec gzip {} \; -print | wc -l)


# ------------------------------------------------------------
# Step 3 — Delete .gz files older than 30 days
# ------------------------------------------------------------

# find old .gz files
# print filenames
# delete them
# count deleted files

deleted=$(find "$LOG_DIR" -name "*.gz" -mtime +30 -print -delete | wc -l)


# ------------------------------------------------------------
# Step 4 — Print results
# ------------------------------------------------------------

echo "Compressed files: $compressed"
echo "Deleted files: $deleted"


# ============================================================
# KEY CONCEPTS
# ============================================================

# {}      → current file found by find
# \;      → end of -exec command
# -exec   → run command on each found file
# -print  → output filename
# wc -l   → count lines
# $( )    → capture command output
# set -e  → exit on error
# set -u  → exit on undefined variable
# set -o pipefail → fail if any piped command fails

# find ... -exec gzip {} \; -print | wc -l
# compresses files and counts them in one pipeline

# Unix philosophy:
# small tools chained together


# ============================================================
# OUTCOME
# ============================================================

# Real log rotation implemented:
# • Old logs compressed
# • Very old archives deleted
# • Counts displayed
# • Errors handled safely


# ============================================================
# DAY 19 — TASK 2: SERVER BACKUP SCRIPT
# ============================================================

# Goal:
# 1. Take source directory and backup destination as arguments
# 2. Create timestamped .tar.gz archive
# 3. Verify archive creation
# 4. Print archive name and size
# 5. Delete backups older than 14 days
# 6. Exit if source directory does not exist

# Example run:
# ./backup.sh /data /backups


#!/bin/bash
set -euo pipefail

SOURCE="$1"
DEST="$2"

# ------------------------------------------------------------
# Step 1 — Validate inputs
# ------------------------------------------------------------

if [ -z "$SOURCE" ] || [ -z "$DEST" ]; then
  echo "Usage: $0 <source_directory> <backup_destination>"
  exit 1
fi

if [ ! -d "$SOURCE" ]; then
  echo "Source directory does not exist"
  exit 1
fi

mkdir -p "$DEST"


# ------------------------------------------------------------
# Step 2 — Create timestamped archive
# ------------------------------------------------------------

TIMESTAMP=$(date +%Y-%m-%d)
ARCHIVE="backup-$TIMESTAMP.tar.gz"

tar -czf "$DEST/$ARCHIVE" "$SOURCE"


# ------------------------------------------------------------
# Step 3 — Verify archive creation
# ------------------------------------------------------------

if [ ! -f "$DEST/$ARCHIVE" ]; then
  echo "Backup failed"
  exit 1
fi


# ------------------------------------------------------------
# Step 4 — Print archive name and size
# ------------------------------------------------------------

SIZE=$(du -h "$DEST/$ARCHIVE" | cut -f1)

echo "Backup created: $ARCHIVE"
echo "Size: $SIZE"


# ------------------------------------------------------------
# Step 5 — Delete backups older than 14 days
# ------------------------------------------------------------

deleted=$(find "$DEST" -name "backup-*.tar.gz" -mtime +14 -print -delete | wc -l)

echo "Old backups deleted: $deleted"


# ============================================================
# KEY CONCEPTS
# ============================================================

# tar -czf
# c → create archive
# z → gzip compression
# f → output filename

# date +%Y-%m-%d
# generates timestamp for filename

# du -h | cut -f1
# extracts file size

# find ... -mtime +14 -delete
# removes backups older than 14 days

# set -euo pipefail
# stops script on any failure


# ============================================================
# OUTCOME
# ============================================================

# Automated server backups:
# • Timestamped archives
# • Verification
# • Size reporting
# • Cleanup of old backups
# • Safe error handling

# ============================================================
# DAY 19 — TASK 3: CRONTAB
# ============================================================

# View current cron jobs
crontab -l


# ------------------------------------------------------------
# Cron Syntax
# ------------------------------------------------------------

# * * * * * command
# │ │ │ │ │
# │ │ │ │ └── Day of week (0–7) (Sunday = 0 or 7)
# │ │ │ └──── Month (1–12)
# │ │ └────── Day of month (1–31)
# │ └──────── Hour (0–23)
# └────────── Minute (0–59)


# ------------------------------------------------------------
# Cron Entries (written in markdown only — not applied)
# ------------------------------------------------------------

# Run log rotation every day at 2 AM
0 2 * * * /root/log_rotate.sh /var/log/myapp

# Run backup every Sunday at 3 AM
0 3 * * 0 /root/backup.sh /data /backups

# Run health check every 5 minutes
*/5 * * * * /root/health_check.sh


# ------------------------------------------------------------
# Explanation
# ------------------------------------------------------------

# 0 2 * * *
# → At minute 0, hour 2, every day

# 0 3 * * 0
# → At 3 AM every Sunday

# */5 * * * *
# → Every 5 minutes


# ------------------------------------------------------------
# Important Notes
# ------------------------------------------------------------

# Always use FULL PATHS in cron:
# /root/script.sh instead of ./script.sh

# Cron has limited environment variables
# Always specify absolute paths for commands and files

# Redirect output for debugging:
# 0 2 * * * /root/log_rotate.sh /var/log/myapp >> /var/log/cron.log 2>&1

# This captures both stdout and errors


# ============================================================
# OUTCOME
# ============================================================

# Scripts are scheduled automatically:
# • Log rotation runs daily
# • Backups run weekly
# • Health checks run every 5 minutes

# ============================================================
# DAY 19 — TASK 4: SCHEDULED MAINTENANCE SCRIPT
# ============================================================

# Goal:
# 1. Run log rotation
# 2. Run backup
# 3. Add timestamps to every message
# 4. Log everything to /var/log/maintenance.log
# 5. Prepare cron entry to run daily at 1 AM

# Example cron:
# 0 1 * * * /root/maintenance.sh


#!/bin/bash
set -euo pipefail

LOGFILE="/var/log/maintenance.log"

LOG_DIR="/var/log/myapp"
SOURCE="/data"
DEST="/backups"


# ------------------------------------------------------------
# Logging function
# ------------------------------------------------------------

log() {
  echo "$(date '+%Y-%m-%d %H:%M:%S') : $1" >> "$LOGFILE"
}


# ------------------------------------------------------------
# Log rotation function
# ------------------------------------------------------------

log_rotate() {

  log "Starting log rotation"

  compressed=$(find "$LOG_DIR" -name "*.log" -mtime +7 -exec gzip {} \; -print | wc -l)

  deleted=$(find "$LOG_DIR" -name "*.gz" -mtime +30 -print -delete | wc -l)

  log "Logs compressed: $compressed"
  log "Old archives deleted: $deleted"
}


# ------------------------------------------------------------
# Backup function
# ------------------------------------------------------------

backup() {

  log "Starting backup"

  TIMESTAMP=$(date +%Y-%m-%d)
  ARCHIVE="backup-$TIMESTAMP.tar.gz"

  tar -czf "$DEST/$ARCHIVE" "$SOURCE"

  if [ ! -f "$DEST/$ARCHIVE" ]; then
    log "Backup failed"
    exit 1
  fi

  SIZE=$(du -h "$DEST/$ARCHIVE" | cut -f1)

  log "Backup created: $ARCHIVE"
  log "Backup size: $SIZE"

  removed=$(find "$DEST" -name "backup-*.tar.gz" -mtime +14 -print -delete | wc -l)

  log "Old backups removed: $removed"
}


# ------------------------------------------------------------
# Main function
# ------------------------------------------------------------

main() {

  log "===== Maintenance started ====="

  log_rotate
  backup

  log "===== Maintenance finished ====="
}


# ------------------------------------------------------------
# Run main
# ------------------------------------------------------------

main


# ============================================================
# CRON ENTRY (write in markdown)
# ============================================================

# Run daily at 1 AM:
# 0 1 * * * /root/maintenance.sh


# ============================================================
# OUTCOME
# ============================================================

# • Log rotation automated
# • Backups automated
# • Timestamped logging
# • Old files cleaned
# • Safe execution using set -euo pipefail
