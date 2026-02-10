task 1 :
  
  #!/bin/bash
  echo "hello dk"

task 2 :

    #!/bin/bash
    #
    #
    NAME="DK"
    ROLE="DEVOPS GUY"


    echo "hello iam $NAME and iam $ROLE"


task 3 :
   
    #!/bin/bash
    #
    #
    read -p "Enter your name : " NAME

    read -p "Enter your favourite tool : " TOOL

    echo " your name is $NAME  and your favourite tool is $TOOL"

task 4 :
    
    #!/bin/bash
    #
    #

    read -p "Enter the number you want to check: " NUMBER

    if [ "$NUMBER" -gt 0 ]; then
        echo "number is positive"

    elif [ "$NUMBER" -lt 0 ]; then
      echo "number is negative"

    else
        echo "number is zero"

    fi

task 5 :
    

    #!/bin/bash
    #
    #
    #
    read -p "Enter the service name you want to check: " SERVICE

    read -p "Do you want to check the status? (y/n): " INPUT

    if [ "$INPUT" = "y" ]; then
        sudo systemctl status $SERVICE | grep active
    elif [ "$INPUT" != "y" ]; then
        echo "skipped"
    fi
