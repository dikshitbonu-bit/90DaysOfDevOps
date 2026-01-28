
##  Target Service / Process
Service: SSH  
Process: sshd  
Purpose: Remote login access to the system



##  Environment Basics

### 1. Kernel and system info

uname -a

Observation:  
Shows kernel version, architecture, and OS type. Confirms what kernel I am troubleshooting.



### 2. OS release details

lsb_release -a

Observation:  
Shows distribution name and version. Helps identify OS-specific behavior.



##  Filesystem Sanity Check

### 3. Create a test directory

mkdir /tmp/runbook-demo

Observation:  
Directory created successfully, filesystem is writable.



### 4. Copy a file and list it

cp /etc/hosts /tmp/runbook-demo/hosts-copy && ls -l /tmp/runbook-demo

Observation:  
File copied correctly. Confirms disk write and permissions are normal.



##  Snapshot: CPU & Memory

### 5. Check overall CPU and process usage

htop

Observation:  
CPU usage is normal. No abnormal spikes from sshd.  
(Press `q` to exit)



### 6. Check memory usage

free -h

Observation:  
Sufficient free memory available. No memory pressure observed.



##  Snapshot: Disk & IO

### 7. Check disk usage

df -h

Observation:  
Root filesystem has sufficient free space. No disk-full condition.



### 8. Check log directory size

du -sh /var/log

Observation:  
Log size is reasonable - 560mb. No runaway log growth detected.



##  Snapshot: Network

### 9. Verify SSH port listening

ss -tulnp | grep :22

Observation:  
SSHD is listening on port 22. Network socket is active.



### 10. Test local SSH response

curl -I localhost

Observation:  
System responds locally. Network stack is functional.



##  Logs Reviewed

### 11. SSH service logs

journalctl -u ssh -n 50

Observation:  
No recent errors. Normal authentication and connection messages.



### 12. System log tail

tail -n 50 /var/log/syslog

Observation:  
No critical errors related to SSH or system resources.


##  If This Worsens (Next Steps)

1. Restart SSH safely  
   
   sudo systemctl restart ssh
   

2. Increase SSH log verbosity  
   - Edit `/etc/ssh/sshd_config`
   - Set `LogLevel DEBUG`
   - Reload service

3. Deep diagnostics  
   - Capture process details:

     ps -o pid,pcpu,pmem,comm -p $(pgrep sshd)
    
   - Collect strace if CPU spikes or hangs


