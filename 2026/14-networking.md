Day 14 – Core Networking + Troubleshooting Basics

Quick Concepts
1) OSI vs TCP/IP (in my own words)

2) OSI has 7 layers (Physical, Data Link, Network, Transport, Session,        Presentation, Application). It’s mainly a learning and debugging model.

3) TCP/IP has 4 layers (Link, Internet, Transport, Application). This is what actually runs on real systems and the internet.

4) OSI helps understand where a problem lives.
TCP/IP explains how data actually flows.

    Where things sit in the stack

    IP → Internet layer (TCP/IP) / Network layer (OSI)

1) TCP/UDP → Transport layer

2) DNS → Application layer

3) HTTP/HTTPS → Application layer

 Real example:
 curl https://google.com
 = Application (HTTP) over Transport (TCP) over Internet (IP)

   Hands-on Checklist

1) Target used: google.com
 


2) Used hostname -I

     Observation: Shows my private IP address assigned to the machine.


3) Used ping google.com
 
     Observation: Host reachable with low latency and no packet loss.



4)  Used traceroute google.com

     Observation: Traffic passes through multiple ISP hops; some hops timed out (normal due to ICMP blocking).



5)  Used ss -tulpn

     Observation: SSH service listening on port 22.


6)  Used dig google.com

     Observation: Domain resolved to multiple IP addresses (load balancing).



7) Used curl -I https://google.com

     Observation: Received HTTP response (301/200), confirming web server availability.



8)  Used netstat -an | head

     Observation: Mix of LISTEN and ESTABLISHED connections.

Mini Task – Port Probe & Interpret

1)  Listening port identified: SSH on port 22

2)   Tested locally using nc on port 22.

3)   Result: Port reachable.

If not reachable, next checks would be:

     Service status (systemctl status ssh)

     Firewall rules (ufw or iptables)

Reflection

1) Fastest signal when something breaks:

2) ping for basic connectivity

3) curl -I for application availability

If DNS fails:

   Inspect Application layer first.

If HTTP 500 appears:

   Start at Application layer and check logs.

Two follow-up checks in a real incident:

1)  Verify service status and logs
   
         (systemctl and journalctl)


2)  Check firewall and security group rules