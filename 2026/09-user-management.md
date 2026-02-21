# Day 09 Challenge – User & Group Management

## Users & Groups Created
- Users: tokyo, berlin, professor, nairobi
- Groups: developers, admins, project-team

## Group Assignments
- tokyo → developers, project-team
- berlin → developers, admins
- professor → admins
- nairobi → project-team

## Directories Created
- /opt/dev-project → group: developers, permissions: 775
- /opt/team-workspace → group: project-team, permissions: 775

## Commands Used
- useradd -m
- passwd
- groupadd
- usermod -aG
- chgrp
- chmod
- groups
- ls -ld
- sudo -u

## What I Learned
1. Linux access control is managed using users, groups, and permissions.
2. Group ownership allows secure collaboration without exposing files to everyone.
3. Testing permissions using `sudo -u` simulates real user behavior.
