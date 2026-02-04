# Day 12 – Fundamentals Revision (Days 01–11)

# Goal
Consolidate Linux + DevOps fundamentals built over Days 01–11 so nothing slips away.

This day is focused on retention, not new learning.



# Mindset & Plan Review (Day 01)

- Original goal:
  - Become confident with Linux fundamentals
  - Build DevOps foundation step-by-step
- Current status:
  - Comfortable with basic commands
  - Better understanding of permissions, ownership, services
- Tweaks to plan:
  - Spend more time on troubleshooting
  - Practice real scenarios (permissions + services)
  - Start connecting Linux → Docker → AWS concepts



# Processes & Services (Day 04 / 05)

Commands rerun:


ps aux | head
systemctl status ssh
journalctl -u ssh --no-pager | tail


Observations

ps shows running processes with PID + CPU/memory

systemctl status confirms service state (active/running)

journalctl shows service logs and errors


File Skills Practice (Days 06–11)

Commands practiced:

echo "revision test" >> revision.txt
chmod 644 revision.txt
ls -l revision.txt
mkdir test-dir
cp revision.txt test-dir/

what i noticed:

appends safely without overwriting

chmod 644 gives owner write, others read

ls -l verifies permissions

cp copies files between directories


 Cheat Sheet Refresh (Day 03)

Top 5 incident-response commands:

ls -l – check permissions quickly

ps aux – see what’s running

df -h – disk usage

ss -tulnp – listening ports

journalctl -xe – system errors

 Which 3 commands save you the most time right now, and why?

ls -l – instantly shows permissions + ownership

systemctl status – fast service health check

journalctl – debugging starts here


 How do you check if a service is healthy?

First commands:

      systemctl status <service>
      journalctl -u <service>
      ps aux | grep <service>


What will you focus on improving in the next 3 days?

Stronger Linux troubleshooting

Docker deeper practice

AWS + EC2 workflows

Writing cleaner markdown notes



