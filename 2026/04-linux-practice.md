
##  Process Checks

### 1. List processes in current terminal

ps

Explanation:  
Shows processes started in the current terminal session.  
Helps understand that every command you run becomes a process.



### 2. List all running processes

ps aux

Explanation:  
Displays all running processes on the system along with CPU and memory usage.  
Used to identify high resource usage or confirm if a process is running.



### 3. Live process monitoring

top

Explanation:  
Shows real-time CPU, memory usage, and running processes.  
Useful when the system feels slow.  
Press `q` to exit.



### 4. Find process by name

pgrep ssh

Explanation:  
Returns the process ID of the SSH process if it is running.  
No output means the process is not running.



##  Service Checks (systemd)

### 1. Check service status

systemctl status ssh

Explanation:  
Shows whether the SSH service is running, stopped, or failed.  
Displays service logs and main process ID.



### 2. Start a service

sudo systemctl start ssh

Explanation:  
Starts the SSH service if it is stopped.


### 3. Restart a service

sudo systemctl restart ssh

Explanation:  
Stops and starts the SSH service again.  
Commonly used after configuration changes.



### 4. List running services

systemctl list-units --type=service

Explanation:  
Lists all active services managed by systemd.



## 🔹 Log Checks

### 1. View all system logs

journalctl

Explanation:  
Displays all system logs collected by systemd.  
Press `q` to exit.



### 2. View logs for a specific service

journalctl -u ssh

Explanation:  
Shows logs related only to the SSH service.  
Used to diagnose service issues.



### 3. View recent error logs

journalctl -xe

Explanation:  
Shows recent logs with error explanations.  
Very helpful for troubleshooting failures.



### 4. View latest system logs

tail -n 50 /var/log/syslog

Explanation:  
Displays the last 50 lines of system logs.  
Used to quickly inspect recent system activity.



## 🔹 Mini Troubleshooting Flow (SSH Service)

Problem: Unable to connect via SSH

### Step 1: Check SSH service status

systemctl status ssh

Confirms whether the SSH service is running or failed.



### Step 2: Restart SSH service

sudo systemctl restart ssh

Fixes temporary issues or reloads the service.



### Step 3: Check SSH logs

journalctl -u ssh --no-pager | tail

Identifies authentication or configuration-related errors.



### Step 4: Verify SSH port

ss -tulnp | grep :22

Confirms that SSH is listening on port 22.



### Conclusion
- Service state checked  
- Logs inspected  
- Port verified  
- Issue identified or resolved  




