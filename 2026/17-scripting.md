
# Day 17 – Bash Scripting (Loops, Arguments & Error Handling)

## Task 1 – For Loop
 for_loop.sh


  #!/bin/bash

  fruits=("apple" "banana" "mango" "orange" "grape")

  for fruit in "${fruits[@]}"; do
        echo "$fruit"
  done
Output:

    apple
    banana
    mango
    orange
    grape

count.sh
    #!/bin/bash

    for i in {1..10}; do
       echo $i
    done
Output:

    1
    2
    3
    4
    5
    6
    7
    8
    9
    10

Task 2 – While Loop
countdown.sh
    #!/bin/bash

    read -p "Enter a number: " num

    while [ $num -ge 0 ]; do
     echo $num
     ((num--))
    done

    echo "Done!"
Sample Output:

    3
    2
    1
    0
    Done!

Task 3 – Command Line Arguments
greet.sh
    #!/bin/bash

    if [[ $# -eq 0 ]]; then
     echo "Usage: ./greet.sh <name>"
    else
     echo "Hello, $1!"
    fi
Output:

    Hello dk!

args_demo.sh
    #!/bin/bash

    echo "Script name: $0"
    echo "Total arguments: $#"
    echo "All arguments: $@"
Sample Output:

    Script name: ./args_demo.sh
    Total arguments: 2
    All arguments: dk devops

Task 4 – Install Packages via Script
install_packages.sh
    #!/bin/bash

    if [[ $EUID -ne 0 ]]; then
     echo "Run as root"
     exit 1
    fi

    packages=("nginx" "curl" "wget")

    for pkg in "${packages[@]}"; do
     if dpkg -s "$pkg" &>/dev/null; then
        echo "$pkg already installed"
     else
        echo "Installing $pkg..."
        apt install -y "$pkg"
    fi
    done
Sample Output:

    nginx already installed
    curl already installed
    Installing wget...

Task 5 – Error Handling
safe_script.sh
    #!/bin/bash
    set -e

    mkdir /tmp/devops-test || echo "Directory already exists"
    cd /tmp/devops-test || echo "Failed to enter directory"
    touch demo.txt || echo "File creation failed"

    echo "All steps completed"
Output:

    All steps completed

Key Learnings
    Learned how to use for and while loops in Bash.

    Understood command line arguments ($1, $#, $@).

    Automated package installation with root checks and error handling.