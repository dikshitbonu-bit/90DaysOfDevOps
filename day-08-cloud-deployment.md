# Day 08 – Cloud Deployment & Web Server Setup

## Objective
Deploy a real web server on the cloud, manage access, extract logs, and verify public availability.



## Commands Used

### SSH Connection
ssh -i your-key.pem ubuntu@<public-ip>

### System Update
sudo apt update && sudo apt upgrade -y

### Install Nginx
sudo apt install nginx -y

### Check Nginx Status
systemctl status nginx

### View Logs
sudo cat /var/log/nginx/access.log

### Save Logs
sudo cat /var/log/nginx/access.log > ~/nginx-logs.txt

### Download Logs
scp -i your-key.pem ubuntu@<public-ip>:~/nginx-logs.txt .



## Challenges Faced

- Initially nginx page was not accessible due to missing port 80 rule in security group.
- Solved by allowing HTTP (port 80) from anywhere in cloud firewall.



## What I Learned

- How to launch and access a cloud VM
- How web servers work in real production environments
- Importance of security groups and firewall rules
- Where nginx logs are stored and how to extract them
- End-to-end deployment workflow used in DevOps roles
