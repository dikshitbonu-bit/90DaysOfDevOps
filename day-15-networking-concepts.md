Day 15 – Core Networking Concepts (Beginner Friendly)
 Task 1: DNS – How Names Become IPs

     When I type google.com in my browser, my computer first checks its local DNS cache (memory).
     If it doesn’t find the IP, it asks my ISP’s DNS server.
    If ISP DNS doesn’t know, it asks Root DNS → then TLD DNS (.com) → then Google’s authoritative DNS.
    Google’s DNS finally returns the IP address, and my browser connects to that IP.

    DNS is a distributed system — not one server — where each layer owns part of the responsibility.

 DNS hierarchy:

     Local DNS cache → saved answers on my laptop

    ISP DNS → my internet provider’s DNS

    Root DNS → knows where TLDs live

    TLD DNS → knows who owns domains (.com, .in, etc)

    Authoritative DNS → owned by the company (Google, AWS, etc) and stores real records

 DNS Record Types:

    A – domain → IPv4 address

    AAAA – domain → IPv6 address

    CNAME – alias to another domain

    MX – mail servers for email delivery

    NS – tells which DNS servers control the domain

Ran:

    dig google.com


    Observed:

    A  record gives IP

    TTL shows how long the result is cached

Task 2: IP Addressing

    An IPv4 address is a 32-bit number written in dotted format like:

    192.168.1.10


    Public IP: reachable from internet (example: EC2 public IP)
    Private IP: internal network only (example: 10.0.0.5)

Private IP ranges:

    10.0.0.0 – 10.255.255.255

    172.16.0.0 – 172.31.255.255

    192.168.0.0 – 192.168.255.255

Checked my system IP:

    ip addr show


    Identified my private IP.

Task 3: CIDR & Subnetting

    CIDR tells how many bits are used for network.

    Example:

    192.168.1.0/24


    Means:

    24 bits = network

    remaining bits = hosts

    Formula:

    Usable hosts = 2^(32 − CIDR) − 2


    Subtract 2 because:

    first IP = network address

    last IP = broadcast address

Filled table:

    CIDR	Subnet Mask	    Total IPs  	Usable
    /24	     255.255.255.0	  256  	      254
    /16	     255.255.0.0	  65536	     65534
    /28	     255.255.255.240    16	           14

Subnetting is used to:

   isolate networks

    improve security

    manage IP usage

    organize infrastructure

Task 4: Ports – Doors to Services

    A port identifies which service on a machine you want to talk to.

    Common ports:

        22 – SSH

         80 – HTTP

        443 – HTTPS

        53 – DNS

        3306 – MySQL

        6379 – Redis

        27017 – MongoDB

Checked listening services:

    ss -tulpn


    Matched ports to services.

Task 5: Putting It Together
    curl http://myapp.com:8080

        Involves:

            DNS resolution

            IP routing

            TCP connection

            HTTP protocol

            port 8080

App can’t reach DB at 10.0.1.50:3306

    Troubleshooting order:

            Check host reachable:

                ping 10.0.1.50


            Check port open:

                nc -zv 10.0.1.50 3306


            Check service running:

                systemctl status mysql


                    or

                    docker ps


Real flow:

    Host → Port → Process