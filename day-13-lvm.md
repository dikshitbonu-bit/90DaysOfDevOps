# Day 13 – Linux LVM Hands-on

## Objective
Learn Logical Volume Manager (LVM) to flexibly manage storage by creating, mounting, and extending logical volumes.


## Storage Check

Commands:
lsblk  
pvs  
vgs  
lvs  
df -h  


## Create Physical Volume

pvcreate /dev/loop0  
pvs  



## Create Volume Group

vgcreate devops-vg /dev/loop0  
vgs  



## Create Logical Volume

lvcreate -L 500M -n app-data devops-vg  
lvs  



## Format and Mount

mkfs.ext4 /dev/devops-vg/app-data  
mkdir -p /mnt/app-data  
mount /dev/devops-vg/app-data /mnt/app-data  
df -h /mnt/app-data  



## Extend Logical Volume

lvextend -L +200M /dev/devops-vg/app-data  
resize2fs /dev/devops-vg/app-data  
df -h /mnt/app-data  



## Learning Outcome

- Understood LVM architecture (PV → VG → LV)
- Created and mounted logical volumes
- Extended storage without downtime
- Learned real-world dynamic disk management
