# Troubleshooting Log

## Issue 1 — Interfaces Administratively Down

**Diagnostic Output**
```
GigabitEthernet0/0   unassigned   YES   unset   administratively down   down
GigabitEthernet0/1   unassigned   YES   unset   administratively down   down
```

**Root Cause**
Cisco IOS interfaces are turned off by default and must be turned on with the no shutdown command.

**Fix**
```
configure terminal
interface GigabitEthernet0/0
no shutdown
exit
interface GigabitEthernet0/1
no shutdown
exit
end
write memory
```

**Verification**
```
GigabitEthernet0/0   up   up
GigabitEthernet0/1   up   up
```

**Lesson**
Always bring interfaces up immediately after applying an IP, or first.

---

## Issue 2 — Server0 IP Conflict With Router Gateway

**Symptom**
Admin_PC could not ping Server0 (10.10.40.10). 100% packet loss.

**Diagnostic Commands**
```
ping 10.10.40.10
show ip arp
```

**Root Cause**
Server0 had been configured with IPv4 10.10.40.1, which conflicts with Router0's Gig0/1.40, because they are on the same VLAN 40. The router will not ARP for its own IP on the LAN, resulting in packets being dropped.

**Fix**
Reconfigured Server0 via Desktop → IP Configuration:
- Before: 10.10.40.1
- After: 10.10.40.10
- Gateway remained 10.10.40.1 (correct)

**Verification**
Ping 10.10.40.10 from Admin_PC:
```
Reply from 10.10.40.10: bytes=32 time=26ms TTL=127
Reply from 10.10.40.10: bytes=32 time<1ms TTL=127
Reply from 10.10.40.10: bytes=32 time=19ms TTL=127
```

**Lesson Learned**
The .1 address is always reserved for the gateway. End devices begin at .10 or above. Follow the addressing plan.

---

## Issue 3 — Packet Tracer in Simulation Mode (Pings Hang)

**Symptom**
Ping commands simply hung, and no further output was generated.
```
Pinging 10.10.10.1 with 32 bytes of data:
```
No replies, no timeouts, nothing more.

**Diagnostic**
Bottom right of PT indicated the Simulation mode was active (clock icon highlighted). A purple captured packet was visible on the Switch1 uplink.

**Root Cause**
Packet Tracer (PT) runs in Simulation mode, where each packet is captured and the user must press Play ▶ to continue. Pings in Simulation mode hang until the user takes this action.

**Fix**
- Clicked Realtime tab (bottom right)
- Deleted the purple packet that was causing the hang
- Ran the ping again

**Verification**
Ping replied instantly with TTL=255.

**Lesson Learned**
Always use the Realtime tab for connectivity tests. Use Simulation mode only for packet analysis.

---

## Issue 4 — First Ping Always Times Out (ARP Resolution)

**Symptom**
Even on a healthy network, the first ping to a new destination loses 1 of 4 packets.
```
Reply from 10.10.10.1: bytes=32 time=616ms TTL=255
Reply from 10.10.10.1: bytes=32 time<1ms TTL=255
Reply from 10.10.10.1: bytes=32 time<1ms TTL=255
Request timed out.
```

**Root Cause**
Normal behavior of ARP: First packet triggers ARP resolution of the destination MAC, and as a result, the first ICMP packet is dropped.

**Fix**
No fix – expected behavior. Work around: Run the ping twice, and capture the second run, as an exception in the evidence screenshots.

**Lesson Learned**
"Expected first packet loss" vs "actual first packet loss". A packet loss of 1/4 is normal. A 1/1 packet loss is not.

---

## Issue 5 — Servers on DHCP Mode Instead of Static

**Symptom**
Server0 and Server_Test both had blank or 0.0.0.0 IP. Neither could be contacted nor could they initiate connections.

**Diagnostic**
Server0 → Desktop → IP Configuration displayed:
```
Mode: DHCP
IPv4 Address: [blank or 0.0.0.0]
```

**Root Cause**
End devices in Packet Tracer default to using DHCP mode. Because VLAN 40 and the WAN segment have no DHCP pool (by design), the servers did not get an IP address.

**Fix**
Server0:
```
Mode: Static
IPv4: 10.10.40.10 / 255.255.255.0
Gateway: 10.10.40.1
DNS: 8.8.8.8
Services → HTTP → ON
```
Server_Test:
```
Mode: Static
IPv4: 203.0.113.10 / 255.255.255.0
Gateway: 203.0.113.1
DNS: 8.8.8.8
Services → HTTP → ON
```

**Verification**
- Admin_PC → Server0
- Admin_PC → Server_Test
- Kiosk_PC → Server_Test
- Kiosk_PC browser → http://203.0.113.10

**Lesson Learned**
Static devices need to be manually configured. DHCP mode is for end-user devices.

---

## Issue 6 — Cloud0 Port Mappings Empty

**Symptom**
Server_Test could not be contacted from LAN, and Server_Test could not contact LAN, even with correct IPs.

**Diagnostic**
Cloud0 → Config → DSL tab displayed an empty port mapping table.

**Root Cause**
Cloud-PT device does not bridge packets automatically. Mapping of which internal port maps to which physical port must be done manually. Without mapping, Cloud drops packets.

**Fix**
- Open Cloud0 → Config → DSL tab
- From: Modem4
- To: Ethernet6
- Add

**Verification**
Server_Test became contactable: ping 203.0.113.10 successful from Admin_PC.

**Lesson Learned**
PT Cloud is a gotcha. Always check port mapping first.

---

## Issue 7 — STP Convergence Delay on Switch2 Trunk

**Symptom**
show interfaces trunk on Switch2 displayed:
```
Vlans in spanning tree forwarding state and not pruned
Fa0/1   none
```

**Root Cause**
Spanning Tree Protocol (STP) had not yet converged after the trunk was configured. 802.1D STP takes 30–50 seconds after a topology change.

**Fix**
Waited 30 seconds and re-ran the command.

**Verification**
```
Vlans in spanning tree forwarding state and not pruned
Fa0/1   10,20,30,40,66,99
```

**Lesson Learned**
Never assume a trunk is failed, just because STP shows nothing. Wait for convergence, then re-test.

---

## Issue 8 — Typo: "how" Instead of "show"

**Symptom**
```
% Invalid input detected at '^' marker.
```

**Diagnostic**
User had typed "how cdp neighbors" instead of "show cdp neighbors".

**Root Cause**
Typing error dropped the s.

**Fix**
Retyped correctly.

**Lesson Learned**
Use Tab completion to auto-complete commands. Cisco IOS also allows abbreviations – e.g., sh cdp n.

---

## Issue 9 — switchport trunk encapsulation dot1q Rejected

**Symptom**
Applying the command on Cisco 2960 returned:
```
% Invalid input detected at '^' marker.
```

**Root Cause**
2960 is a Layer 2-only device. It only uses 802.1Q encapsulation on trunks, and does not allow the encapsulation command (which only applies on switches that support both ISL and 802.1Q, such as the 3560).

**Fix**
Skipped line – trunks on 2960 default to 802.1Q.

**Verification**
show interfaces trunk confirmed trunking with 802.1q encapsulation.

**Lesson Learned**
Know the platform: 2960 automatically uses 802.1Q; no need to specify.
