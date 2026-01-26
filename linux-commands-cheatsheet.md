# 🐧 Linux Commands Cheat Sheet (Troubleshooting Focus)

##  Process Management

| Command           | Usage                                   |
| ----------------- | --------------------------------------- |
| `ps aux`          | Show all running processes with details |
| `ps -ef`          | Full-format listing of all processes    |
| `top`             | Real-time process and CPU/memory usage  |
| `htop`            | Improved interactive process viewer     |
| `pidof <process>` | Get PID of a running process            |
| `kill <PID>`      | Gracefully stop a process               |
| `kill -9 <PID>`   | Force kill a stuck process              |
| `pkill <name>`    | Kill process by name                    |
| `bg`              | Resume stopped job in background        |
| `fg`              | Bring background job to foreground      |
| `jobs`            | List background jobs                    |
| `uptime`          | Show system run time and load average   |



##  File System & Disk

| Command                 | Usage                                |
| ----------------------- | ------------------------------------ |
| `ls -lh`                | List files with human-readable sizes |
| `pwd`                   | Show current directory               |
| `cd <dir>`              | Change directory                     |
| `du -sh <dir>`          | Size of directory                    |
| `df -h`                 | Disk space usage                     |
| `stat <file>`           | Detailed file info                   |
| `touch <file>`          | Create empty file                    |
| `mkdir -p a/b/c`        | Create nested directories            |
| `rm -rf <dir>`          | Delete directory forcefully          |
| `cp -r src dst`         | Copy directories                     |
| `mv old new`            | Rename or move file                  |
| `find / -name file`     | Search file by name                  |
| `chmod 755 file`        | Change file permissions              |
| `chown user:group file` | Change ownership                     |



##  Networking & Troubleshooting

| Command                 | Usage                            |
| ----------------------- | -------------------------------- |
| `ip addr`               | Show IP addresses and interfaces |
| `ip route`              | Display routing table            |
| `ping google.com`       | Check network connectivity       |
| `ss -tuln`              | Show listening ports             |
| `netstat -tulnp`        | Ports and associated processes   |
| `curl <url>`            | Test HTTP/HTTPS response         |
| `curl -I <url>`         | Fetch only HTTP headers          |
| `dig google.com`        | DNS lookup                       |
| `nslookup domain`       | Query DNS records                |
| `traceroute google.com` | Trace network path               |



##  System & Logs 

| Command                      | Usage                     |
| ---------------------------- | ------------------------- |
| `uname -a`                   | Kernel and system info    |
| `free -h`                    | Memory usage              |
| `vmstat`                     | CPU, memory, I/O stats    |
| `whoami`                     | Current user              |
| `id`                         | User and group IDs        |
| `journalctl -xe`             | View systemd logs         |
| `systemctl status <service>` | Check service status      |
| `history`                    | Show command history      |
| `!123`                       | Re-run command number 123 |
