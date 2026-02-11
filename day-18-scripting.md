Today I learned how to write cleaner and reusable bash scripts using functions, strict mode, and real-world scripting patterns.

--------------------------------------------------

## Task 1 – Basic Functions

Goal: Understand how functions work and how to pass arguments.

### Script: functions.sh

#!/bin/bash

greet() {
  echo "Hello, $1!"
}

add() {
  echo "Sum: $(($1 + $2))"
}

greet "DevOps"
add 5 10

Explanation:

- greet accepts a name using $1
- add accepts two numbers using $1 and $2
- Functions are called just like commands
- This teaches parameter passing and reusability.

--------------------------------------------------

## Task 2 – Functions with Practical Commands

Goal: Use functions for system checks.

### Script: disk_check.sh

#!/bin/bash

check_disk() {
  df -h /
}

check_memory() {
  free -h
}

main() {
  echo "===== Disk Usage ====="
  check_disk

  echo

  echo "===== Memory Usage ====="
  check_memory
}

main

Explanation:

- check_disk checks root disk usage
- check_memory shows RAM
- main() controls execution flow
- This mimics real production scripts.

--------------------------------------------------

## Task 3 – Strict Mode (Safety First)

Goal: Prevent silent failures.

### Script: strict_demo.sh

#!/bin/bash
set -euo pipefail

echo $UNDEFINED_VAR
false
ls /fake | grep hello

Explanation:

set -e → Script exits if any command fails  
set -u → Script exits if undefined variable is used  
set -o pipefail → Pipeline fails if ANY command fails  

Strict mode protects production scripts from hidden bugs.

--------------------------------------------------

## Task 4 – Local Variables

Goal: Learn variable scope.

### Script: local_demo.sh

#!/bin/bash

local_func() {
  local VAR="inside"
  echo "Inside VAR: $VAR"
}

global_func() {
  VAR2="outside"
}

local_func
echo "Outside VAR: ${VAR:-not set}"

global_func
echo "Outside VAR2: $VAR2"

Explanation:

- local keeps variables inside functions
- Normal variables leak globally
- This prevents accidental overwrites.

--------------------------------------------------

## Task 5 – System Info Reporter (Real DevOps Script)

Goal: Combine everything into one professional script.

### Script: system_info.sh

#!/bin/bash
set -euo pipefail

host_info() {
  echo "===== Host Info ====="
  hostname
  grep PRETTY_NAME /etc/os-release
}

uptime_info() {
  echo "===== Uptime ====="
  uptime -p
}

disk_info() {
  echo "===== Top 5 Disk Usage ====="
  du -xhd1 / 2>/dev/null | sort -hr | head -5
}

memory_info() {
  echo "===== Memory Usage ====="
  free -h | awk 'NR==2 {print}'
}

cpu_info() {
  echo "===== Top 5 CPU Processes ====="
  ps -eo pid,comm,%cpu --sort=-%cpu | head -6
}

main() {
  host_info
  echo
  uptime_info
  echo
  disk_info
  echo
  memory_info
  echo
  cpu_info
}

main

Explanation:

Each system metric is inside its own function.
main() controls execution.
Strict mode ensures safety.
This mirrors real monitoring scripts.

--------------------------------------------------

## What I Learned

1. Functions make scripts modular and readable.
2. Strict mode avoids dangerous silent failures.
3. local variables prevent scope leaks.

Day 18 completed successfully.
EOF


cat <<'EOF' > functions.sh
#!/bin/bash
greet(){ echo "Hello, $1!"; }
add(){ echo "Sum: $(($1+$2))"; }
greet "DevOps"
add 5 10
EOF

cat <<'EOF' > disk_check.sh
#!/bin/bash
check_disk(){ df -h /; }
check_memory(){ free -h; }
main(){ check_disk; echo; check_memory; }
main
EOF

cat <<'EOF' > strict_demo.sh
#!/bin/bash
set -euo pipefail
echo $UNDEFINED_VAR
false
ls /fake | grep hi
EOF

cat <<'EOF' > local_demo.sh
#!/bin/bash
local_func(){ local A=10; echo $A; }
local_func
echo ${A:-not_set}
EOF

cat <<'EOF' > system_info.sh
#!/bin/bash
set -euo pipefail
du -xhd1 / | sort -hr | head -5
EOF

chmod +x *.sh

echo "Day 18 ready."

