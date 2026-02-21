
# FILE: log_analyzer.sh


#!/bin/bash
set -euo pipefail

file="$1"

# ---------------- VALIDATION ----------------

if [[ $# -eq 0 ]]; then
  echo "Usage: $0 <logfile>"
  exit 1
fi

if [[ ! -f "$file" ]]; then
  echo "Log file not found"
  exit 1
fi

# ---------------- FUNCTIONS ----------------

error_count(){
  grep -i "error\|failed" "$file" | wc -l || true
}

critical(){
  CRITICAL_EVENTS=$(grep -in "critical" "$file" || true)

  if [[ -z "$CRITICAL_EVENTS" ]]; then
    echo "No critical events found"
  else
    echo "$CRITICAL_EVENTS"
  fi
}

top_5_errors(){
  grep -i "error" "$file" \
  | awk '{ $1=$2=$3=$4=$5=$6=""; print }' \
  | sort | uniq -c | sort -rn | head -5 || true
}

summary_report(){

TIMESTAMP=$(date +"%Y-%m-%d_%H-%M-%S")
REPORT="log_report_of_$(basename "$file")_$TIMESTAMP.txt"

TOTAL_LINES=$(wc -l < "$file")
ERRORS=$(error_count)

{
echo "===== LOG ANALYSIS REPORT ====="
echo
echo "Date of Analysis: $(date)"
echo "Log file: $(basename "$file")"
echo "Total Lines Processed: $TOTAL_LINES"
echo "TOTAL ERROR COUNT : $ERRORS"
echo
echo "----- Critical Events -----"
critical
echo
echo "##### TOP 5 MOST COMMON ERROR MESSAGES #####"
top_5_errors
} > "$REPORT"

echo "LOG SUMMARY REPORT CREATED : $REPORT"
}

archive_logs(){
mkdir -p archive
mv "$file" archive/
echo "Log file archived to archive/"
}

# ---------------- EXECUTION ----------------

summary_report
archive_logs



# Day 20 – Log Analyzer Script

## Objective
Build a Bash script that analyzes log files and generates automated summary reports.

---

## Features Implemented

### Task 1 – Input Validation
- Accepts logfile as argument
- Exits if missing
- Exits if file not found

---

### Task 2 – Error Count
Counts lines containing:
- ERROR
- Failed

Command used:

grep -i "error\|failed"

---

### Task 3 – Critical Events
Displays CRITICAL lines with line numbers:

grep -in "critical"

---

### Task 4 – Top 5 Error Messages
Pipeline:

grep → awk → sort → uniq → sort → head

Extracts message text and shows most frequent errors.

---

### Task 5 – Summary Report
Creates timestamped file:

log_report_of_<logfile>_<datetime>.txt

Includes:

- Date
- Log filename
- Total lines
- Error count
- Critical events
- Top 5 errors

---

### Task 6 – Archiving
Processed logs moved to:

archive/

Directory auto-created.

---

## How To Run

chmod +x log_analyzer.sh

./log_analyzer.sh logfile

---

## What I Learned

- grep filtering
- awk field removal
- wc input redirection
- pipeline chaining
- set -euo pipefail
- defensive scripting
- log automation
- real-world Bash workflows

---


