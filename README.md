**Lab 6 Wrap up**

**1. Linux building blocks behind a Mininet host and cable**
- Host = a `bash` process with its own network namespace (h1 = pid 26, matching `lsns`)
- Cable = a veth pair: one end inside the host (h1-eth0), the other on the switch (s1-eth1)
- Switch (lxbr) = a Linux bridge

**2. MTU and throughput, underlay vs overlay**

| | MTU | iperf3 (receiver) |
|---|---|---|
| Underlay h1-eth0 | 1500 | 78.6 Gbits/sec |
| Overlay vxlan100 (IPv4 underlay) | 1450 | 55.9 Gbits/sec (~71%) |
| Overlay vxlan106 (IPv6 underlay) | 1430 | – |

- 50 bytes = inner Ethernet 14 + VXLAN 8 + outer UDP 8 + outer IPv4 20
- Over IPv6 the outer IP header is 40 bytes instead of 20, so 14 + 8 + 8 + 40 = 70 bytes and the MTU is 1430
- Confirmed with DF pings: 1422 OK / 1423 too long (IPv4); 1402 OK / 1403 too long (IPv6)

**3. Why small pings worked during the MTU mismatch but TCP almost stopped**
- Small pings (84 bytes) fit under h2's MTU of 1400
- TCP sends full-size segments (1450 inner + 50 = 1500 on the wire), and h2 silently dropped all of them
- No error came back, so TCP kept retransmitting and its window collapsed: 628 Kbits/sec vs 55.9 Gbits/sec
- The largest working DF size was 1326 (1326 + 28 + 50 = 1404, i.e. 1400 + 4 bytes of VLAN allowance)

**4. What a Docker container has that a Mininet host doesn't**
- Its own filesystem image (Alpine Linux, not the host's OS)
- Its own process namespace (`ps` shows only its own processes; `sleep` is PID 1)
- A Mininet host only has its own network namespace and shares everything else

**5. VXLAN vs my Week 5 P4 tunnel**

| | Week 5 P4 tunnel | VXLAN |
|---|---|---|
| Bytes added | 4 | 50 (70 over IPv6) |
| Devices that must understand it | Every switch on the path must parse the tunnel header | Only the two endpoints (VTEPs); s1 saw plain UDP |
| What stops loops | No TTL in the tunnel header; relies on correct forwarding tables | The outer IP header has a TTL that routers decrement |

## AI use
- Claude (Anthropic): step-by-step guidance debugging the lab commands.