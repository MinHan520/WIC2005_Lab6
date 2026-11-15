# Week 6 Lab Notes (NN=16, NNNN=5016)

## Part 1 – Mininet hosts are namespaces

```
mininet> dump
<Host h1: h1-eth0:10.16.0.1 pid=26>
<Host h2: h2-eth0:10.16.0.2 pid=28>
<LinuxBridge s1: lo:127.0.0.1,s1-eth1:None,s1-eth2:None pid=34>

mininet> sh lsns -t net -o NS,PID,COMMAND
        NS PID COMMAND
4026532558   1 bash
4026532690  26 bash --norc --noediting -is mininet:h1
4026532819  28 bash --norc --noediting -is mininet:h2

mininet> h1 ip -br link
lo               UNKNOWN  00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
h1-eth0@if12     UP       00:00:00:00:00:01 <BROADCAST,MULTICAST,UP,LOWER_UP>
(plus default tunnel devices tunl0, gre0, sit0 etc. – all DOWN, unused)
```

- h1 = bash process pid 26 with its own network namespace; h2 = pid 28
- h1 only sees h1-eth0; its peer (@if12) is the switch port s1-eth1
- Mininet host = process + network namespace; Mininet cable = veth pair