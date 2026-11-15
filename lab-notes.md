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


## Part 2 – Week 1's network built by hand

```
# ip netns exec red ping -c 2 10.16.9.2
2 packets transmitted, 2 received, 0% packet loss

# bridge fdb show br br0   (learned entries only)
c2:b6:bd:a2:24:94 dev red-br master br0
22:ce:66:43:05:c8 dev blue-br master br0
```

- Built 2 namespaces (red, blue), 2 veth pairs, 1 bridge (br0)
- Ping works; bridge learned veth-red's MAC on port red-br and veth-blue's MAC on port blue-br
- That is all Mininet does: namespaces + veth pairs + bridge


## Part 3 – VXLAN overlay by hand (VNI 100, overlay 10.16.100.0/24)

```
mininet> h1 ping -c 2 10.16.100.2
2 packets transmitted, 2 received, 0% packet loss

mininet> h1 ping -c 6 10.16.100.2 > /dev/null &
mininet> h2 timeout 5 tcpdump -ni h2-eth0 udp port 4789
IP 10.16.0.1.33565 > 10.16.0.2.4789: VXLAN, flags [I] (0x08), vni 100
IP 10.16.100.1 > 10.16.100.2: ICMP echo request, id 93, seq 6, length 64
IP 10.16.0.2.33565 > 10.16.0.1.4789: VXLAN, flags [I] (0x08), vni 100
IP 10.16.100.2 > 10.16.100.1: ICMP echo reply, id 93, seq 6, length 64
IP 10.16.0.2.51687 > 10.16.0.1.4789: VXLAN, flags [I] (0x08), vni 100
ARP, Request who-has 10.16.100.1 tell 10.16.100.2, length 28
...
```

- Outer header: underlay 10.16.0.1 <-> 10.16.0.2, UDP 4789, VNI 100
- Inner: overlay ping 10.16.100.1 <-> 10.16.100.2; ARP also tunnelled (VXLAN carries Ethernet frames)
- s1 forwarded plain UDP; it didn't know about the tunnel (unlike the Week 5 P4 tunnel, where the switch had to parse the tunnel header)
- Note: a background tcpdump (&) captured 0 packets on my Mac; running the ping in the background and tcpdump in the foreground worked


## Part 4 – Cost of tunnelling

```
h1-eth0: mtu 1500    vxlan100: mtu 1450

h1 ping -M do -c 1 -s 1422 10.16.100.2  -> 1422(1450) bytes: 0% packet loss
h1 ping -M do -c 1 -s 1423 10.16.100.2  -> ping: local error: message too long, mtu=1450

h1 iperf3 -c 10.16.0.2 -t 5    (underlay)  78.6 Gbits/sec receiver
h1 iperf3 -c 10.16.100.2 -t 5  (overlay)   55.9 Gbits/sec receiver
```

- Overlay = ~71% of underlay throughput (~29% slower): encap/decap work + 50 fewer payload bytes per packet
- 50 bytes = inner Ethernet 14 + VXLAN 8 + outer UDP 8 + outer IPv4 20
- Ping math: 1422 data + 28 (IPv4 20 + ICMP 8) = 1450 = tunnel MTU
- Absolute numbers inflated (nothing leaves the laptop); the ratio is what matters



## Part 5 – VXLAN over an IPv6 underlay (VNI 106, overlay 10.16.106.0/24)

```
h1 ping -6 -c 2 fd00:5016::2        -> IPv6 underlay reachable
h1 ping -c 2 10.16.106.2            -> 2 received, 0% packet loss
h1 ip link show vxlan106            -> mtu 1430
h1 ping -M do -c 1 -s 1402 ...      -> 1402(1430) bytes: 0% packet loss
h1 ping -M do -c 1 -s 1403 ...      -> message too long, mtu=1430
```

- IPv6 outer header = 40 bytes vs IPv4 = 20, so overhead = 14 + 8 + 8 + 40 = 70 bytes
- Tunnel MTU = 1500 - 70 = 1430 (vs 1450 over IPv4)
- The overlay is still IPv4: the underlay and overlay protocols are independent