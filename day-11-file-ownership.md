# Day 11 Challenge – File & Directory Ownership in Linux

## Objective

Master Linux file and directory ownership concepts:

* Understand **user (owner)** and **group** ownership
* Use `chown` and `chgrp`
* Apply ownership changes recursively



## Task 1: Understanding Ownership

### Command


ls -l 


### Format Explained


-rw-r--r-- 1 owner group size date filename


* **owner** → The user who owns the file
* **group** → The group associated with the file

### Difference Between Owner and Group

* **Owner**: Primary controller of the file (can change permissions & ownership)
* **Group**: Allows shared access to multiple users without making files public



## Task 2: Basic `chown` Operations

### Commands


touch devops-file.txt
ls -l devops-file.txt

sudo useradd tokyo
sudo useradd berlin

sudo chown tokyo devops-file.txt
ls -l devops-file.txt

sudo chown berlin devops-file.txt
ls -l devops-file.txt


### Ownership Change

* `devops-file.txt`: `user:user` → `tokyo:user` → `berlin:user`



## Task 3: Basic `chgrp` Operations

### Commands


touch team-notes.txt
ls -l team-notes.txt

sudo groupadd heist-team
sudo chgrp heist-team team-notes.txt
ls -l team-notes.txt


### Ownership Change

`team-notes.txt`: `user:user` → `user:heist-team`



## Task 4: Combined Owner & Group Change

### Commands


touch project-config.yaml
sudo useradd professor

sudo chown professor:heist-team project-config.yaml
ls -l project-config.yaml

mkdir app-logs
sudo chown berlin:heist-team app-logs
ls -ld app-logs




## Task 5: Recursive Ownership

### Directory Structure


mkdir -p heist-project/vault
mkdir -p heist-project/plans
touch heist-project/vault/gold.txt
touch heist-project/plans/strategy.conf

sudo groupadd planners


### Recursive Change

sudo chown -R professor:planners heist-project/
ls -lR heist-project/


### Result

* Entire directory tree now owned by `professor:planners`



## Task 6: Practice Challenge

### Setup


sudo useradd tokyo
sudo useradd berlin
sudo useradd nairobi

sudo groupadd vault-team
sudo groupadd tech-team

mkdir bank-heist
touch bank-heist/access-codes.txt
touch bank-heist/blueprints.pdf
touch bank-heist/escape-plan.txt


### Ownership Assignment


sudo chown tokyo:vault-team bank-heist/access-codes.txt
sudo chown berlin:tech-team bank-heist/blueprints.pdf
sudo chown nairobi:vault-team bank-heist/escape-plan.txt

ls -l bank-heist/




## Files & Directories Created

* devops-file.txt
* team-notes.txt
* project-config.yaml
* app-logs/
* heist-project/
* bank-heist/



## Commands Reference


ls -l filename
sudo chown newowner filename
sudo chgrp newgroup filename
sudo chown owner:group filename
sudo chown -R owner:group directory/
sudo chown :groupname filename




## What I Learned

1. Ownership controls **who** can access and manage files in Linux
2. `chown` can change both user and group in one command
3. Recursive ownership is critical for managing project directories



 Screenshots captured after each ownership change using `ls -l`
